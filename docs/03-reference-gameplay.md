# MTP Target — référence de gameplay

> **Livrable de la Phase 0.** Ce document extrait du code toutes les valeurs qui
> définissent la sensation de jeu. Il sert de contrat pour la réécriture : la
> version Rust devra reproduire ces comportements, puis être re-tunée en
> playtest car Rapier ne se comporte pas comme ODE.
>
> Sources : `mtp-target/server/mtp_target_service_default.cfg`,
> `server/src/variables.{h,cpp}`, `server/src/physics.cpp`,
> `server/src/entity.cpp`, `common/constant.h`.

---

## 1. Unités du monde

`GScale = 0.01` (`client/src/global.h:31`) et la config client indique
« 1.0 = 100 mètres ».

**1 unité monde = 100 m.** Toutes les valeurs ci-dessous sont dans cette unité,
sauf mention contraire.

| Grandeur | Valeur brute | Équivalent métrique |
|---|---|---|
| Rayon du pingouin | `0.01` (`entity.cpp:125`) | **1 m** (boule de 2 m de diamètre) |
| Position de départ typique | `z = 0.616442` | ~62 m d'altitude |
| Plan d'eau | `z = 0` | niveau 0 |
| Portée du brouillard | `50.0` → `150.0` | 5 km → 15 km |

> **Point d'attention n° 1 pour le portage.** `Gravity = -0.981` en unités monde
> vaut **-98,1 m/s²**, soit **10 fois la gravité terrestre**. Ce n'est pas une
> erreur de conversion des auteurs mais un choix de rythme : le jeu est
> délibérément rapide. Il faudra reproduire ce rapport, pas « corriger » la
> gravité.

---

## 2. Constantes physiques

Toutes déclarées via la macro `DEC_VAR` (`server/src/variables.h:39-70`),
lues depuis la config, et **surchargeables par niveau** via `optionCallback`
(`physics.cpp:82-110`).

### 2.1 Intégration

| Paramètre | Valeur | Source |
|---|---|---|
| Pas de simulation | **1 ms** fixe | `physics.cpp:57` |
| Plafond de sous-pas par frame | **aucun** ⚠ | `physics.cpp:370` |
| Gravité monde (session en cours) | `-0.981` sur Z | `waiting_start_session_state.cpp:66` |
| Gravité monde (attente) | `0` | `physics.cpp:545`, `waiting_clients_session_state.cpp:165` |
| ERP | valeur ODE par défaut | `physics.cpp:548` (commenté) |
| CFM | `1e-2` (défaut ODE : `1e-5`) | `physics.cpp:549` |
| Espace de collision | `dHashSpace` | `physics.cpp:553` |
| Eau | plan infini `z = 0` | `physics.cpp:562` |
| Garde-fou vitesse | toute composante > `1000` → vélocité remise à zéro | `physics.cpp:493-497` |

### 2.2 Corps du joueur

| Paramètre | Valeur | Source |
|---|---|---|
| `SphereDensity` | `20.0` | config |
| Rayon | `0.01` | `entity.cpp:125` |
| Masse | `dMassSetSphere(density=20, r=0.01)` | `entity.cpp:126` |
| Vitesse linéaire max | `0` (illimitée) par défaut, fixée par script — ex. `0.2` en arena | `entity.cpp`, `level_arena_server.lua` |

### 2.3 Mode fermé (roulé / chute)

| Paramètre | Valeur | Effet |
|---|---|---|
| `CloseAccelCoef` | `1.0` | multiplie l'accélération d'entrée : `currentAccel = CloseAccelCoef * Accel` (`entity.cpp:445`) |
| `ModuleAccel` | `0.0001` | accélération conférée par un module au contact |
| `ModuleFriction` | `5` | friction conférée par un module |
| `AngularDecreasing` | `0.9` | amortissement angulaire |
| Friction | `friction = 1 - (Friction / 1000)` appliqué à la vitesse **angulaire** | `physics.cpp:466-472` |
| Friction dans l'eau | `Friction = 999` → quasi-arrêt | `physics.cpp:340` |

### 2.4 Mode ouvert (planeur) — le cœur du jeu

Calculé analytiquement dans `physics.cpp:414-441`, à partir de `Force.z`
(l'assiette, 0..1) et `Force.x` (le cap, en radians) :

```
angle  = (Force.z - 0.5) * π
hspeed = angle < 0 ? OpenAccelCoef
                   : max(OpenAccelCoef * cos(angle), OpenMinHSpeed)

minAngle = 1 - OpenMinAngleToZSpeed
zspeed = Force.z < minAngle
         ? -OpenZSpeed * sin(((minAngle - Force.z) / minAngle) * π/2)
         : 0
zspeed = min(zspeed, -OpenMinZSpeed)        // toujours descendant

force = ( cos(Force.x) * hspeed,
          sin(Force.x) * hspeed,
          zspeed )
```

Puis **`linearVel` et `angularVel` sont forcées à zéro** (`physics.cpp:439-440`).

| Paramètre | Valeur |
|---|---|
| `OpenAccelCoef` | `0.12` |
| `OpenZSpeed` | `0.1` |
| `OpenMinHSpeed` | `0.02` |
| `OpenMinZSpeed` | `0.005` |
| `OpenMinAngleToZSpeed` | `0.01` |

> **Point d'attention n° 2.** Le vol plané n'est pas de la physique : c'est un
> mouvement scripté qui écrase l'état du solide à chaque sous-pas. Une
> réimplémentation « propre » avec de vraies forces aérodynamiques donnera un
> jeu **différent**. Il faut porter cette formule telle quelle, quitte à
> l'améliorer ensuite consciemment.

### 2.5 Rebonds

Trois familles de contact, résolues dans `nearCallback` (`physics.cpp:117-360`) :

| Contact | Mode ODE | Valeurs |
|---|---|---|
| Joueur ↔ **eau** | `dContactBounce`, `mu = ∞`, `mu2 = 0` | `BounceWater = 0.0`, `BounceVelWater = 0.0` |
| Joueur ↔ **joueur** | `dContactBounce` si le niveau l'active, sinon `mu = mu2 = ∞` | `BounceClient = 1.0`, `BounceVelClient = 0.0` |
| Joueur ↔ **décor** | selon le module : `dContactBounce` si `module.bounce()`, sinon `mu = mu2 = ∞` | `BounceScene = 0.2`, `BounceVelScene = 0.1`, surchargés par module |

Un module avec `collide() == false` n'engendre aucun joint : il est seulement
enregistré dans `entity->collideModules` pour que le Lua puisse réagir
(`physics.cpp:250-258`). C'est le mécanisme des **zones-déclencheurs**.

### 2.6 Fin de mouvement

| Paramètre | Valeur | Rôle |
|---|---|---|
| `MinVelBeforeEnd` | `0.03` | en dessous, le joueur est considéré immobilisé (`running_session_state.cpp:76`) |
| `ArrivalTime` | mesuré | temps jusqu'à immobilisation, envoyé au client (`TimeArrival`) |

---

## 3. Règles de session

| Paramètre | Valeur | Unité |
|---|---|---|
| `NbMaxClients` | `16` | joueurs |
| `NbWaitingClients` | `1` | joueurs minimum pour lancer |
| `ForcedClientCount` | `5` | bots ajoutés pour compléter |
| `TimeBeforeStart` | `5000` | ms |
| `TimeBeforeRestart` | `5000` | ms |
| `TimeBeforeCheck` | `10000` | ms |
| `TimeTimeout` | `60000` | ms — durée max d'une session (désactivé dans le code, `variables.h:60`) |
| `WaitingReadyTimeout` | `20` | s |
| `DefaultMaxOpenClose` | `2` | ouvertures autorisées par session |
| `MaxAfkSessionCount` | `3` | sessions AFK avant éjection |

Machine à états : `WaitingClients → WaitingReady → WaitingStart → Running → Ending → …`

### Règles de score

- Le score de session est **remis à zéro à chaque `Entity:preUpdate()`** dans la
  plupart des scripts, puis re-attribué au contact d'un module scoré. Le score
  final est donc celui du **dernier contact valide**, pas un cumul.
- **Toucher le décor en mode ouvert**, si `EnableCrashInFly`, gèle l'entité,
  met `CurrentScore = 0`, consomme toutes les ouvertures et sort du jeu
  (`physics.cpp:305-322`).
- **Tomber à l'eau** annule le score dans les modes classiques ; certains
  niveaux (arena) téléportent au point de départ à la place
  (`level_arena_server.lua`).
- Un joueur immobilisé sur une cible peut être **éjecté par un autre joueur** —
  le score n'est acquis qu'à la fin de session.

### Bots

| Paramètre | Valeur |
|---|---|
| `BotAccuracyOpen` | `0.0002` |
| `BotAccuracyClose` | `0.0005` |
| Noms | `bill, richard, xobik, linus, moumou, starfox, alice, bob, biche, astorm, ringo, menew, chikant, gijoe, klavich` |

Le README affirme que « plus les humains jouent bien, mieux les bots jouent » :
la précision des bots est indexée sur le niveau des joueurs présents
(`server/src/bot.cpp`).

---

## 4. Constantes réseau

| Paramètre | Valeur | Source |
|---|---|---|
| Cadence effective des snapshots | **40 ms (25 Hz)** | `network.cpp:141` |
| `MT_NETWORK_UPDATE_FREQUENCE` | `10` Hz | `common/constant.h:25` |
| `MT_NETWORK_MY_UPDATE_FREQUENCE_RATIO` | `2` → 20 Hz | `common/constant.h:29` |
| Période full update | `10 × 20 = 200` ticks (~8 s) | `common/constant.h:35` |
| `MinDeltaToSendFullUpdate` | `3` | config — au-delà, full update immédiat |
| Quantification de position | flottant maison **10 bits** (4 exp + 6 mantisse) × 3 axes = 30 bits dans un `uint32` | `custom_floating_point.cpp`, `network.cpp:352-364` |
| LCT client (retard d'interpolation) | **150 ms à 600 ms**, auto-ajusté | `interpolator.cpp:114-115` |
| Transport | TCP | `NLNET::CBufServer` |
| Port | `51574` | config |
| `NetworkVersion` | `5` | config |
| Types de messages | 24 | `common/net_message.h:56-81` |

---

## 5. Contrôles de référence

| Action | Touche | Sémantique |
|---|---|---|
| Diriger | ← / → | modifie `Force.x` (cap, radians) |
| Accélérer / assiette | ↑ / ↓ | modifie `Force.z` (assiette, 0..1) en mode ouvert ; accélération en mode fermé |
| Ouvrir / fermer la boule | Ctrl droit | `swapOpenClose()` → message `OpenClose` |
| Caméra | souris | orbite locale, aucun effet serveur |
| Observer joueur suivant / précédent / soi | F9 / F10 / F11 | client uniquement |
| Tableau des scores | Tab | |
| Capture d'écran | F2 | |
| Fin / reset de session | F5 / F6 | **admin uniquement** |
| Boîtes de collision | F1 | debug |

---

## 6. Trajectoires de référence — à compléter

Ces cas de test sont la **spécification exécutable** du *feel*. Ils doivent être
implémentés dans `tools/feel-harness` et échouer tant que `mtpt-sim` n'existe
pas.

| # | Niveau | Scénario | Attendu |
|---|---|---|---|
| T1 | `classic_flat` | départ, aucun input, chute libre | temps d'impact à l'eau, position |
| T2 | `classic_flat` | départ, accélération max jusqu'au bout de la rampe | vitesse et position au décollage |
| T3 | `classic` | décollage puis ouverture immédiate, assiette neutre | portée horizontale, altitude à l'impact |
| T4 | `classic` | décollage, ouverture, assiette max (`Force.z = 1`) | portée maximale théorique |
| T5 | `classic` | décollage, ouverture, assiette min (`Force.z = 0`) | descente la plus rapide |
| T6 | `classic` | ouverture puis contact décor en vol | `CurrentScore == 0`, entité gelée |
| T7 | `arena` | chute à l'eau | téléportation au point de départ |
| T8 | tous | 2 joueurs en collision à vitesse égale | conservation, `BounceClient = 1.0` |

> Les valeurs attendues restent à mesurer. Deux méthodes possibles :
> a) instrumenter une reconstruction du serveur d'origine (coûteux, cf. §2 du
> document 01) ; b) les dériver analytiquement des formules ci-dessus pour les
> phases sans collision (T1-T5 sont entièrement calculables à la main), et
> caler T6-T8 en playtest.
>
> **Recommandation : commencer par la voie (b).** Les cinq premiers cas ne
> dépendent que de la formule du §2.4 et de la gravité — ils sont vérifiables
> sans jamais faire tourner le jeu original.
