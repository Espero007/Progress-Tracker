# CAHIER_DES_CHARGES.md — Progress Tracker

> **Application de suivi de progression personnelle**

**Version : 2.0** (révision consolidée)
**Statut : Cahier des charges fonctionnel et technique — document de référence**

---

## Note de révision (à lire en premier)

Ce document remplace et fusionne les deux versions initiales du cahier des charges (identiques), rédigées avant que l'architecture détaillée du projet ne soit figée. Entre-temps, cinq documents techniques ont été produits : `README_ARCHITECTURE.md`, `README_DATABASE.md`, `README_IMPORT_EXPORT.md`, `README_API.md` et `README_CONVENTIONS.md`. Cette révision **aligne le cahier des charges sur les décisions réellement prises** dans ces documents, plutôt que l'inverse — conformément à la logique de tout cahier des charges, qui doit rester la référence fonctionnelle cohérente avec ce qui a effectivement été conçu.

Trois écarts ont été identifiés entre la version initiale et l'architecture retenue ; ils sont corrigés ici et signalés explicitement à l'endroit où ils apparaissent dans le document :

| Écart identifié | Version initiale | Version retenue dans cette révision |
|---|---|---|
| Nombre de validations autorisées par jour | "Une tâche ne peut être validée qu'une seule fois par jour" (RG-01) | Une tâche peut être validée **plusieurs fois par jour**, jusqu'à son objectif quotidien (`daily_target`) — nécessaire pour des tâches comme "boire suffisamment d'eau" (section 5.3, section 6). |
| Nombre d'entités du modèle métier | Trois entités "égales" : Tâche, Série, Validation | Deux entités **stockées** (Tâche, Événement) et un concept **dérivé, jamais stocké** (Série) — conformément à la philosophie de calcul plutôt que de stockage (section 7). |
| Notion d'"objectif" | Un seul type d'objectif ("30 jours"), ambigu | Deux objectifs distincts et non ambigus : l'**objectif quotidien** (occurrences par jour) et l'**objectif de série** (durée cible d'une série, optionnel) — voir section 5.7. Le second nécessite une extension du schéma de `README_DATABASE.md`, documentée en section 10.4. |

Le vocabulaire employé dans ce document est celui du glossaire officiel du projet (`README_CONVENTIONS.md`, section 18) : *Tâche*, *Événement*, *Validation*, *Reset*, *Série*, *Progression*, *Archive*.

---

## Sommaire

- [1. Présentation du projet](#1-présentation-du-projet)
- [2. Contexte](#2-contexte)
- [3. Vision du projet](#3-vision-du-projet)
- [4. Objectifs](#4-objectifs)
- [5. Fonctionnalités](#5-fonctionnalités)
- [6. Règles de gestion](#6-règles-de-gestion)
- [7. Modèle métier](#7-modèle-métier)
- [8. Interfaces prévues](#8-interfaces-prévues)
- [9. Contraintes techniques](#9-contraintes-techniques)
- [10. Architecture globale](#10-architecture-globale)
- [11. Évolutivité vers une application mobile](#11-évolutivité-vers-une-application-mobile)
- [12. Évolutions futures](#12-évolutions-futures)
- [13. Critères d'acceptation](#13-critères-dacceptation)
- [14. Documents associés](#14-documents-associés)
- [15. Conclusion](#15-conclusion)

---

# 1. Présentation du projet

## Description

**Progress Tracker** est une application permettant de suivre des habitudes ou des objectifs personnels nécessitant une répétition régulière.

Chaque fois qu'une tâche est réalisée, l'utilisateur la valide en cliquant sur un bouton **Accompli**. L'application enregistre alors :

- la date et l'heure exactes de la validation ;
- la quantité associée, lorsque la tâche le nécessite (par exemple, le nombre de verres d'eau bus) ;
- la progression actuelle, recalculée automatiquement ;
- l'historique complet, jamais tronqué.

L'utilisateur peut également interrompre volontairement sa progression en cours (*Reset*) lorsqu'il manque un jour actif, sans jamais perdre les données des séries précédentes.

L'objectif est de fournir un véritable historique de progression, auditable et fiable, plutôt qu'un simple compteur qui pourrait diverger silencieusement de la réalité.

---

# 2. Contexte

Certaines habitudes demandent une répétition régulière :

- se laver matin et soir ;
- faire du sport ;
- lire ;
- travailler un cours ;
- boire suffisamment d'eau ;
- prier ;
- méditer ;
- apprendre une langue.

L'utilisateur souhaite mesurer sa régularité sur plusieurs semaines ou plusieurs mois. L'application devra notamment permettre de répondre à des questions telles que :

- Depuis combien de jours suis-je régulier sur cette tâche ?
- À quelle heure ai-je validé ma tâche aujourd'hui ?
- Combien de séries ai-je réalisées pour cette tâche ?
- Quelle est ma meilleure série ?
- Quelles périodes de régularité ai-je déjà accomplies ?

Toutes les données doivent rester consultables, indéfiniment, y compris pour des tâches abandonnées puis reprises, ou archivées.

---

# 3. Vision du projet

L'objectif n'est pas uniquement de créer une petite application web. Le projet est pensé comme une base solide, destinée à servir de référence pendant plusieurs années, et à évoluer progressivement vers un véritable outil personnel de suivi multi-plateforme.

L'application est conçue dès le départ autour des principes suivants :

- architecture propre, en couches, formalisée dans `README_ARCHITECTURE.md` ;
- séparation stricte des responsabilités entre Interface, Controllers, Services et Repositories ;
- logique métier totalement indépendante de l'interface, du protocole HTTP et de la base de données ;
- historique des événements comme source de vérité unique, jamais de compteur stocké tel quel ;
- forte évolutivité, garantie par des conventions de compatibilité explicites (`README_CONVENTIONS.md`, section 16) ;
- possibilité de développer plusieurs interfaces (Web, PWA, Mobile) sans jamais réécrire le cœur de l'application (`README_ARCHITECTURE.md`, section 16).

Le projet constitue également un exercice approfondi de conception logicielle et d'architecture orientée objet, à vocation autant pédagogique que pratique.

---

# 4. Objectifs

L'application doit permettre de :

- créer plusieurs tâches, chacune configurée indépendamment (objectif quotidien, jours actifs) ;
- suivre leur progression au quotidien ;
- enregistrer chaque validation, avec sa date, son heure et sa quantité éventuelle ;
- conserver l'historique complet, immuable, de tous les événements survenus sur une tâche ;
- calculer automatiquement les séries de progression, sans jamais les stocker directement ;
- interrompre volontairement une série (*Reset*) sans supprimer les anciennes ;
- gérer plusieurs tâches simultanément, actives ou archivées ;
- exporter et importer l'intégralité des données dans un format ouvert et documenté ;
- préparer le projet à une évolution vers une API REST, une Progressive Web App, puis une application mobile Flutter, sans duplication de la logique métier.

---

# 5. Fonctionnalités

## 5.1. Création d'une tâche

Chaque tâche possède :

- un **nom** (obligatoire) ;
- une **description** (optionnelle) ;
- un **objectif quotidien** (`daily_target`) : le nombre de fois où la tâche doit être accomplie dans une même journée active pour que cette journée soit considérée comme complète (`1` par défaut ; un nombre plus élevé pour des tâches comme "boire de l'eau", section 6) ;
- des **jours actifs** (`target_days`) : les jours de la semaine où la tâche doit être suivie (par exemple, du lundi au vendredi pour une tâche liée au travail) ;
- un **objectif de série** (optionnel, voir section 5.7) : une durée cible, en jours, que l'utilisateur souhaite atteindre (par exemple, un défi de 30 jours).

Exemple :

```
Nom               Faire du sport
Description       30 minutes minimum.
Objectif quotidien   1 fois par jour
Jours actifs         Lun · Mer · Ven
Objectif de série    30 jours (optionnel)
```

Exemple d'une tâche à occurrences multiples :

```
Nom               Boire de l'eau
Description       Objectif journalier d'hydratation
Objectif quotidien   8 fois par jour
Jours actifs         Tous les jours
```

## 5.2. Tableau de bord

Chaque tâche active affiche :

- son nom ;
- la série de progression actuelle (en jours) ;
- l'objectif de série, s'il est défini, sous forme de barre de progression (section 5.7) ;
- la dernière validation (date et heure) ;
- un bouton **Accompli** ;
- un bouton **Reset**.

Exemple :

```
SPORT

18 jours

Objectif de série
18 / 30 jours

Dernière validation
Aujourd'hui, 08h15

[ Accompli ]     [ Reset ]
```

Pour une tâche à occurrences multiples, le tableau de bord affiche également la progression du jour :

```
BOIRE DE L'EAU

Aujourd'hui : 5 / 8 verres

Série actuelle : 12 jours

[ Accompli (+1) ]     [ Reset ]
```

## 5.3. Validation

Lorsqu'une tâche est validée :

- une validation est enregistrée sous la forme d'un **événement** (`CHECK`, `README_CONVENTIONS.md` section 9) ;
- la date et l'heure exactes sont enregistrées (horodatage complet, pas seulement le jour) ;
- une **quantité** est associée à la validation (`1` par défaut ; une valeur supérieure pour les tâches à occurrences multiples) ;
- **plusieurs validations peuvent être enregistrées le même jour**, jusqu'à ce que la somme des quantités du jour atteigne l'objectif quotidien de la tâche ;
- une journée est considérée comme **complète** dès que cette somme atteint (ou dépasse) l'objectif quotidien — c'est cette notion de journée complète qui alimente le calcul de la série de progression (section 7.2) ;
- la progression est recalculée automatiquement après chaque validation, jamais stockée directement (section 7).

> **Écart corrigé par rapport à la version initiale du cahier des charges :** la règle "une seule validation par jour" (ancienne RG-01) ne correspond qu'au cas particulier d'une tâche dont l'objectif quotidien vaut `1`. La règle générale, retenue ici, est la notion de journée complète, qui englobe ce cas particulier sans le contredire.

## 5.4. Historique

Toutes les validations sont conservées, sans limite de durée ni de volume.

| Date | Heure | Quantité |
|---|---|---|
| 01/04 | 08h15 | 1 |
| 02/04 | 08h12 | 1 |
| 03/04 | 08h08 | 1 |

Pour une tâche à occurrences multiples :

| Date | Heure | Quantité |
|---|---|---|
| 05/04 | 07h30 | 2 |
| 05/04 | 12h15 | 3 |
| 05/04 | 19h00 | 3 |

## 5.5. Reset

Le Reset :

- interrompt volontairement la série de progression en cours ;
- ne supprime **aucune** donnée historique (les validations passées restent visibles) ;
- démarre implicitement une nouvelle série, dont le premier jour sera la prochaine journée complète après le reset ;
- conserve toutes les anciennes séries, consultables indéfiniment (section 5.6).

## 5.6. Historique des séries

Chaque tâche peut afficher l'historique de ses séries passées, reconstitué à partir des événements `RESET` de son historique (section 7.2) :

```
Série 1
01 Avril → 11 Avril
11 jours

Série 2
13 Avril → Aujourd'hui
8 jours (en cours)
```

## 5.7. Objectif de série (optionnel)

Distinct de l'objectif quotidien (section 5.1), l'**objectif de série** est une durée cible optionnelle, exprimée en jours, que l'utilisateur peut se fixer pour une tâche (par exemple, un défi personnel de 30 jours consécutifs). Il est purement indicatif : il n'affecte ni la validation, ni le calcul de la série elle-même, seulement son affichage.

```
18 / 30 jours
```

Une barre de progression est affichée lorsque cet objectif est défini.

> **Point de cohérence à traiter :** cette notion n'existe pas encore dans le schéma de données décrit par `README_DATABASE.md` (qui ne connaît que l'objectif quotidien, `daily_target`). Son intégration est documentée en section 10.4 de ce document, à la manière des extensions déjà actées dans `README_CONVENTIONS.md` (sections 10 et 11) : elle doit être ajoutée à `README_DATABASE.md`, `README_ARCHITECTURE.md` (DTO de tâche) et `README_API.md` avant d'être implémentée.

---

# 6. Règles de gestion

## RG-01 — Journée complète

Une journée est considérée comme complète pour une tâche lorsque la somme des quantités validées ce jour-là atteint l'objectif quotidien (`daily_target`) de la tâche. *(Révise l'ancienne règle "une seule validation par jour" — voir Note de révision.)*

## RG-02 — Horodatage complet

Chaque validation possède une date et une heure précises (horodatage complet), pas seulement un jour.

## RG-03 — Immuabilité des validations

Les validations ne sont jamais supprimées ni modifiées, conformément à l'immuabilité de la table `events` (`README_DATABASE.md`, section 3.5).

## RG-04 — Préservation des données lors d'un Reset

Le Reset ne supprime jamais les données existantes ; il ajoute un nouvel événement `RESET` à l'historique, sans effacer aucun événement antérieur.

## RG-05 — Clôture de série

Chaque Reset clôt la série en cours et ouvre implicitement la possibilité d'une nouvelle série, dès la prochaine journée complète.

## RG-06 — Pluralité des séries

Une tâche peut posséder plusieurs séries au cours de son existence, toutes consultables indéfiniment.

## RG-07 — Série active uniquement affichée par défaut

Le compteur de progression affiché en priorité sur le tableau de bord correspond uniquement à la série active (la plus récente, non close par un `RESET`).

## RG-08 — Calcul automatique

Le nombre de jours d'une série est toujours calculé automatiquement à partir des événements, jamais saisi ni stocké manuellement (`README_DATABASE.md`, section 10.6).

## RG-09 — Jours actifs

Une tâche n'est évaluée (et ne peut être validée comme "manquée") que sur ses jours actifs (`target_days`) ; un jour non actif n'interrompt jamais une série en cours.

## RG-10 — Archivage réversible

Une tâche archivée conserve l'intégralité de son historique et peut être restaurée à tout moment sans perte de données (`ARCHIVE_TASK` / `RESTORE_TASK`).

---

# 7. Modèle métier

> **Écart corrigé par rapport à la version initiale :** le modèle métier repose sur **deux entités stockées**, pas trois. La "Série", présentée initialement comme une entité à part entière, est en réalité un **concept dérivé**, jamais stocké — conformément au principe "calcul plutôt que stockage" déjà retenu pour tout le projet (`README_DATABASE.md`, section 1.2 ; `README_CONVENTIONS.md`, section 5.4).

## 7.1. Tâche (entité stockée)

Représente une habitude suivie par l'utilisateur.

Exemples : Lecture, Sport, Douche, Prière, Hydratation.

Une tâche possède une configuration courante (nom, description, objectif quotidien, jours actifs, statut) et un historique complet d'événements qui lui sont rattachés.

## 7.2. Événement (entité stockée)

Représente un fait immuable et horodaté concernant une tâche : sa création, une validation, une interruption de série, un archivage, une restauration, une modification de configuration. C'est la **source de vérité unique** du système (section 10.2).

## 7.3. Série (concept dérivé, jamais stocké)

Une série représente une période continue de journées complètes pour une tâche, délimitée par des événements `RESET` (ou par le début de l'historique de la tâche).

```
01 Avril  →  15 Avril
15 jours
```

Une tâche possède, à tout instant :

- une série active, calculée à partir des événements les plus récents (depuis le dernier `RESET`, ou depuis la création de la tâche) ;
- plusieurs séries passées, chacune reconstituable à la demande à partir de l'historique complet, sans qu'aucune d'elles ne soit jamais stockée en tant que telle.

## 7.4. Validation (sous-type d'événement)

Une validation représente un clic sur **Accompli**. Techniquement, c'est un événement de type `CHECK` (`README_CONVENTIONS.md`, section 9), portant :

- la date et l'heure exactes (horodatage complet) ;
- une quantité (1 par défaut, ou davantage pour une tâche à occurrences multiples) ;
- une note optionnelle.

---

# 8. Interfaces prévues

## 8.1. Tableau de bord

```
Mes tâches

---------------------------------
SPORT
18 jours
[ Accompli ]   [ Reset ]
---------------------------------
LECTURE
6 jours
[ Accompli ]   [ Reset ]
---------------------------------
BOIRE DE L'EAU
Aujourd'hui : 5 / 8
[ Accompli (+1) ]   [ Reset ]
```

## 8.2. Historique d'une tâche

```
SPORT

Série actuelle
18 jours

Historique des séries

Série 1 — 11 jours
01 Avril → 11 Avril
-----------------
Série 2 — 18 jours (en cours)
13 Avril → Aujourd'hui
```

## 8.3. Création d'une tâche

```
Nom
Description
Objectif quotidien
Jours actifs
Objectif de série (optionnel)

[ Enregistrer ]
```

---

# 9. Contraintes techniques

## Technologies

| Composant | Choix retenu |
|---|---|
| Backend | PHP 8+ (`README_ARCHITECTURE.md`, `README_CONVENTIONS.md` section 13) |
| Base de données | MySQL 8, moteur InnoDB, encodage `utf8mb4` (`README_DATABASE.md`) |
| Accès aux données | PDO exclusivement, requêtes préparées systématiques |
| Frontend (V1) | HTML5, CSS3, JavaScript, Bootstrap (optionnel) |
| Frontend (V2) | Progressive Web App, consommant l'API REST (`README_API.md`) |
| Frontend (V3) | Application Flutter (Android, iOS, et potentiellement Windows/Linux/macOS/Web) |
| Format d'échange | `.ptracker` (archive ZIP contenant `manifest.json`, `tasks.csv`, `events.csv`) — `README_IMPORT_EXPORT.md` |
| Identifiants | UUID v4 publics, clé interne `BIGINT` non exposée (`README_DATABASE.md`, section 3.1) |
| Style de code | PSR-12, autoloading PSR-4, typage strict (`README_CONVENTIONS.md`, section 13) |

Le détail complet de chacun de ces choix, avec sa justification et ses alternatives écartées, figure dans les documents dédiés référencés en section 14.

---

# 10. Architecture globale

## 10.1. Philosophie

Le projet est conçu selon une architecture en couches, détaillée intégralement dans `README_ARCHITECTURE.md` :

```
Interface
    │
    ▼
Controllers
    │
    ▼
Services
    │
    ▼
Repositories
    │
    ▼
Database
    │
    ▼
MySQL
```

Chaque couche possède une responsabilité unique. La logique métier reste strictement indépendante de la base de données, de l'interface graphique et des appels HTTP — une règle de dépendance stricte (`README_ARCHITECTURE.md`, section 1.2), pas une simple convention de style.

## 10.2. Source de vérité

L'application n'enregistre jamais directement un compteur. Elle enregistre des **événements métier** :

- création d'une tâche ;
- validation ;
- reset ;
- archivage / restauration ;
- modification de configuration.

Le compteur — et plus largement toute statistique ou série de progression — est **toujours calculé automatiquement** à partir de ces événements, jamais stocké tel quel (`README_DATABASE.md`, section 10.6).

Cette approche présente plusieurs avantages, déjà détaillés dans `README_DATABASE.md` (section 1.2) :

- aucune incohérence possible entre un compteur et son historique ;
- possibilité de recalculer intégralement les statistiques si les règles de calcul évoluent, sans migration de données ;
- ajout facile de nouvelles règles métier ;
- meilleure évolutivité et auditabilité.

L'historique constitue donc la **source de vérité** unique du système.

## 10.3. Modèle de données (résumé)

Le schéma complet, avec justification de chaque choix, figure dans `README_DATABASE.md`. Il repose sur trois tables principales :

| Table | Rôle |
|---|---|
| `tasks` | État courant de chaque tâche (projection mutable, toujours reconstructible depuis `events`) |
| `task_target_days` | Jours actifs d'une tâche (table de normalisation) |
| `events` | Journal complet et immuable de tous les événements — la source de vérité |

## 10.4. Extension actée : objectif de série

Conformément à la section 5.7, l'objectif de série (durée cible optionnelle, en jours) nécessite l'ajout d'une colonne à la table `tasks` :

```sql
ALTER TABLE tasks
    ADD COLUMN series_goal_days SMALLINT UNSIGNED NULL AFTER daily_target;
```

Cette colonne est **nullable**, sans valeur par défaut contraignante : son absence de valeur signifie simplement qu'aucun objectif de série n'a été fixé pour la tâche. Il s'agit d'une évolution strictement additive, sans impact sur les tâches existantes (cohérent avec `README_CONVENTIONS.md`, section 16.2). Cette extension doit être répercutée, avant implémentation, dans :

- `README_DATABASE.md` (colonne `series_goal_days`, section 4) ;
- `README_ARCHITECTURE.md` (propriété `seriesGoalDays` sur `Domain\Task` et sur les DTO associés) ;
- `README_API.md` (champ optionnel `series_goal_days` dans les requêtes/réponses de `/tasks`) ;
- `README_IMPORT_EXPORT.md` (colonne optionnelle ajoutée en fin de `tasks.csv`, évolution `MINOR` du format).

---

# 11. Évolutivité vers une application mobile

L'une des exigences majeures du projet est de permettre le développement futur d'une application mobile sans devoir réécrire le backend. Le projet est conçu dès le départ autour d'une architecture compatible avec une API REST (`README_API.md`).

## Étape 1 — Interface Web (actuelle)

```
Navigateur
    ↓
Controllers
    ↓
Services
    ↓
Repositories
    ↓
MySQL
```

## Étape 2 — API REST (spécifiée dans `README_API.md`)

```
Application Web            API REST
        │                      │
        └──────────┬───────────┘
                   ▼
               Services
                   ▼
             Repositories
                   ▼
                 MySQL
```

Les Services constituent le cœur de l'application (`README_ARCHITECTURE.md`, section 2.1). Les Controllers Web et les Controllers API ne font que les appeler — aucune règle métier n'est jamais dupliquée entre les deux.

## Étape 3 — Application Flutter

```
Flutter
    ↓ HTTP
API REST
    ↓
Services
    ↓
Repositories
    ↓
MySQL
```

Ainsi, aucune logique métier n'est dupliquée, les règles restent centralisées dans les Services, et le backend reste unique (`README_ARCHITECTURE.md`, section 16.3).

## Étape intermédiaire — Progressive Web App

Avant même le développement de l'application mobile, l'application web pourra évoluer vers une **Progressive Web App (PWA)**, permettant :

- une installation sur smartphone ;
- un lancement comme une application native, en plein écran ;
- un accès plus rapide ;
- une base pour d'éventuelles fonctionnalités hors-ligne et notifications futures.

Cette étape ne nécessite, comme l'étape 2, aucune modification de la logique métier existante (`README_ARCHITECTURE.md`, section 16.2).

## Choix technologiques

Le backend reste développé en PHP, pour sa simplicité de développement, la maîtrise complète qu'il permet sur l'architecture, la facilité d'hébergement, et la facilité de mise en place d'une API REST.

Pour la future application mobile, **Flutter** est retenu, permettant de cibler Android, iOS, et potentiellement Windows, Linux, macOS et Web à partir d'un code unique.

---

# 12. Évolutions futures

Certaines évolutions listées ici disposent déjà d'un chemin d'intégration documenté (`README_CONVENTIONS.md`, section 19) ; d'autres restent des pistes non détaillées à ce stade.

## 12.1. Déjà anticipées dans la documentation technique

| Évolution | Document(s) concerné(s) |
|---|---|
| API REST | `README_API.md` (déjà spécifiée intégralement) |
| Progressive Web App | `README_ARCHITECTURE.md`, section 16.2 |
| Application Flutter | `README_ARCHITECTURE.md`, section 16.3 |
| Synchronisation multi-appareils | `README_IMPORT_EXPORT.md`, section 1.1 ; `README_CONVENTIONS.md`, section 10 (source d'un événement) |
| Multi-utilisateurs | `README_DATABASE.md`, section 12.5 ; `README_API.md`, section 14 |
| Export PDF / Excel | Naturellement compatible avec le principe de recalcul à la demande (section 10.2), sans modification du schéma |

## 12.2. Pistes non encore détaillées

- statistiques avancées et graphiques ;
- calendrier de suivi ;
- notifications et rappels ;
- catégories et priorités de tâches ;
- tâches hebdomadaires ou mensuelles (au-delà des jours actifs déjà pris en charge) ;
- badges et score de régularité ;
- API publique (au-delà de l'API privée destinée aux clients officiels) ;
- synchronisation Cloud.

Conformément à `README_CONVENTIONS.md` (section 16.3), toute évolution retenue devra être **documentée avant d'être implémentée**, dans le document technique concerné.

---

# 13. Critères d'acceptation

Le projet est considéré conforme à ce cahier des charges lorsque :

- plusieurs tâches peuvent être créées, chacune avec son propre objectif quotidien et ses propres jours actifs ;
- une tâche peut être validée un nombre de fois par jour cohérent avec son objectif quotidien (RG-01) ;
- les validations sont historisées de façon immuable, avec date et heure exactes (RG-02, RG-03) ;
- un Reset interrompt la série active sans jamais supprimer de données (RG-04, RG-05) ;
- les séries — actives et passées — sont toujours calculées automatiquement à partir de l'historique, jamais stockées (RG-06, RG-07, RG-08) ;
- une tâche peut être archivée puis restaurée sans perte de données (RG-10) ;
- l'intégralité des données peut être exportée puis réimportée via le format `.ptracker`, avec validation stricte à l'import (`README_IMPORT_EXPORT.md`) ;
- l'architecture en couches permet l'ajout d'une API REST sans modification de la logique métier existante ;
- une future application Flutter pourra consommer cette API sans duplication de règles métier ;
- toutes les conventions transversales (nommage, dates, identifiants) sont respectées de façon uniforme dans le code, la base de données, le format d'échange et l'API (`README_CONVENTIONS.md`).

---

# 14. Documents associés

Ce cahier des charges définit **le besoin fonctionnel**. Il s'appuie sur les documents techniques suivants, qui en détaillent la mise en œuvre et priment sur ce document pour toute question de conception ou de convention :

| Document | Contenu |
|---|---|
| `README_ARCHITECTURE.md` | Organisation du code en couches, rôle de chaque composant, scénarios complets |
| `README_DATABASE.md` | Schéma relationnel MySQL complet, justifications, requêtes types |
| `README_IMPORT_EXPORT.md` | Spécification du format d'échange `.ptracker` |
| `README_API.md` | Documentation officielle de l'API REST |
| `README_CONVENTIONS.md` | Charte des conventions transversales (nommage, langue, dates, identifiants, ADR, glossaire) |

En cas de divergence apparente entre ce cahier des charges et l'un de ces documents, la présente révision (version 2.0) fait foi sur le besoin fonctionnel ; les documents techniques font foi sur la mise en œuvre. Toute divergence constatée doit être corrigée dans les deux sens, jamais ignorée.

---

# 15. Conclusion

**Progress Tracker** est conçu comme un projet à la fois d'apprentissage et de production.

À court terme, il offre une interface web permettant de suivre efficacement des habitudes personnelles, avec un modèle de données fondé sur l'historique complet des événements plutôt que sur de simples compteurs.

À moyen terme, il pourra évoluer vers une **Progressive Web App**, offrant une expérience proche d'une application native sur smartphone, sans aucune duplication de la logique métier déjà écrite.

À long terme, son architecture permettra le développement d'une application **Flutter**, utilisant le même backend PHP via l'API REST déjà spécifiée, garantissant la réutilisation intégrale de la logique métier et la pérennité du projet.

L'objectif reste, in fine, de construire un logiciel robuste, modulaire et évolutif, capable d'accompagner de futures fonctionnalités sans jamais remettre en cause son architecture fondamentale — et documenté avec un niveau de rigueur permettant à ce cahier des charges, comme aux documents techniques qui l'accompagnent, de rester une référence fiable pendant toute la durée de vie du projet.

---

*Fin du document — `CAHIER_DES_CHARGES.md`, version 2.0, cohérent avec `README_ARCHITECTURE.md`, `README_DATABASE.md`, `README_IMPORT_EXPORT.md`, `README_API.md` et `README_CONVENTIONS.md`.*
