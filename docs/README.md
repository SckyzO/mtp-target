# Documentation — reprise de MTP Target

Ce dossier documente l'analyse du code original (2003-2005) et le plan de
réécriture en Rust.

| Document | Contenu |
|---|---|
| [01 — Analyse de l'existant](01-analyse-existant.md) | Inventaire, architecture, netcode, physique, assets, dette technique. Ce qui a de la valeur et ce qu'il faut jeter. |
| [02 — Plan de réécriture Rust](02-plan-reecriture-rust.md) | Choix technologiques, architecture cible, netcode, contraintes mobile (Android/iOS), feuille de route en 8 phases, décisions à trancher. |
| [03 — Référence de gameplay](03-reference-gameplay.md) | Toutes les constantes physiques, règles de session, formules de vol, constantes réseau et cas de test de non-régression du *feel*. |

## Résumé en trois phrases

Le dépôt n'est pas une base de code à refactoriser : ses dépendances (NeL, ODE
0.5, STLport, FMOD) ne sont plus buildables, et il n'y a aucun test dans
50 000 lignes. En revanche c'est une **excellente spécification** : les 24
niveaux et les 21 scripts de règles sont en Lua, le netcode est raisonné et
documenté, et le jeu est petit (36 maillages, une mécanique).

La réécriture consiste donc à écrire un jeu neuf en se servant de celui-ci
comme document de conception — avec trois risques à lever en priorité : le
format binaire `.shape`, la licence GPL face à l'App Store, et la fidélité de
la sensation de jeu.
