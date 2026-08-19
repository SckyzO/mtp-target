# Document de conception — jeu inspiré de MTP Target

> **Statut : document vivant.** Il fige ce qui est décidé, nomme ce qui ne l'est
> pas, et sert de garde-fou contre l'élargissement du périmètre.
>
> **Ce document décrit un jeu neuf.** Il ne réutilise ni le code, ni les assets,
> ni le nom de MTP Target (2003-2005, GPL v2+, assets appartenant à leurs
> auteurs). L'original sert de référence de conception, analysée dans les
> documents [01](01-analyse-existant.md) et [03](03-reference-gameplay.md).
> Ce fichier a vocation à déménager dans le dépôt propre du nouveau jeu.
>
> **Nom de code : `PROJET-PINGOUIN`.** Un vrai nom reste à trouver.

---

## 1. Le pitch

Huit pingouins en boule dévalent une rampe géante, décollent, déploient leurs
ailes pour planer, puis se referment pour plonger sur une cible. Le plus précis
gagne — sauf si un copain le percute au dernier moment.

Trois minutes par manche. Jouable seul contre des bots, ou entre amis avec un
code de partie.

**La promesse :** un jeu d'adresse simple à comprendre, difficile à maîtriser,
où l'échec est aussi drôle que la réussite.

---

## 2. Contraintes du projet

Ce sont elles qui dictent tout le reste du document.

| Contrainte | Valeur | Conséquence |
|---|---|---|
| **Équipe** | 1 personne | Aucune tâche ne peut dépendre d'une compétence absente |
| **Temps** | soirs et week-ends, ~10 h/semaine | ~500 h par an. Le périmètre est la seule variable d'ajustement |
| **Compétence art** | à sous-traiter | **Le plus gros risque du projet.** L'original avait trois graphistes |
| **Plateformes** | Windows, Linux, macOS, Android, iOS | Le tactile est une contrainte de conception, pas un portage |
| **Modèle** | Steam premium 5-8 € + mobile gratuit avec pub | La version PC est le produit, le mobile est le canal |
| **Moteur** | Godot 4 | MIT, zéro royalties, export mobile documenté, physique Jolt |
| **Budget d'exploitation** | quelques euros par mois | Impose l'hébergement par les joueurs eux-mêmes |

> **La règle qui prime sur toutes les autres :** à 10 h par semaine, le projet
> ne meurt pas d'un mauvais choix technique. Il meurt d'un périmètre trop large.
> Chaque fois qu'une idée apparaît dans ce document, la question n'est pas
> « est-ce bien ? » mais « qu'est-ce que je retire en échange ? »

---

## 3. Le jeu

### 3.1 Anatomie d'une manche

```
  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
  │  DÉPART  │──▶│  ROULER  │──▶│  PLANER  │──▶│  PLONGER │──▶│  SCORE   │
  └──────────┘   └──────────┘   └──────────┘   └──────────┘   └──────────┘
   8 pingouins    accélérer      ailes         se refermer    cible touchée
   en haut de     et diriger     déployées,    → chute,       = points
   la rampe       sur la rampe   on plane      on rebondit    eau = 0
                                 vers la cible  et on s'arrête
```

Une **partie** = 5 manches, score cumulé, classement final. Environ 3 à 5 minutes.

### 3.2 La seule mécanique qui compte

Tout le jeu tient dans **une décision, prise deux fois** : *quand j'ouvre, et
quand je referme*.

- **Ouvrir trop tôt** : on plane longtemps mais on perd la vitesse du décollage,
  on n'atteint pas les cibles lointaines.
- **Ouvrir trop tard** : on a la vitesse mais plus l'altitude pour corriger.
- **Refermer trop tôt** : on tombe court.
- **Refermer trop tard** : on arrive trop vite, on rebondit et on dépasse la cible.

C'est le cœur. Tout le reste — les niveaux, les bots, les cosmétiques — est au
service de cette tension.

**Règle de conception :** aucune fonctionnalité n'entre dans le jeu si elle ne
rend pas cette décision plus intéressante.

### 3.3 Ce qu'on garde de l'original, et ce qu'on jette

| On garde | Pourquoi |
|---|---|
| La boucle rouler / planer / plonger | C'est le jeu |
| Les collisions entre joueurs | C'est ce qui fait rire, et c'est ce qui distingue le multi du solo |
| L'eau qui annule le score | Enjeu clair, lisible, immédiat |
| Le remplissage par des bots | Permet de jouer seul et de ne jamais avoir une partie vide |
| Des cibles de valeurs différentes | Crée l'arbitrage risque/récompense |

| On jette | Pourquoi |
|---|---|
| 16 joueurs | Tes amis sont 2 à 5. 8 suffit et allège le mobile |
| 24 modes de jeu | Un seul mode, bien réglé, à la sortie |
| Le crash en vol qui annule tout | Punition trop sèche pour un jeu grand public. À remplacer par une pénalité de score |
| Les comptes obligatoires | On joue sans créer de compte |
| L'éditeur de niveaux intégré | Peut-être un jour. Pas en v1 |

### 3.4 Le modèle de vol

**On ne reprend pas la formule de 2005.** Elle écrasait la vélocité à zéro à
chaque pas de 1 ms — un bricolage conçu pour un clavier.

On conçoit un modèle neuf, avec trois exigences :

1. **Lisible.** Le joueur doit comprendre en trois manches pourquoi il a raté.
2. **Tactile.** Contrôlable au pouce, sans précision au pixel.
3. **Réglable.** Toutes les constantes dans un fichier de données, pas dans le code.

Point de départ proposé, à valider en playtest : portance proportionnelle à la
vitesse horizontale, traînée croissante avec l'assiette cabrée, et une vitesse
de chute minimale garantie pour que la manche se termine toujours.

Les valeurs de l'original ([document 03](03-reference-gameplay.md)) servent
d'ordres de grandeur, pas de cible. Notamment : la gravité y valait environ dix
fois celle de la Terre, ce qui donnait un jeu rapide. C'est un choix de rythme
à reproduire consciemment.

### 3.5 Les contrôles

Conçus pour le pouce d'abord, le clavier ensuite.

| Phase | Tactile | Clavier | Manette |
|---|---|---|---|
| Rouler | glissement horizontal ou inclinaison | ← → | stick gauche |
| Planer | stick virtuel (direction + assiette) | ← → ↑ ↓ | stick gauche |
| Ouvrir / refermer | **un gros bouton à droite** | Espace | A / X |
| Caméra | glissement libre à droite | souris | stick droit |

Une seule action discrète dans tout le jeu. C'est ce qui le rend jouable d'une
main dans le métro — et c'est un atout à ne pas gâcher en ajoutant des verbes.

---

## 4. Les bots

Sur mobile, la majorité des parties se joueront hors ligne. **La qualité des
bots est donc la qualité du jeu** pour la plupart des joueurs. Ce n'est pas une
tâche de fin de projet.

### 4.1 Ils ne sont pas là pour gagner

Un bot optimal est ennuyeux et décourageant. **Le rôle d'un bot est de créer du
chaos et des histoires.** Trois bots qui visent la même cible et se percutent en
vol valent mieux que trois bots qui atterrissent parfaitement.

### 4.2 Conception

Techniquement, c'est de la balistique, pas de l'IA :

1. **Choisir une cible** selon un tempérament (voir plus bas)
2. **Calculer une trajectoire** vers elle
3. **Ajouter du bruit** proportionnel à la difficulté
4. **Ne pas corriger** les erreurs en cours de vol au-delà d'un certain seuil —
   c'est ce qui produit les échecs spectaculaires

### 4.3 Tempéraments

Trois personnalités, tirées au sort, affichées par le nom ou la couleur :

| Tempérament | Comportement |
|---|---|
| **Prudent** | Vise la cible sûre, réussit souvent, marque peu |
| **Ambitieux** | Vise la cible la plus payante, échoue souvent, spectaculaire |
| **Chaotique** | Vise la cible où il y a déjà quelqu'un. Cherche la collision |

### 4.4 Difficulté

Quatre niveaux, réglés par l'amplitude du bruit. **Pas d'adaptation dynamique en
v1** — l'original le faisait, c'est difficile à équilibrer et impossible à
déboguer seul.

---

## 5. Architecture réseau

### 5.1 Le principe fondateur

**Il n'existe qu'un seul mode de jeu.** Un joueur héberge la partie et fait
tourner la simulation faisant autorité. Le nombre de joueurs distants connectés
peut valoir zéro.

```
   HORS LIGNE                        EN LIGNE
   ┌─────────────────┐               ┌─────────────────┐
   │ Hôte (le joueur)│               │ Hôte (le joueur)│
   │  simulation     │               │  simulation     │
   │  + 7 bots       │               │  + 3 copains    │
   │  + 0 connecté   │               │  + 4 bots       │
   └─────────────────┘               └────────┬────────┘
                                              │ relais
                                     ┌────────┴────────┐
                                     │ 3 clients       │
                                     └─────────────────┘
```

**Le hors ligne n'est pas un mode dégradé, c'est le cas général avec un
compteur à zéro.** Un seul chemin de code, une seule simulation, et les bots
sont testés à chaque partie jouée.

### 5.2 Le transport

Décision : **relais dès le premier jour, pas de P2P.**

C'est le choix d'Among Us, et pour la bonne raison : un relais fonctionne
**toujours**, derrière n'importe quel NAT, y compris le CGNAT des opérateurs
mobiles. Une connexion directe échoue chez une partie des joueurs, de façon
impossible à diagnostiquer à distance. Pour un jeu qu'on vend, ce mode d'échec
est inacceptable.

| Étape | Ce qu'on construit | Coût |
|---|---|---|
| **Prototype** | IP directe en réseau local | Zéro |
| **v1 desktop** | Relais Steam (gratuit, inclus) | Zéro |
| **v1 mobile** | Notre relais, ou un service tiers | Un petit VPS |
| **Plus tard** | Promotion en connexion directe | *Voir 5.5* |

### 5.3 Le code de partie

Six caractères, sans ambiguïté visuelle (pas de `0`/`O`, pas de `1`/`I`).
L'hôte le génère, le partage par le moyen qu'il veut, les copains le tapent.

Pas de compte requis. Pas de liste d'amis à construire. C'est le point de
friction le plus bas possible, et c'est ce qui fait vivre les jeux entre potes.

### 5.4 Économie de bande passante

**La quantification des positions est une décision du premier jour, pas une
optimisation tardive** : c'est elle qui détermine la facture d'hébergement.

En reprenant le principe de l'original — un delta de position comprimé sur
quelques octets pour les trois axes, avec réinjection de la valeur quantifiée
dans la référence de l'émetteur pour éviter toute dérive :

| Charge | Bande passante | Coût |
|---|---|---|
| 1 partie à 8, 20 Hz | ~12 Ko/s | — |
| 100 parties simultanées | ~10 Mbit/s | VPS à 5 €/mois |
| 1 000 parties simultanées | ~100 Mbit/s | machine dédiée à ~40 €/mois |

Sans quantification, multiplier par cinq ou six.

### 5.5 Ce qu'on prépare sans le construire

**Une couche transport qui ignore son itinéraire.** Une interface
`envoyer / recevoir` derrière laquelle on peut brancher : réseau local, relais,
ou connexion directe.

Cette couture coûte une interface aujourd'hui. Elle permet d'ajouter plus tard
la **promotion en connexion directe** : la partie démarre via le relais, une
connexion directe est tentée en arrière-plan, et le trafic bascule dessus si
elle aboutit — sans que le joueur voie quoi que ce soit.

**Ce n'est pas un jalon.** Sur mobile ça n'aboutira quasiment jamais (CGNAT),
sur Steam le relais le fait déjà, et une connexion directe expose les adresses
IP des joueurs. Le gain réel est faible ; on garde seulement la porte ouverte.

### 5.6 Ce qu'on assume

| Compromis | Pourquoi c'est acceptable |
|---|---|
| **L'hôte a zéro latence, les autres ont du ping** | Partie entre amis, pas de compétition classée. À indiquer honnêtement dans l'interface |
| **L'hôte part, la partie s'arrête** | La migration d'hôte coûte cher pour un bénéfice marginal ici |
| **L'hôte peut tricher** | Entre copains, personne ne s'en soucie |
| **Pas de classement mondial** | Conséquence directe du point précédent. Si classement un jour, il portera sur le solo validé côté serveur |

---

## 6. Monétisation

Modèle retenu, calqué sur ce qui fonctionne déjà pour Among Us : **le mobile
fait le volume, le PC fait l'argent.**

### 6.1 Le modèle de référence, correctement lu

Among Us est la référence la plus proche. Attention à ne pas s'arrêter au
modèle de 2020, qui est souvent celui décrit en ligne :

- **En 2020** : 5 $ sur Steam débloquaient l'ensemble des cosmétiques payants
  du mobile. Le PC ne représentait que ~3 % de la base de joueurs mais une
  large part des revenus.
- **Depuis 2021-2022** : une économie cosmétique complète existe **sur toutes
  les plateformes, PC compris** — DLC payants sur Steam entre 0,99 et 2,99 $,
  monnaie premium (Stars) achetable en argent réel, Cosmicubes achetés avec
  ces Stars, boutique à deux monnaies. Environ 75 % des joueurs actifs y
  touchent.

Détail structurel intéressant : **les Stars s'obtiennent en regardant des
publicités sur mobile, et s'achètent en argent sur PC.** Le joueur mobile paie
en attention, le joueur PC en euros, et les deux alimentent la même économie.

**Ce qu'on en retient :** le prix d'achat n'est pas le produit fini, c'est le
ticket d'entrée. **Ce qu'on n'en retient pas :** l'économie qui va avec.

### 6.2 Pourquoi on n'en copie que la moitié

Une économie de cosmétiques vivante est du **LiveOps permanent** : contenu
saisonnier, événements, collaborations. C'est un studio qui livre en continu.
À 10 h par semaine en solo, c'est un tapis roulant intenable — et une boutique
abandonnée fait plus de mal que pas de boutique.

L'autre moitié du problème est arithmétique : 75 % de plusieurs millions de
joueurs est une entreprise ; 75 % de cinq cents joueurs est zéro. **Une
économie cosmétique a besoin d'une audience avant d'avoir besoin d'un
catalogue.**

### 6.3 Steam — le produit

Prix : **5 à 8 €**, incluant un lot de cosmétiques suffisant pour que le jeu
paraisse complet. Aucune publicité, aucune monnaie, aucune boutique intégrée.

**Après la sortie**, et seulement si le jeu trouve son public : un ou deux
**packs cosmétiques en DLC Steam** à 1-3 €. C'est le modèle Polus Skins, et
c'est le seul morceau de l'économie d'Among Us qui tient dans ce budget — pas
de monnaie, pas de boutique, pas de backend, pas de synchronisation des droits
entre plateformes. Steam gère les droits, on dépose un pack.

### 6.4 Mobile — le canal

Gratuit. Deux sources :

- **Publicité récompensée**, à la fin d'une partie uniquement. Jamais pendant
  une manche, jamais dans une partie en ligne entre amis.
- **Un achat unique** « sans publicité + tous les cosmétiques ».

### 6.5 Cosmétiques

Couleurs et motifs de coquille, traînées de vol. Pas de contenu payant qui
touche au gameplay — jamais.

> **Point d'attention.** Les cosmétiques ne rapportent que s'ils sont vus.
> S'ils ne sont visibles qu'en solo contre des bots, ils ne se vendront pas.
> **C'est un argument de conception, pas de monétisation : la partie entre amis
> doit être la vedette du jeu, et le solo le mode d'entraînement.**

### 6.6 Ce qu'on ne fait pas

Pas de monnaie premium, pas de boutique intégrée, pas de coffres, pas de passe
saisonnier, pas d'énergie, pas de pub interstitielle forcée.

Ce sont exactement les systèmes qu'Among Us a ajoutés — et ils les ont ajoutés
**après** avoir eu des millions de joueurs et une équipe pour les entretenir.
Dans cet ordre. On garde la porte ouverte, on ne construit rien avant d'avoir
l'audience qui le justifie.

## 7. Périmètre de la v1

### 7.1 Dans la v1

- 1 mode de jeu : score sur cibles, 5 manches
- **5 niveaux**
- 8 participants (joueurs + bots)
- Solo hors ligne contre bots, 4 difficultés
- Partie en ligne par code, jusqu'à 8
- Partie en réseau local
- Une poignée de cosmétiques
- 5 plateformes

### 7.2 Hors v1, explicitement

Éditeur de niveaux · modes d'équipe · classements mondiaux · comptes joueurs ·
chat textuel ou vocal · replays · progression et niveaux de compte · saisons ·
migration d'hôte · connexion directe P2P · portage console · localisation
au-delà du français et de l'anglais.

Chacun de ces points est une bonne idée. Chacun coûte des mois. Ils attendront
d'avoir des joueurs qui les réclament.

---

## 8. Jalons

À ~10 h par semaine. Les durées sont des ordres de grandeur, pas des engagements.

| # | Objectif | Critère de sortie | Durée |
|---|---|---|---|
| **M1** | **Le noyau est-il amusant ?** Un cube roule, décolle, plane, plonge sur une cible. Art bidon, un seul niveau, pas de bots, pas de réseau | **Cinq personnes y jouent vingt minutes et veulent recommencer.** Si non → on retravaille le modèle de vol, on n'avance pas | 2-3 mois |
| **M2** | Boucle de jeu complète : bots, score, 5 manches, 3 niveaux | Une partie solo complète est jouable et se termine | 2-3 mois |
| **M3** | Multijoueur en réseau local, IP directe | Deux machines du même Wi-Fi jouent ensemble | 3-4 mois |
| **M4** | Relais et code de partie | Deux joueurs sur deux réseaux différents jouent avec un code | 3 mois |
| **M5** | Habillage : art définitif, son, interface, build mobile | Le jeu tourne sur un vrai téléphone et se montre sans rougir | 4-6 mois |
| **M6** | Publication : pages boutique, pub, achat intégré, tests | Disponible sur Steam et sur les deux stores mobiles | 3 mois |

**Total : de l'ordre de 18 à 24 mois**, si le périmètre tient.

### La règle de M1

**M1 est un test, pas une étape.** Un jeu d'adresse dont le noyau n'est pas
amusant avec un cube gris ne le sera pas davantage avec un joli pingouin. Si
M1 échoue, l'échec est peu coûteux — et c'est exactement pour ça qu'il vient
en premier.

---

## 9. Risques

Par ordre de probabilité de tuer le projet.

| # | Risque | Parade |
|---|---|---|
| 1 | **Le périmètre gonfle** | Ce document. La liste §7.2 est un contrat avec toi-même |
| 2 | **L'art** — aucune compétence dans l'équipe, et l'original avait trois graphistes | Budgéter de la sous-traitance dès M2. Style volontairement simple et stylisé, pas réaliste. Ne pas attendre M5 pour poser la question |
| 3 | **L'essoufflement** sur 18-24 mois en soirée | Des jalons courts et jouables. Montrer le jeu tôt et souvent |
| 4 | **Le noyau n'est pas amusant** | C'est précisément ce que M1 mesure, avant tout investissement |
| 5 | **Le juridique** — GPL, assets, nom de l'original | Dépôt neuf, code neuf, assets neufs, nom neuf. Aucune ligne ne traverse |
| 6 | **La plomberie de monétisation** sur mobile | Prévoir M6 large. C'est la partie la plus ingrate et la plus sous-estimée |
| 7 | Le relais coûte cher | Quantification dès le premier jour. Le mode réseau local comme porte de sortie |

---

## 10. Décisions figées

| Sujet | Décision | Quand |
|---|---|---|
| Repartir de zéro | Oui, dépôt neuf | ✅ |
| Moteur | Godot 4 | ✅ |
| Modèle économique | Steam premium + mobile gratuit avec pub | ✅ |
| Économie cosmétique | Pas de monnaie ni de boutique. DLC Steam éventuels après la sortie | ✅ |
| Hors ligne | Bots, pas de ghosts | ✅ |
| En ligne | Hôte-joueur + code de partie | ✅ |
| Transport | Relais, pas de P2P | ✅ |
| Taille de partie | 8 | ✅ |

## 11. Décisions ouvertes

1. **Le nom.** Bloque le dépôt, le domaine, les pages boutique, l'identité visuelle.
2. **La direction artistique.** Détermine le budget de sous-traitance — le
   deuxième risque du projet. À trancher avant M2, pas à M5.
3. **Le personnage.** Un pingouin, comme l'hommage l'appelle ? Ou une créature
   propre au jeu, plus facile à défendre juridiquement et à décliner en
   cosmétiques ?
4. **La pénalité de crash en vol.** L'original annulait tout le score. Trop sec.
   Quelle punition à la place ?
5. **Le relais mobile** : construit sur mesure ou service tiers ? À trancher à M4.
