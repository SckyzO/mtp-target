# MTP Target — analyse en profondeur du code existant

> Document d'archéologie logicielle. Il décrit ce que le dépôt contient
> *réellement*, comment le jeu fonctionne, et ce qui est réutilisable comme
> source de vérité pour une réécriture.
>
> Toutes les affirmations ci-dessous ont été vérifiées dans le code ; les
> références sont données sous la forme `chemin:ligne`.

---

## 1. Identité du projet

MTP Target est un clone de **Monkey Target** (mini-jeu de *Super Monkey Ball*)
développé par le groupe **Melting Pot** à partir de 2003. Le joueur dévale une
rampe dans une boule-pingouin, décolle, ouvre la boule pour planer, puis la
referme pour atterrir sur une cible et marquer des points.

- Licence : **GNU GPL v2 or later** (`mtp-target/COPYING`, en-tête de chaque fichier source).
- Auteurs code : Vianney Lecroart (*Ace*), Alban Lecocq (*Skeet*), Muf.
- Historique git : **50 commits, du 2005-03-15 au 2005-06-22** — il s'agit d'une
  conversion d'un dépôt CVS, l'historique antérieur est perdu (plusieurs commits
  `*** empty log message ***`).
- Multijoueur : **16 joueurs max** (`mtp-target/server/mtp_target_service_default.cfg:17`),
  complétés par des bots quand il manque des humains.

Le dépôt est un mono-repo de 47 Mo contenant quatre sous-projets indépendants.

---

## 2. Inventaire

### 2.1 Volumétrie

| Type | Fichiers | Remarque |
|---|---|---|
| `.cpp` / `.c` | 122 | ~29 300 lignes |
| `.h` | 133 | ~20 800 lignes |
| `.lua` | 92 | niveaux + logique serveur + GUI |
| `.php` | 122 | site web + stats |
| `.shape` | 36 | maillages binaires NeL |
| `.max` | 18 | sources 3ds Max |
| `.tga` / `.dds` / `.png` / `.jpg` | 170 | textures + UI |
| `.wav` / `.mp3` | 7 | SFX + musique |

**Total code C/C++ : ~50 000 lignes.**

### 2.2 Modules du dépôt

```
mtp-target/                 le jeu
├── client/     131 fichiers  client 3D (NeL)
├── server/      54 fichiers  service de jeu autoritatif
├── common/      23 fichiers  code partagé client/serveur
├── chat/        14 fichiers  démon de chat autonome, en C pur (2001)
├── login_service/ 6 fichiers service d'authentification (NeL + MySQL)
├── bin/                      scripts shell de lancement
└── www/                      pages du serveur de jeu
mtp-target-original-data/     sources d'assets (.max, .psd, .fla)
mtp-target-web/               site officiel + statistiques (PHP)
wpkg-sync/                    patcheur/launcher Windows (MFC)
CVSROOT/                      résidu de la conversion CVS (à supprimer)
```

### 2.3 Dépendances externes (toutes datées de 2003-2005)

| Lib | Rôle | État en 2026 |
|---|---|---|
| **NeL** (Nevrax) | moteur 3D, réseau, sérialisation, service, config, log | Abandonné par Nevrax ; survit dans Ryzom Core |
| **ODE 0.5** + OPCODE | physique corps rigides + collision trimesh | Vivant mais figé, remplacé partout par PhysX/Bullet/Rapier |
| **Lua 5.0** (+ Lunar/Luna) | scripting niveaux & entités | Lua est vivant (5.4), les bindings sont à refaire |
| **STLport 4.5** | remplacement de la libstdc++ | Mort |
| **FMOD** (optionnel, `USE_FMOD`) | audio | Propriétaire, licence payante |
| **libxml2, freetype, curl, zlib** | XML GUI, polices, téléchargement, compression | Toujours là |
| **MySQL** | comptes + stats | OK |
| **NSIS**, **MFC** | installeur & launcher Windows | Windows-only |

> **Conclusion immédiate : le projet n'est plus buildable en pratique.** Il
> dépend d'un moteur (NeL 2005) et d'une STL alternative (STLport) qu'on ne
> peut plus raisonnablement reconstruire. Le dépôt vaut comme **spécification
> exécutable en lecture seule**, pas comme base de compilation.

---

## 3. Architecture d'ensemble

```
                        ┌───────────────────┐
                        │   Site PHP/MySQL  │  news, stats, classements,
                        │ (mtp-target-web)  │  système d'invitations
                        └─────────┬─────────┘
                                  │ MySQL
   ┌──────────┐  TCP/NeL   ┌──────┴────────┐
   │ Launcher │───────────▶│ login_service │  login/mot de passe, cookie,
   │ wpkg-sync│            │  (NeL + SQL)  │  bans, liste des serveurs
   └────┬─────┘            └──────┬────────┘
        │ HTTP (CRC + fichiers)   │ NeL Unified Network ("LS")
        ▼                         ▼
   ┌──────────────┐  TCP    ┌───────────────────┐  TCP  ┌─────────────┐
   │   client     │◀───────▶│ mtp_target_service│──────▶│ chat daemon │
   │ (NeL 3D)     │  jeu    │  physique ODE     │ bot   │   (C, 2001) │
   └──────────────┘         │  Lua, autoritatif │       └─────────────┘
                            └───────────────────┘
```

Quatre binaires + un site. Le couplage entre eux est **ad hoc** : le serveur de
jeu se connecte au chat comme un *client bot* sur un socket TCP en clair, avec
identifiants dans la config (`mtp-target/server/src/network.cpp:118`, adresse
`mtp-target.dyndns.org:4000` **en dur dans le code**).

---

## 4. Le serveur de jeu

### 4.1 Boucle principale

Le serveur est un *NeL service* (`NLNET_SERVICE_MAIN`, `server/src/main.cpp:174`).
Sa boucle `update()` (`server/src/main.cpp:139-162`) enchaîne, **en mono-thread** :

1. `CNetwork::update()` — émission des snapshots
2. `CLuaEngine::levelPreUpdate()`
3. `updatePhysics()`
4. `CEntityManager::update()`
5. `CSessionManager::update()` — machine à états de la partie
6. `CLevelManager::update()` + `levelPostUpdate()`
7. `CNetwork::sleep(1)` — pompe les sockets pendant ~1 ms

Le multithreading a été **désactivé mais pas retiré** : `PhysicsThread` existe
toujours (`server/src/physics.cpp:526`) mais son démarrage est commenté
(`//ace more thread`, ligne 566), et toutes les fonctions de pause/reprise
commencent par `// ace no thread \n return;` (`physics.cpp:648, 665, 690, 707`).
Il reste ainsi ~250 lignes de machinerie de synchronisation morte, plus
`CSynchronized<>` sur des données qui ne sont plus partagées.

### 4.2 Physique (ODE)

- **Pas fixe de 1 ms** (`physics.cpp:57`), rattrapage du temps réel par une boucle
  `for(step=0; step<nbLoop; step++)` où `nbLoop = deltaTime / worldStep`
  (`physics.cpp:370-382`). **Aucun plafond sur `nbLoop`** → si une frame prend
  200 ms, le serveur exécute 200 sous-pas, prend encore plus de retard :
  *spiral of death* classique.
- **La gravité est commutée par la machine à états de session** : elle vaut
  `0` à l'initialisation et pendant l'attente (`physics.cpp:545`,
  `waiting_clients_session_state.cpp:165`), puis est réglée à `Gravity`
  (`-0.981`) au démarrage de la session (`waiting_start_session_state.cpp:66`).
  Les joueurs flottent donc littéralement tant que la partie n'a pas commencé.
- En mode **planeur** (boule ouverte), la portance et la traînée ne sont pas
  simulées : elles sont calculées analytiquement à partir de l'angle de la
  commande, et la vitesse linéaire est **écrasée à zéro** à chaque sous-pas
  avant application des forces (`physics.cpp:414-441`). Le vol plané est donc
  un mouvement scripté, pas un résultat physique.
- `CFM` relevé à `1e-2` au lieu de `1e-5` (`physics.cpp:549`) pour éviter que les
  corps n'accrochent les arêtes entre deux modules — c'est un contournement
  d'un défaut de la géométrie de collision, pas un choix physique.
- **L'eau est un plan infini** `z = 0` (`physics.cpp:562`), pas un volume.
- Les joueurs sont des **sphères**, les décors des **trimesh** issus des `.shape`.
- Garde-fou contre l'explosion numérique : toute vélocité > 1000 est remise à
  zéro après chaque sous-pas (`physics.cpp:493-497`).
- `nearCallback` alloue `dContact contact[128]` sur la pile et **remplit les 128
  entrées** avant de savoir combien de contacts existent réellement
  (`physics.cpp:369` et boucle `physics.cpp:183`) — coût inutile à chaque paire.
- Les pointeurs utilisateurs des geoms sont castés sans discriminant :
  `(CEntity*)dGeomGetData(o1)` vs `(CModule*)dGeomGetData(o2)`, le type étant
  déduit de la présence d'un `dBodyID`. Une erreur de configuration produit un
  cast invalide silencieux.

### 4.3 Modèle de jeu

`CEntity` (`server/src/entity.h`) est la structure centrale : ~45 champs
**publics** avec le commentaire honnête `// ugly public variables`
(`entity.h:112`). On y trouve mêlés : identité réseau, apparence, score,
état physique ODE, état de session, état de ping, état d'interpolation réseau
(`LastSent2MePos`, `LastSent2OthersPos`), état Lua et compteurs anti-AFK.

Le cœur du gameplay tient en quelques champs :

| Champ | Rôle |
|---|---|
| `OpenClose` | boule fermée (roule/tombe) vs ouverte (plane) |
| `NbOpenClose` / `MaxOpenClose` | nombre d'ouvertures autorisées (1 par défaut) |
| `Force` | commande du joueur (direction + assiette), reçue du client |
| `CurrentScore` | score de la session en cours, **recalculé** par le Lua |
| `ArrivalTime` | temps avant immobilisation complète |
| `FreezeCommand` | l'entité ne répond plus aux commandes (crash) |
| `EnableCrashInFly` | toucher le décor en vol = crash, score 0 |

Règle notable (`physics.cpp:305-322`) : toucher un module **en mode ouvert** avec
`EnableCrashInFly` gèle l'entité, met son score à zéro et la sort du jeu.

`CEntityManager` est un objet-dieu de **1490 lignes** qui gère à la fois les
entités, la sérialisation réseau du login, les scores, les bots, le chat, les
commandes admin et les bans.

### 4.4 Machine à états de session

`CSessionManager` (`server/src/session_manager.h`) + un état par fichier :

```
WaitingClients ──▶ WaitingReady ──▶ WaitingStart ──▶ Running ──▶ Ending
       ▲                                                            │
       └────────────────────────────────────────────────────────────┘
```

À la fin (`running_session_state.cpp:156-175`), les scores partent vers le
login service pour être écrits en base, et `CLevelManager::updateStats()`
alimente les statistiques du site.

### 4.5 Scripting Lua — la couche la plus intéressante

Le serveur expose trois proxys aux scripts via des bindings *Lunar*
(`common/lunar.h`) :

- **`Entity`** : 34 méthodes (`server/src/entity_lua_proxy.cpp:49-82`) —
  `getPos/setPos`, `getIsOpen/setIsOpen`, `setCurrentScore`, `setOpenCloseMax`,
  `setMaxLinearVelocity`, `setDefaultAccel/Friction`, `displayText`,
  `getStartPointPos`, `getTeam`, `setFreezCommand`…
- **`Module`** : 17 méthodes (`module_lua_proxy.cpp:47-64`) — `setCollide`,
  `setBounce`, `setScore`, `setAccel`, `setFriction`, `setVisible`, `setPos`…
- **`Level`** : 4 méthodes (`level_lua_proxy.cpp:47-50`).

Les scripts implémentent des *callbacks* de gameplay :
`Entity:init()`, `Entity:preUpdate()`, `Entity:update()`,
`entitySceneCollideEvent(entity, module)`, `entityEntityCollideEvent()`,
`entityWaterCollideEvent(entity)`, `Module:collide(entity)`.

**C'est ici que vivent les règles de chaque mode de jeu.** Exemple complet
(`server/data/lua/level_arena_server.lua`) : dans l'arène, tomber dans l'eau
téléporte au point de départ au lieu d'éliminer.

Il y a **24 niveaux** (`server/data/level/*.lua`) et **21 scripts serveur**
(`server/data/lua/*_server.lua`) : arena, classic, classic_fight, darts, donuts,
extra_ball, hit_me, paint, race, run_away, snow_*, stairs, team, team_mirror,
the_lane, the_wall, wood…

Un fichier de niveau est du **Lua de données** :

```lua
Name = "Arena"; Author = "Skeet"; ServerLua = "level_arena_server.lua"
sunDirection = CVector(-1,0,-1); fogDistMax = 150;
StartPoints = { CVector(0.092517,-0.235331,0.616442), ... }   -- 16 points
Modules = {
  { Position = CVector(0,0,0.5), Scale = CVector(1,1,1),
    Rotation = CAngleAxis(1,0,0,0), Lua="arena", Shape="arena" }, ...
}
```

Les constructeurs `CVector`, `CRGBA`, `CAngleAxis` sont exposés par
`common/lua_nel.cpp`. **Ce format est trivialement réimplémentable** : c'est le
principal actif transposable du projet.

> **Attention à l'échelle** : `GScale = 0.01` (`client/src/global.h:31`) et la
> config indique « 1.0 = 100 mètres ». Les coordonnées des niveaux sont donc en
> **hectomètres** : `0.616442` ≈ 61,6 m. Toute réécriture doit décider si elle
> conserve cette unité ou convertit les 24 niveaux en mètres.

---

## 5. Le netcode — analyse détaillée

C'est la partie la plus travaillée du projet, et la plus datée.

### 5.1 Transport

**TCP uniquement**, via `NLNET::CBufServer` / `CBufClient`. Pas d'UDP, pas de
canal non fiable. Pour un jeu de positions à 25 Hz, c'est le mauvais choix :
une perte de paquet bloque toute la file (*head-of-line blocking*) et se traduit
par un gel visible.

### 5.2 Cadence

- La fonction `CNetwork::update()` sort immédiatement si moins de **40 ms** se
  sont écoulées (`server/src/network.cpp:141`) → **~25 Hz** effectifs.
- Les constantes déclarées (`common/constant.h`) annoncent une base de 10 Hz,
  doublée à 20 Hz pour l'entité locale (`MT_NETWORK_MY_UPDATE_FREQUENCE_RATIO 2`)
  et un *full update* tous les 200 ticks. Ces constantes **ne correspondent plus
  au garde-fou de 40 ms** : les deux mécanismes se superposent.

### 5.3 Trois familles de mises à jour

| Message | Contenu | Fréquence |
|---|---|---|
| `FullUpdate` | positions **absolues** de toutes les entités | tous les 200 ticks (~8 s) ou dès qu'un delta dépasse `MinDeltaToSendFullUpdate` (`network.cpp:277`) |
| `Update` | **deltas** de position des entités « actives » | 1 tick sur 2 |
| `UpdateOne` | delta de la position du destinataire uniquement | les autres ticks |

### 5.4 Compression : un format flottant maison

`common/custom_floating_point.cpp` implémente un **flottant 10 bits** :
4 bits d'exposant (`dx`) + 6 bits de mantisse (`sx`). Les trois axes sont
empaquetés dans un seul `uint32` via `packBit32` (`network.cpp:352-364`) :

```
[ dx.x:4 | sx.x:6 | dx.y:4 | sx.y:6 | dx.z:4 | sx.z:6 ] = 30 bits
```

Point crucial : le serveur **ré-injecte la valeur quantifiée** dans son propre
état de référence (`LastSent2OthersPos = LastSent2OthersPos + sendDPos`,
`network.cpp:388`) — client et serveur partagent donc exactement la même
position reconstruite, et l'erreur de quantification ne s'accumule pas. C'est
propre, et c'est une idée à conserver.

En cas de dépassement de capacité, `packBits` **saturre silencieusement**
(`custom_floating_point.cpp:74-79`, commentaire `//TODO SKEET`) : un mouvement
trop rapide est tronqué au lieu de déclencher un full update.

### 5.5 Interpolation côté client — le « LCT »

Documenté en français dans `mtp-target-web/doc/doc_lct.txt`. Le client
maintient une file de clés `(position, onWater, openClose, crashEvent)`
horodatées au temps serveur (`client/src/interpolator.h`), et affiche l'état à
`ServerTime = LocalTime - LCT` (`interpolator.cpp:240`) — c'est-à-dire **dans le
passé**, pour toujours interpoler entre deux clés reçues plutôt que d'extrapoler.

Le LCT est auto-ajusté entre **150 ms et 600 ms**
(`interpolator.cpp:114-115` : 3 à 12 périodes de 50 ms) selon la santé de la file.

`CExtendedInterpolator` dérive en plus l'**orientation visuelle** de la vitesse
interpolée (`interpolator.cpp:541-612`) : le serveur n'envoie **jamais de
rotation**, uniquement des positions. Les pingouins roulent et s'orientent par
un calcul purement client. Élégant, et très économe en bande passante.

### 5.6 Le protocole applicatif

24 types de messages (`common/net_message.h:56-81`), sérialisés par le
mécanisme `serial()` de NeL (`CMemStream`). Faiblesses :

- **Pas de schéma versionné.** Un champ `Version` existe mais l'ajout d'un champ
  casse toute compatibilité ; les callbacks lisent les champs dans l'ordre en
  espérant que l'autre bout écrit le même ordre. Le client attrape une exception
  et logue `Malformed Message` (`network.cpp:534`).
- **`ExecLua` (type 22) est un message serveur → client qui exécute du Lua
  arbitraire chez le joueur** (`client/src/net_callbacks.cpp:670-678`). C'est
  une exécution de code à distance par conception : un serveur communautaire
  hostile contrôle totalement les clients connectés.
- Le dispatch serveur est en **O(n) par message** : pour chaque paquet reçu, on
  parcourt toutes les entités pour retrouver le socket émetteur
  (`network.cpp:498-522`). Idem dans `Update`, où pour chaque id de la liste on
  reparcourt toutes les entités (`network.cpp:333-341`) → O(n²) par tick.

### 5.7 Modèle d'autorité

**Serveur strictement autoritatif, aucune prédiction client.** Le client envoie
une `Force` (message `Force`, type 7) et attend la position en retour. Combiné
au LCT de 150-600 ms, cela signifie que **l'action du joueur n'apparaît à
l'écran qu'après un aller-retour complet + le retard d'interpolation**. Sur le
réseau de 2005 c'était acceptable ; en 2026 c'est le premier défaut de
« feel » à corriger.

---

## 6. Le client

### 6.1 Architecture en tâches

Un `CTaskManager` orchestre 15 `ITask` (`client/src/task.h`), chacune avec
`init/update/render/release` : `C3DTask`, `CNetworkTask`, `CGameTask`,
`CGuiTask`, `CHudTask`, `CChatTask`, `CScoreTask`, `CIntroTask`, `CSkyTask`,
`CWaterTask`, `CBackgroundTask`, `CLensFlareTask`, `CTimeTask`, `CEditorTask`,
`CConfigFileTask`. C'est un pattern propre et lisible — l'équivalent moderne
serait un ordonnancement de systèmes ECS.

### 6.2 Rendu

Tout passe par l'API haut niveau de NeL (`UDriver`, `UScene`, `UInstance`), avec
choix OpenGL ou Direct3D à l'exécution (`client/src/3d_task.cpp:138-144`). Il n'y
a **aucun shader écrit par le projet** : les effets (eau « pixel shader », nuages
dynamiques, lens flares) sont des fonctionnalités de NeL pilotées par la config.
Conséquence directe pour la réécriture : **il n'y a pas de code de rendu à
porter, il y a un rendu à recréer.**

### 6.3 GUI maison

~25 fichiers `gui_*.cpp` (≈ 6 000 lignes) implémentent un toolkit retenu complet :
`gui_box`, `gui_button`, `gui_listview`, `gui_scrollbar`, `gui_text`,
`gui_progress_bar`, `gui_stretched_quad`, `gui_xml`, `gui_script`… avec mise en
page **XML** (`client/data/gui/*.xml`) et comportements **Lua**. C'est un travail
considérable, entièrement remplaçable aujourd'hui par un toolkit immédiat
(egui) pour un coût proche de zéro.

### 6.4 Distribution des assets

Système de patch maison (`client/src/resource_manager2.cpp`) :
1. le client télécharge un fichier de CRC par HTTP via **libcurl** (ligne 646) ;
2. il compare le CRC de chaque fichier local ;
3. en cas d'écart il demande au serveur (`RequestCRCKey`, `RequestDownload`) puis
   télécharge le fichier compressé.

C'est un gestionnaire de paquets rudimentaire, doublé par `wpkg-sync`, un
launcher **MFC Windows-only**. Aucune signature : le canal HTTP en clair pousse
des fichiers exécutés par le client (y compris du Lua).

### 6.5 Audio

`CSoundManager` (608 lignes) entièrement conditionné par `#ifdef USE_FMOD`
(`client/src/sound_manager.cpp:53, 72, 174, 241`). Sans FMOD, le jeu est muet.
FMOD est propriétaire — c'est une incompatibilité de fait avec la GPL du reste
du projet, et une raison suffisante de repartir sur autre chose.

---

## 7. Les services annexes

### 7.1 `login_service`

Service NeL + MySQL (`login_service/connection_client.cpp`). Il gère
inscription implicite (le premier login crée le compte, ligne 211),
authentification, cookies de session, bans par IP et durée.

Problèmes de sécurité, à connaître avant de réutiliser quoi que ce soit :

- **Injection SQL** : toutes les requêtes sont construites par concaténation de
  chaînes (`"select * from user where Login='"+login+"'"`, ligne 203). La seule
  protection est `checkLogin()` (ligne 143), une liste blanche de caractères.
- **`crypt()` avec un sel fixe** partagé par tous les comptes (ligne 97) →
  DES tronqué à 8 caractères significatifs sur les systèmes historiques, et
  hachages comparables entre utilisateurs.
- Mots de passe limités à 20 caractères, espaces interdits (ligne 111-121).

### 7.2 Démon de chat

`mtp-target/chat/` : serveur de chat autonome en **C ANSI de 2001**, avec ses
propres `list.c`, `socket.c`, `crypt.c`, `database.c`. Fonctionne comme un
mini-IRC (canaux, groupes, tokens). Le serveur de jeu s'y connecte comme un bot.
Aucune raison de le conserver.

### 7.3 Site web

122 fichiers PHP procéduraux : news, classements, statistiques par jour/heure/
carte/clan, système d'invitations, galerie de captures. Les requêtes SQL sont là
aussi concaténées à la main (`mysql-func.php`). Valeur du code : nulle.
**Valeur du modèle de données : réelle** — le schéma des stats (score par carte,
par joueur, par session, temps d'arrivée) décrit ce que les joueurs suivaient
vraiment.

---

## 8. Les assets — le vrai point dur

C'est le seul élément **non reproductible** du projet.

| Ressource | Format | Réutilisable ? |
|---|---|---|
| 36 maillages | `.shape` (binaire NeL, 1-12 Ko, ~1 Mo au total) | Nécessite un parseur du format de sérialisation NeL |
| 18 sources 3D | `.max` (3ds Max ~5, 2003) | Nécessite 3ds Max ; couvre **la moitié** des shapes seulement |
| Textures | `.tga`, `.dds`, `.psd` | Oui, formats ouverts |
| Sons | `.wav`, `.mp3` | Oui |
| Polices | `.ttf`, `.pfb` | Oui, sous réserve de licence |
| Niveaux | `.lua` | Oui, format texte |

`mtp-target-original-data/README` confirme l'intention : ce module contient
« toutes les données nécessaires pour régénérer les données binaires NeL
finales ». Mais l'inventaire montre **18 `.max` pour 36 `.shape`** : les sources
de la moitié des maillages (arena, boxae, race_*, wood_*, team_target_*,
box_sol, col_box, sky) **ne sont pas dans le dépôt**.

Autre subtilité : les `.shape` servent **aussi** de géométrie de collision. Le
serveur les charge et les convertit en trimesh ODE (`common/load_mesh.cpp`), et
il en extrait des « auto-edges » (`CAutoEdge`, `load_mesh.cpp:60-135`) : le
centre et la normale des faces coplanaires, utilisés par l'éditeur pour **aimanter
les modules entre eux**. Si on perd les shapes, on perd à la fois le visuel, la
collision et l'aimantation de l'éditeur.

---

## 9. Dette technique et pièges

Classés par gravité pour une réécriture.

1. **Code mort de multithreading** (~250 lignes) : `PhysicsThread`, `pauseAll*`,
   `CSynchronized<>` inutiles. Ne pas les porter, ne pas s'en inspirer.
2. **Objets-dieu** : `CEntityManager` (1490 l.), `CEntity` avec 45 champs publics.
3. **Singletons partout** (`NLMISC::CSingleton`, `getInstance()` à chaque ligne) :
   état global implicite, impossible à tester unitairement, impossible de faire
   tourner deux parties dans un processus.
4. **Constantes contradictoires** entre `constant.h` et le garde-fou 40 ms.
5. **Endpoints en dur** : `mtp-target.dyndns.org:4000` dans le code du serveur.
6. **Sécurité** : injection SQL, `crypt()` à sel fixe, `ExecLua` distant,
   téléchargement d'assets non signé sur HTTP.
7. **Boucle physique non bornée** : *spiral of death* garantie sous charge.
8. **Complexité O(n²)** dans la boucle réseau à chaque tick.
9. **Commentaires-aveux** disséminés : `// ugly public variables`,
   `//TODO SKEET`, `//SKEET_WARNING`, `// ace no thread`. Ce sont des marqueurs
   utiles : ils signalent exactement les zones que les auteurs savaient fragiles.
10. **Aucun test.** Pas un seul fichier de test dans les 50 000 lignes.

---

## 10. Ce qui a de la valeur — et ce qu'il faut jeter

### À conserver comme spécification

| Actif | Pourquoi |
|---|---|
| **Les 24 niveaux + 21 scripts Lua** | C'est le contenu, le level design et les règles des modes de jeu. Format texte, portable. |
| **L'API Lua (55 méthodes sur 3 proxys)** | Contrat de moddabilité déjà éprouvé par 21 scripts. À reprendre presque tel quel. |
| **Le modèle LCT / interpolation** | Solution correcte et bien documentée au problème de lissage. |
| **La quantification 10 bits avec ré-injection** | Idée juste : le serveur suit la valeur quantifiée, pas la valeur réelle. |
| **La machine à états de session** | 5 états, simple et suffisante. |
| **Le paramétrage physique** | Les valeurs de rebond, friction, portance, `MaxOpenClose` par niveau *sont* le game feel. À extraire numériquement avant toute réécriture. |
| **Les textures, sons, `.max`, `.psd`** | Assets originaux irremplaçables. |
| **Le schéma de stats** | Décrit ce que la communauté suivait. |

### À jeter sans regret

NeL, ODE, STLport, FMOD, le toolkit GUI XML maison, le démon de chat en C,
le launcher MFC, le site PHP, le système de CRC/patch, `CVSROOT/`, tout le
code de threading mort, et l'ensemble des `#ifdef NL_OS_WINDOWS`.

---

## 11. Ce qu'il faut mesurer *avant* de réécrire

La physique de MTP Target n'est pas dans un design document : elle est dans les
constantes d'ODE et dans l'interaction entre le pas de 1 ms, le CFM à `1e-2` et
les forces analytiques du mode planeur. Un moteur moderne (Rapier) donnera
**des trajectoires différentes** avec les mêmes chiffres.

Il faut donc, avant d'écrire la moindre ligne de Rust, extraire du code les
valeurs de référence : `OpenAccelCoef`, `OpenZSpeed`, `OpenMinHSpeed`,
`OpenMinZSpeed`, `OpenMinAngleToZSpeed`, `Gravity`, `SphereDensity`,
`BounceWater/Client/Scene` et leurs `*_vel`, `MinVelBeforeEnd`, `ModuleFriction`
(`server/src/variables.cpp`, `server/mtp_target_service_default.cfg`, et les
surcharges par niveau via `optionCallback`, `physics.cpp:82-110`), puis définir
un jeu de **trajectoires de référence** (départ, commandes, position finale) qui
servira de test de non-régression du *feel*.

C'est le point le plus important de tout ce document : **le risque n° 1 d'une
réécriture n'est pas technique, il est de perdre la sensation de jeu.**
