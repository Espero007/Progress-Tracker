# Progress Tracker

> Application personnelle de suivi d'habitudes et d'objectifs récurrents, fondée sur un historique d'événements plutôt que sur de simples compteurs.

![PHP](https://img.shields.io/badge/PHP-8%2B-777BB4)
![MySQL](https://img.shields.io/badge/MySQL-8-4479A1)
![Statut](https://img.shields.io/badge/statut-en%20développement-yellow)

---

## Présentation

**Progress Tracker** permet de créer des tâches récurrentes (faire du sport, lire, boire suffisamment d'eau, méditer, etc.) et de les valider au fil du temps pour suivre sa régularité. Contrairement à une simple application de "checklist", Progress Tracker ne stocke **jamais** un compteur directement : chaque action de l'utilisateur (création d'une tâche, validation, interruption volontaire d'une série, archivage...) est enregistrée sous forme d'un **événement immuable**, et toutes les statistiques affichées — séries en cours, meilleure série, taux de régularité — sont **recalculées automatiquement** à partir de cet historique.

Ce choix, détaillé dans la documentation technique, garantit qu'aucune incohérence n'est possible entre les statistiques affichées et l'historique réel de l'utilisateur.

Le projet est développé en PHP 8 / MySQL 8, selon une architecture en couches strictement séparées, conçue dès le départ pour évoluer vers une API REST, une Progressive Web App, puis une application mobile Flutter — sans jamais dupliquer la logique métier.

---

## Sommaire

- [Fonctionnalités](#fonctionnalités)
- [Philosophie du projet](#philosophie-du-projet)
- [Stack technique](#stack-technique)
- [Architecture](#architecture)
- [Documentation](#documentation)
- [Démarrage rapide](#démarrage-rapide)
- [Structure du dépôt](#structure-du-dépôt)
- [Feuille de route](#feuille-de-route)
- [Export et sauvegarde des données](#export-et-sauvegarde-des-données)
- [Licence](#licence)

---

## Fonctionnalités

- Création de tâches récurrentes, avec objectif quotidien configurable (une ou plusieurs fois par jour) et jours actifs personnalisés.
- Validation quotidienne avec horodatage précis (date et heure).
- Calcul automatique des séries de progression (*streaks*), sans jamais stocker de compteur directement.
- Interruption volontaire d'une série (*Reset*) sans jamais perdre l'historique des séries précédentes.
- Archivage et restauration d'une tâche, sans perte de données.
- Export et import complets des données via un format ouvert et documenté (`.ptracker`).
- Architecture pensée dès le départ pour une future API REST et une application mobile Flutter.

---

## Philosophie du projet

Trois principes gouvernent toutes les décisions de conception du projet :

1. **L'historique est la seule source de vérité.** Aucune donnée n'est stockée si elle peut être recalculée à partir des événements passés.
2. **La logique métier est indépendante de tout détail technique.** Elle ne connaît ni HTTP, ni PDO, ni le protocole utilisé pour y accéder — ce qui permettra à une future API et une future application mobile de la réutiliser telle quelle, sans duplication.
3. **Chaque décision de conception est documentée et justifiée**, pas seulement implémentée — ce dépôt contient autant de documentation technique que de code.

---

## Stack technique

| Composant          | Choix                                             |
| ------------------ | ------------------------------------------------- |
| Langage backend    | PHP 8+                                            |
| Base de données    | MySQL 8 (InnoDB, `utf8mb4`)                       |
| Accès aux données  | PDO, requêtes préparées exclusivement             |
| Frontend (V1)      | HTML5, CSS3, JavaScript, Bootstrap (optionnel)    |
| Frontend (à venir) | Progressive Web App, puis application Flutter     |
| Format d'échange   | `.ptracker` (archive ZIP : `manifest.json` + CSV) |
| Style de code      | PSR-12, autoloading PSR-4                         |

---

## Architecture

Le projet suit une architecture en couches strictement séparées :

```
Interface
    ↓
Controllers
    ↓
Services       ← cœur de l'application, logique métier pure
    ↓
Repositories
    ↓
Database
    ↓
MySQL
```

Chaque couche ne connaît que la couche immédiatement inférieure. Les Controllers ne contiennent jamais de logique métier ; les Services ne dépendent jamais de HTTP ni de PDO. Cette règle, plus que le schéma lui-même, est ce qui permettra à une future API REST et à une future application Flutter de réutiliser l'intégralité de la logique métier sans en réécrire une seule ligne.

Le détail complet — rôle de chaque dossier, de chaque classe, scénarios pas à pas — est documenté dans [`docs/README_ARCHITECTURE.md`](docs/README_ARCHITECTURE.md).

---

## Documentation

Toute la conception du projet est documentée en détail dans le dossier [`docs/`](docs). C'est la référence à consulter avant toute contribution ou évolution du projet.

| Document                                                       | Contenu                                                                                 |
| -------------------------------------------------------------- | --------------------------------------------------------------------------------------- |
| [`docs/CAHIER_DES_CHARGES.md`](docs/CAHIER_DES_CHARGES.md)     | Besoin fonctionnel, règles de gestion, modèle métier, critères d'acceptation            |
| [`docs/README_ARCHITECTURE.md`](docs/README_ARCHITECTURE.md)   | Organisation du code en couches, rôle de chaque composant, scénarios complets           |
| [`docs/README_DATABASE.md`](docs/README_DATABASE.md)           | Schéma relationnel MySQL complet, `CREATE TABLE`, requêtes types, index                 |
| [`docs/README_IMPORT_EXPORT.md`](docs/README_IMPORT_EXPORT.md) | Spécification officielle du format d'échange `.ptracker`                                |
| [`docs/README_API.md`](docs/README_API.md)                     | Documentation de référence de la future API REST                                        |
| [`docs/README_CONVENTIONS.md`](docs/README_CONVENTIONS.md)     | Charte des conventions du projet (nommage, langue, dates, identifiants, glossaire, ADR) |

> En cas de question sur une règle de nommage, de format ou de style, [`docs/README_CONVENTIONS.md`](docs/README_CONVENTIONS.md) fait référence. En cas de question sur le besoin fonctionnel, [`docs/CAHIER_DES_CHARGES.md`](docs/CAHIER_DES_CHARGES.md) fait référence.

---

## Démarrage rapide

### Prérequis

- PHP 8.0 ou supérieur
- MySQL 8
- Composer

### Installation

```bash
git clone <url-du-dépôt> progress-tracker
cd progress-tracker
composer install
cp .env.example .env
```

Renseigner ensuite les identifiants de connexion à la base de données dans `.env` :

```
DB_HOST=localhost
DB_NAME=progress_tracker
DB_USER=...
DB_PASSWORD=...
APP_DEBUG=true
```

### Initialisation de la base de données

```bash
php bin/migrate.php
```

Ce script applique toutes les migrations SQL du dossier `migrations/` dans l'ordre, en s'appuyant sur la table `schema_migrations` pour ne jamais rejouer une migration déjà appliquée (voir [`docs/README_DATABASE.md`, section 11](docs/README_DATABASE.md)).

### Lancement en local

```bash
php -S localhost:8000 -t public
```

L'application est alors accessible sur `http://localhost:8000`.

---

## Structure du dépôt

```
progress-tracker/
├── public/          # Points d'entrée HTTP (application web et, à terme, API)
├── src/              # Code source (Controllers, Services, Repositories, Domain...)
├── migrations/       # Scripts SQL de migration du schéma
├── bin/              # Scripts utilitaires (migration, génération de jeton API...)
├── tests/            # Tests unitaires et d'intégration
├── docs/             # Documentation technique complète (voir ci-dessus)
├── composer.json
└── .env.example
```

L'arborescence complète et détaillée du dossier `src/` est décrite dans [`docs/README_ARCHITECTURE.md`, section 3](docs/README_ARCHITECTURE.md).

---

## Feuille de route

- [x] Cahier des charges et documentation technique complète
- [ ] Application web (V1) — création et suivi de tâches, calcul des séries
- [ ] Export / import `.ptracker`
- [ ] API REST (V2)
- [ ] Progressive Web App
- [ ] Application mobile Flutter (V3)

Le détail de chaque étape est décrit dans [`docs/CAHIER_DES_CHARGES.md`, section 11](docs/CAHIER_DES_CHARGES.md).

---

## Export et sauvegarde des données

Progress Tracker permet à tout moment d'exporter l'intégralité de ses données (tâches et historique complet) dans un fichier `ProgressTrackerExport.ptracker`, une archive ouverte et documentée, réimportable sur n'importe quelle installation. Le format complet, ses règles de validation et sa stratégie de compatibilité entre versions sont décrits dans [`docs/README_IMPORT_EXPORT.md`](docs/README_IMPORT_EXPORT.md).

---

## Licence

Projet personnel — licence à définir.

---

*Pour toute question de conception, se référer en priorité à la documentation du dossier [`docs/`](docs).*
