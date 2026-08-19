# MTP Target — plan de réécriture en Rust

> Proposition d'architecture et feuille de route. Suite du document
> [`01-analyse-existant.md`](01-analyse-existant.md).
>
> Cibles : **Windows, Linux, macOS, Android, iOS**.

---

## 1. Objectifs et contraintes

### Objectifs

1. Un jeu **jouable en ligne** sur les cinq plateformes, avec les mêmes serveurs.
2. Un **moteur de simulation déterministe et testable**, indépendant du rendu.
3. Un **rendu moderne** (wgpu) : le jeu doit être beau, pas seulement fonctionnel.
4. Une **sensation de jeu fidèle** à l'original — c'est le critère d'acceptation
   principal, pas la fidélité du code.
5. Le **contenu existant réutilisé** : 24 niveaux, 21 scripts de règles.

### Contraintes structurantes

| Contrainte | Conséquence |
|---|---|
| **GPL v2+** hérité de l'original | Toute dépendance doit être GPL-compatible. Exclut FMOD. Attention : la GPL et l'App Store d'Apple sont juridiquement incompatibles (la GPL interdit les restrictions de redistribution imposées par la DRM Apple). **Il faudra soit un accord des ayants droit pour relicencier, soit un contact avec les auteurs (Vianney Lecroart / Alban Lecocq), soit une réécriture *clean-room* qui ne dérive pas du code GPL.** Voir §8. |
| **iOS interdit le téléchargement de code exécutable** (App Store 2.5.2) | Le message `ExecLua` disparaît. Le scripting Lua reste **serveur uniquement**. Le contenu téléchargé doit être de la **donnée pure**, jamais du script. |
| **Réseaux mobiles** (RTT 60-200 ms, perte, bascule Wi-Fi↔4G) | Transport QUIC (migration de connexion native) + **prédiction client obligatoire**. |
| **GPU mobiles** | Textures ASTC/ETC2 via KTX2, budget de draw calls serré, pas de post-process coûteux par défaut. |
| **Écrans tactiles** | Refonte des contrôles, pas un simple portage de touches. |

---

## 2. Choix technologiques

| Domaine | Choix | Justification |
|---|---|---|
| **Langage** | Rust (edition 2024) | Demande explicite ; adapté au serveur et au client. |
| **Rendu / client** | **Bevy** (wgpu + winit) | Backends Vulkan/Metal/DX12/GL, **Android et iOS supportés**, ECS mûr, écosystème riche. Alternative écartée : wgpu+winit à la main (6 à 12 mois de travail en plus pour un résultat inférieur). |
| **Physique** | **Rapier3D** (`enhanced-determinism`) | Pur Rust, pas de FFI, déterminisme cross-platform activable — indispensable pour rejouer une partie serveur/client. |
| **Transport** | **QUIC** via `quinn` | Flux fiables *et* datagrammes non fiables sur une seule connexion chiffrée ; **migration de connexion** quand le mobile change de réseau ; traverse les NAT et les proxys d'opérateur mieux qu'UDP brut. |
| **Sérialisation** | `bitcode` ou encodage binaire maison + quantification | Reprend l'idée du flottant 10 bits de l'original, avec un schéma **versionné**. |
| **Scripting** | **`mlua`** (Lua 5.4, serveur uniquement) | Permet de reprendre les 21 scripts de règles quasi tels quels. |
| **Audio** | **`kira`** (ou `rodio`) | Pur Rust, mixage, spatialisation ; remplace FMOD. Android/iOS supportés via `cpal`. |
| **UI** | `bevy_ui` pour le HUD in-game, `egui` pour les écrans de debug/outils | Remplace 6 000 lignes de toolkit XML maison. |
| **Backend comptes/stats** | `axum` + `sqlx` + PostgreSQL, mots de passe **Argon2id** | Remplace `login_service` + les 122 fichiers PHP. |
| **Assets** | glTF 2.0 + KTX2 (ASTC/ETC2/BC selon plateforme) | Standards, outillage existant. |

### Pourquoi la simulation ne doit **pas** dépendre de Bevy

Le serveur doit tourner sans GPU, sans fenêtre, dans un conteneur, et
idéalement plusieurs parties par processus. Les tests de non-régression du
*feel* doivent s'exécuter en millisecondes dans la CI.

D'où la règle d'architecture : **`mtpt-sim` est une bibliothèque Rust pure**
(entrées → état, pas de rendu, pas de réseau, pas de global). Bevy et le
serveur sont deux *hôtes* de cette bibliothèque. C'est l'inverse exact de
l'original, où la physique, le réseau et le gameplay étaient enchevêtrés dans
`CEntityManager` et des singletons.

---

## 3. Architecture cible

```
mtp-target/                         (workspace Cargo)
├── crates/
│   ├── mtpt-sim/          ★ simulation déterministe, pas de I/O
│   │                        Rapier, entités, machine à états de session,
│   │                        pas fixe, replay, tests de trajectoire
│   ├── mtpt-protocol/     ★ types de messages, quantification, versioning
│   ├── mtpt-content/        chargement des niveaux (.lua données ou .ron),
│   │                        validation, index des assets
│   ├── mtpt-scripting/      bindings mlua sur mtpt-sim (serveur uniquement)
│   ├── mtpt-net-server/     quinn, sessions, snapshots, anti-triche basique
│   ├── mtpt-net-client/     quinn, réconciliation, interpolation, prédiction
│   ├── mtpt-render/         plugins Bevy : entités, décor, eau, ciel, particules
│   ├── mtpt-input/          abstraction clavier/manette/**tactile**
│   └── mtpt-assets/         pipeline : .shape → glTF, .tga/.dds → KTX2
├── apps/
│   ├── mtpt-client/         binaire Bevy (desktop + Android + iOS)
│   ├── mtpt-server/         binaire headless (tokio)
│   ├── mtpt-backend/        axum : comptes, classements, stats, liste de serveurs
│   └── mtpt-editor/         éditeur de niveaux (Bevy + egui), desktop uniquement
├── tools/
│   ├── nel-shape/           parseur du format NeL (crate isolée, réutilisable)
│   └── feel-harness/        comparaison de trajectoires vs valeurs de référence
└── docs/
```

Le symbole ★ marque les crates **sans aucune dépendance à Bevy, tokio ou wgpu**.

---

## 4. Netcode — conception

### 4.1 Le problème à résoudre

L'original : serveur autoritatif + interpolation à 150-600 ms dans le passé +
**zéro prédiction**. Sur mobile en 4G (RTT 120 ms), cela donnerait ~400 ms entre
l'appui et le mouvement à l'écran. Injouable.

### 4.2 Conception proposée

```
Client                                      Serveur
──────                                      ───────
tick N : lit l'input                        pas fixe 60 Hz (Rapier)
         applique localement (prédiction)
         envoie {tick, input}  ──────────▶  applique l'input au tick N
         ...                                snapshot delta à 30 Hz
         ◀──────── {tick, état, ack}        (datagrammes QUIC, non fiable)
réconcilie : si l'état serveur au tick N
diffère → rejoue les inputs N..now
```

- **Entité locale** : prédiction + réconciliation (rollback des seuls inputs
  locaux). Le pingouin réagit instantanément.
- **Entités distantes** : interpolation dans le passé — on garde le principe du
  **LCT** de l'original, avec un buffer adaptatif borné (100-300 ms) au lieu de
  150-600 ms.
- **Événements** (ouverture/fermeture, crash, score, collision) : flux **fiable**
  QUIC, jamais dans le flux de positions.
- **Quantification** : on reprend l'idée validée de l'original — le serveur
  applique la valeur quantifiée à sa référence, donc pas d'accumulation d'erreur.
- **Versioning** : chaque message porte un numéro de schéma ; le handshake
  refuse proprement une version incompatible au lieu de mal parser.

### 4.3 Ce qu'on supprime

`ExecLua` (RCE + interdit sur iOS), le CRC/patch HTTP maison, la connexion
en clair au démon de chat, l'endpoint dyndns en dur.

---

## 5. Mobile — ce que ça implique concrètement

### 5.1 Contrôles tactiles

Le schéma original est en fait **très favorable au tactile** : deux axes + un
bouton.

| Action | Desktop | Tactile proposé |
|---|---|---|
| Tourner / diriger | ←/→ | pouce gauche : stick virtuel (axe X) |
| Accélérer / assiette | ↑/↓ | même stick (axe Y) |
| Ouvrir/fermer la boule | Ctrl droit | **gros bouton unique à droite**, la seule action discrète du jeu |
| Caméra libre | souris | glissement à droite hors bouton |
| Changer de joueur observé | F9/F10/F11 | flèches en overlay pendant l'attente |
| Chat | clavier | clavier système + messages rapides pré-écrits |

À valider en playtest : un **contrôle par inclinaison** (gyroscope) pour la
direction en vol, en option.

### 5.2 Rendu et budget

- Textures compressées **par plateforme** : ASTC (iOS + Android moderne), ETC2
  (fallback Android), BC7 (desktop) — un seul pipeline KTX2, trois variantes.
- Le jeu est géométriquement minuscule (36 maillages de 1 à 12 Ko). Le budget
  n'est pas un problème ; l'effort doit aller dans l'éclairage, l'eau et les
  particules.
- Cible : **60 fps** sur un mobile milieu de gamme de 3 ans, avec plafond
  configurable à 30 fps pour la batterie.
- Résolution dynamique + qualité d'eau dégradable, comme le faisait déjà la
  config originale (`DisplayWater = 0/1/2`).

### 5.3 Cycle de vie applicatif

Android et iOS suspendent l'application. Il faut gérer :
- perte du contexte GPU (Android) → Bevy gère, mais à tester réellement ;
- passage en arrière-plan pendant une partie → **spectateur automatique** puis
  reconnexion à la session, plutôt qu'une déconnexion sèche ;
- reprise réseau : QUIC migre la connexion, mais le serveur doit tolérer un gel
  de 30 s sans éjecter le joueur.

### 5.4 Distribution

- Android : `cargo-apk` ou `xbuild`, AAB signé, Play Console.
- iOS : projet Xcode généré, `cargo-lipo`/`cargo-xcode`, TestFlight.
- **Le contenu ne peut pas être téléchargé sous forme de script sur iOS.** Les
  niveaux sont donc livrés comme **données déclaratives** dans le bundle, et
  toute logique de règles reste sur le serveur. C'est cohérent avec l'original :
  les scripts `*_server.lua` sont déjà exclusivement serveur.

---

## 6. Assets — stratégie

### 6.1 Le format `.shape`

C'est le verrou. Trois options :

| Option | Coût | Risque |
|---|---|---|
| **A. Écrire un parseur Rust du format NeL** (`tools/nel-shape`) | 1 à 3 semaines | Moyen — le format de sérialisation NeL est documenté par son propre code source (GPL, disponible dans Ryzom Core), et les fichiers sont petits et peu nombreux (36 fichiers, ~1 Mo). |
| B. Reconstruire un build NeL de 2005 pour exporter | Élevé, non reproductible | Élevé |
| C. Re-modéliser les 36 maillages | Long, coûteux | Faible techniquement, mais perte de fidélité |

**Recommandation : A, en tout premier.** C'est le *spike* qui décide de la
faisabilité du reste, et il est isolé dans une crate autonome. Fallback : les 18
`.max` disponibles couvrent les maillages « snow_* » et le pingouin, le reste
serait re-modélisé.

Rappel : les `.shape` portent **aussi** la collision et les « auto-edges »
d'aimantation de l'éditeur. Le parseur doit exposer les trois.

### 6.2 Le reste

- `.tga`/`.dds`/`.psd` → PNG source + KTX2 compilé : outillage standard.
- `.wav`/`.mp3` → OGG/Opus.
- Polices : vérifier la licence de `n019003l.pfb` (URW Nimbus, GPL — OK) et de
  `bigfont.ttf` (à identifier, sinon remplacer).
- Niveaux : les 24 `.lua` sont convertis en **données validées** (`.ron` ou
  `.lua` chargé en lecture seule par `mlua` côté outil), avec conversion
  d'échelle explicite (hectomètres → mètres, cf. `GScale = 0.01`).

---

## 7. Feuille de route

Chaque phase a un **critère de sortie vérifiable**. Aucune phase ne démarre
avant que la précédente ne soit démontrée.

### Phase 0 — Conservation et mesure *(1-2 semaines)*

- Extraire dans un fichier de référence **toutes les constantes physiques**
  (variables serveur + surcharges par niveau).
- Documenter les 55 méthodes de l'API Lua comme **contrat cible**.
- Écrire les **trajectoires de référence** attendues (départ, séquence d'inputs,
  position/score final) — issues de la lecture du code, à affiner en playtest.
- **Sortie** : `docs/03-reference-gameplay.md` + `tools/feel-harness` avec des
  tests qui échouent (rien à comparer encore).

### Phase 1 — Spike assets *(2-3 semaines)*

- Crate `nel-shape` : parseur `.shape` → maillage + collision + auto-edges.
- Export glTF des 36 maillages, conversion des textures.
- **Sortie** : les 36 maillages s'affichent dans une visionneuse. Décision
  go/no-go sur la réutilisation des assets.

### Phase 2 — Simulation *(4-6 semaines)*

- `mtpt-sim` : Rapier, pas fixe **borné** (max N sous-pas par frame, contrairement
  à l'original), entités, mode boule/planeur, eau, cibles, scoring.
- Chargement d'un niveau converti, machine à états de session.
- **Sortie** : `feel-harness` passe sur au moins 3 niveaux ; une partie
  complète se déroule en headless, sans réseau ni rendu, de façon déterministe
  (même seed → même résultat, sur les 5 plateformes).

### Phase 3 — Réseau *(4-6 semaines)*

- `mtpt-protocol` versionné, `mtpt-net-server` (quinn + tokio),
  `mtpt-net-client` avec prédiction/réconciliation/interpolation.
- Client de debug minimal (formes filaires, egui).
- **Sortie** : 4 joueurs sur 3 machines, dont **un mobile en 4G**, jouent une
  session complète. Métriques : bande passante par joueur, RTT, taux de
  correction de prédiction.

### Phase 4 — Client de jeu *(8-12 semaines)*

- `mtpt-render` : caméra (reprendre le comportement de `client/src/camera.cpp`),
  eau, ciel, particules, traces, lens flares.
- HUD, tableau des scores, écrans de connexion, chat.
- `mtpt-input` : clavier, manette, **tactile**.
- **Sortie** : le jeu est jouable et présentable sur desktop **et** sur un
  téléphone Android et un iPhone.

### Phase 5 — Contenu et règles *(4-6 semaines)*

- `mtpt-scripting` : API `Entity`/`Module`/`Level` sur `mlua`.
- Portage des 21 scripts de règles, des 24 niveaux, des bots.
- **Sortie** : les 24 modes de jeu originaux sont jouables.

### Phase 6 — Services *(4-6 semaines)*

- `mtpt-backend` : comptes (Argon2id), classements, stats, liste de serveurs,
  télémétrie. Sign in with Apple / Google Play Games en option mobile.
- **Sortie** : un joueur crée un compte sur mobile, joue, retrouve son score.

### Phase 7 — Finition *(continu)*

Éditeur de niveaux, replays, matchmaking, accessibilité, localisation,
publication sur les stores.

---

## 8. Décisions à prendre — j'ai besoin de tes réponses

Ces points changent le plan en profondeur ; je ne peux pas trancher à ta place.

1. **Licence.** Le code original est GPL v2+. Un portage qui s'en inspire
   ligne à ligne en hérite, et **la GPL est incompatible avec l'App Store**.
   Trois voies : (a) contacter Vianney Lecroart / Alban Lecocq pour un
   relicenciement (MIT/Apache-2.0 ou double licence) ; (b) réécriture *clean-room*
   à partir du seul document d'analyse, sans copier de code ; (c) renoncer à iOS
   ou passer par une distribution alternative. **C'est la question la plus
   urgente, avant même la Phase 1.**
   → Note : les **assets** (modèles, textures, musique) ont leurs propres auteurs
   (9dan, Paul, Kaiser Foufou, Hulud, Garou). Leur réutilisation dans un jeu
   publié sur des stores demande un accord distinct.

2. **Remake fidèle ou évolution ?** Reproduire le *feel* de 2005 au pixel près,
   ou moderniser (nouvelles mécaniques, progression, cosmétiques) ?

3. **Modèle de partie.** On garde 16 joueurs / sessions de 60 s / serveurs
   persistants communautaires ? Ou on ajoute du solo, du contre-la-montre, de
   l'asynchrone — indispensables pour que le mobile ait un sens hors connexion ?

4. **Nom et identité.** « MTP Target » reste-t-il ? (marque, domaine, logo `.fla`
   présent dans le dépôt.)

5. **Priorité des plateformes.** Desktop d'abord puis mobile, ou mobile en
   parallèle dès la Phase 3 ? Mon avis : desktop d'abord jusqu'à la Phase 3
   incluse (itération plus rapide), mobile validé **dès la Phase 3** pour ne pas
   découvrir les contraintes réseau et tactiles trop tard.

6. **Scripting.** On garde Lua pour la moddabilité communautaire, ou on passe à
   des règles 100 % Rust (plus rapide, plus sûr, mais on perd la capacité de
   modding qui était une force du projet) ?

---

## 9. Mon avis, en une page

Ce dépôt **n'est pas une base de code à refactoriser** : c'est une
**spécification exécutable en lecture seule** d'un très bon jeu. Le
« refactoring » consiste à écrire un jeu neuf en se servant de celui-ci comme
document de conception détaillé.

Trois choses le rendent inhabituellement bien parti pour ça :

1. **Le contenu est en texte.** 24 niveaux et 21 scripts de règles en Lua : le
   level design et les modes de jeu survivent au changement de moteur.
2. **Le netcode est documenté et raisonné.** Le LCT, la quantification avec
   ré-injection, la séparation full/delta update : ce sont de bonnes idées, pas
   du bricolage. Elles se transposent directement.
3. **Le jeu est petit.** 36 maillages, une mécanique (ouvrir/fermer), une
   physique de sphère. C'est réalisable par une petite équipe — ce qui est faux
   de la quasi-totalité des jeux de 2005.

Trois choses sont dangereuses :

1. **Le format `.shape`.** Tant que le parseur n'existe pas, tout le reste est
   spéculatif. C'est le premier travail à faire.
2. **La licence face à l'App Store.** À régler avant d'investir dans iOS.
3. **La sensation de jeu.** Rapier ne se comportera pas comme ODE. Sans les
   trajectoires de référence de la Phase 0, on écrira un jeu qui ressemble à
   MTP Target sans en être un.

Ma recommandation : **Phase 0 et Phase 1 d'abord, en parallèle du contact avec
les auteurs originaux.** Trois à cinq semaines qui déterminent si le projet est
un portage ou une reconstruction — et ça change tout le reste.
