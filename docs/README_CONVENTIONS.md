# README_CONVENTIONS.md — Charte des conventions du projet

**Projet :** Progress Tracker
**Public visé :** tout développeur rejoignant le projet, à n'importe quel stade de son développement.
**Statut du document :** référence normative — en cas de doute ou de divergence apparente avec un autre document, c'est **ce document** qui fait foi sur les questions de convention (nommage, format, style), tandis que `README_ARCHITECTURE.md`, `README_DATABASE.md`, `README_IMPORT_EXPORT.md` et `README_API.md` restent la référence sur les questions de conception et de comportement fonctionnel.

Ce document ne redéfinit ni l'architecture, ni le schéma de données, ni les fonctionnalités : ces sujets sont respectivement traités par `README_ARCHITECTURE.md`, `README_DATABASE.md` et `CAHIER_DES_CHARGES.md`. Il centralise en revanche **toutes les règles transversales** — nommage, formats, langue, style de code — qui, sans ce document, resteraient éparpillées ou implicites, au risque d'être appliquées différemment d'un module à l'autre.

> **Note de cohérence :** la rédaction de ce document a mis en évidence deux concepts absents des documents précédents (la source d'un événement, section 10, et le statut `DELETED`, section 11). Ils sont introduits ici comme des **extensions actées mais non encore implémentées**, avec leur plan d'intégration explicite dans `README_DATABASE.md`, `README_IMPORT_EXPORT.md` et `README_ARCHITECTURE.md`. Ce choix illustre directement la règle de développement énoncée en section 16 : privilégier l'ajout à la modification, et documenter explicitement chaque évolution avant de l'implémenter.

---

## Sommaire

1. Présentation
2. Philosophie générale
3. Langue du projet
4. Conventions de nommage
5. Identifiants
6. Conventions de dates
7. Conventions CSV
8. Conventions JSON
9. Types d'événements
10. Sources d'événements
11. Statuts
12. Conventions SQL
13. Conventions PHP
14. Conventions API
15. Conventions d'import/export
16. Conventions de développement
17. Décisions d'architecture (ADR)
18. Glossaire
19. Évolutions futures

---

## 1. Présentation

### 1.1. Objectif de ce document

Ce document a un objectif unique : que **deux développeurs différents, travaillant sur deux parties différentes du projet à des moments différents, produisent un code et des données cohérents entre eux**, sans avoir besoin de se concerter à chaque décision de détail. Il centralise les conventions du projet — c'est-à-dire les règles qui n'ont pas de bonne ou de mauvaise réponse dans l'absolu, mais qui doivent être appliquées **de façon identique partout** pour que le projet reste compréhensible.

### 1.2. Pourquoi documenter des conventions plutôt que les laisser implicites

Une convention non écrite existe quand même — dans la tête de la personne qui l'a appliquée en premier. Le problème n'est pas qu'elle soit mauvaise, mais qu'elle soit **invisible** : rien n'empêche un autre développeur (ou le même, six mois plus tard) de faire un choix différent, tout aussi défendable, mais incompatible. Documenter une convention la rend :

- **vérifiable** : on peut relire ce document et confirmer qu'un code la respecte, plutôt que de se fier à la mémoire ;
- **discutable** : une convention écrite peut être révisée consciemment, alors qu'une convention implicite ne peut qu'être silencieusement contredite ;
- **transmissible** : un nouveau développeur n'a pas besoin de "deviner" les règles en lisant du code existant, avec le risque d'en déduire une règle incorrecte à partir d'un cas particulier.

### 1.3. Bénéfices attendus

| Bénéfice                   | Comment ce document y contribue                                                                                                                                                               |
| -------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Cohérence**              | Une même notion (une date, un identifiant, un statut) est toujours représentée de la même façon, quel que soit l'endroit du projet où elle apparaît.                                          |
| **Maintenabilité**         | Un développeur qui modifie une partie du code n'a pas à redécouvrir les règles implicites de cette partie : elles sont déjà écrites ici.                                                      |
| **Évolutivité**            | Les règles de compatibilité (section 16) sont explicites, ce qui évite qu'une évolution future ne casse silencieusement une convention établie.                                               |
| **Réduction des erreurs**  | Nombre de bugs proviennent d'incohérences de convention (ex. une date locale mélangée à des dates UTC). Ce document élimine cette classe entière de bugs par construction, pas par vigilance. |
| **Facilité d'intégration** | Un nouveau développeur dispose d'un point d'entrée unique pour toutes les questions de forme, plutôt que de devoir les déduire de la lecture de plusieurs fichiers de code.                   |

---

## 2. Philosophie générale

Cette section rappelle, sans les redémontrer, les principes fondateurs déjà établis dans les autres documents de référence — ils conditionnent directement les conventions détaillées dans le reste de ce document.

| Principe                              | Rappel                                                                                                                                                     | Document de référence                            |
| ------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------ |
| **Simplicité**                        | Toujours préférer la solution la plus simple qui répond au besoin réel, plutôt que la plus générique ou la plus "future-proof" par anticipation excessive. | `CAHIER_DES_CHARGES.md`                          |
| **Modularité**                        | Chaque composant a une responsabilité unique et clairement délimitée.                                                                                      | `README_ARCHITECTURE.md`                         |
| **Séparation des responsabilités**    | Interface, Controllers, Services, Repositories ne se mélangent jamais.                                                                                     | `README_ARCHITECTURE.md`, section 1              |
| **Architecture en couches**           | Une couche ne connaît que la couche immédiatement inférieure.                                                                                              | `README_ARCHITECTURE.md`, section 1.2            |
| **Logique métier indépendante**       | Les Services ne dépendent ni de HTTP, ni de PDO, ni d'aucun détail technique externe.                                                                      | `README_ARCHITECTURE.md`, section 5.2            |
| **Historique comme source de vérité** | La table `events` est la seule donnée qui compte réellement ; tout le reste en est dérivé.                                                                 | `README_DATABASE.md`, section 1.2                |
| **Calcul plutôt que stockage**        | Aucun compteur, aucune série, aucune statistique n'est jamais stockée telle quelle — toujours recalculée depuis `events`.                                  | `README_DATABASE.md`, section 10.6               |
| **Forte évolutivité**                 | Chaque décision de conception documente explicitement son chemin d'évolution future, pour ne jamais bloquer une extension anticipée.                       | Sections "Évolutions futures" de chaque document |

Ces principes ne sont pas de simples déclarations d'intention : chaque convention détaillée dans ce document en est une conséquence directe et vérifiable. Quand une convention semble arbitraire, elle découle presque toujours de l'un de ces principes — ce document prend soin de le rappeler à chaque fois.

---

## 3. Langue du projet

### 3.1. Répartition officielle

| Élément                                                                                          | Langue             | Justification                                                                                                                                                                                                                                                                                                                                |
| ------------------------------------------------------------------------------------------------ | ------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Code source (noms de classes, méthodes, variables, constantes)                                   | **Anglais**        | Convention quasi-universelle de l'écosystème PHP et des bibliothèques tierces (PSR, Composer) ; facilite la lecture par tout développeur, francophone ou non, qui rejoindrait le projet ; évite le mélange de langues au sein d'une même ligne de code (ex. `class TâcheService` mélangeant français et syntaxe anglaise serait incohérent). |
| Commentaires et PHPDoc dans le code                                                              | **Français**       | Le projet et son unique mainteneur actuel raisonnent en français ; la documentation de référence (ce document inclus) est intégralement en français ; garder les commentaires dans la même langue que la documentation évite un aller-retour mental inutile entre deux langues pour une même idée.                                           |
| Base de données (tables, colonnes)                                                               | **Anglais**        | Cohérent avec le code source qui la manipule directement (`README_DATABASE.md`) ; évite les problèmes d'accents dans les identifiants SQL (jamais totalement garantis selon les outils).                                                                                                                                                     |
| Fichiers CSV (`tasks.csv`, `events.csv`)                                                         | **Anglais**        | Cohérent avec les noms de colonnes SQL qu'ils reflètent (`README_IMPORT_EXPORT.md`) ; un format d'échange a vocation à être universellement lisible, y compris par un futur outil tiers non francophone.                                                                                                                                     |
| JSON (API, `manifest.json`)                                                                      | **Anglais** (clés) | Même raisonnement que le CSV : les clés JSON sont un contrat technique, pas un contenu destiné à un lecteur humain francophone.                                                                                                                                                                                                              |
| Documentation technique (les fichiers `README_*.md`)                                             | **Français**       | Choix assumé dès l'origine du projet ; la documentation s'adresse avant tout à son mainteneur francophone.                                                                                                                                                                                                                                   |
| Interface utilisateur (web, PWA, Flutter)                                                        | **Français**       | Public cible de l'application (utilisateur personnel francophone).                                                                                                                                                                                                                                                                           |
| Messages d'erreur affichés à l'utilisateur (`error.message` de l'API, `README_API.md` section 8) | **Français**       | Cohérent avec l'interface utilisateur : un message d'erreur est un contenu destiné à être lu et compris par l'utilisateur final.                                                                                                                                                                                                             |
| Codes d'erreur machine (`error.code`, ex. `VALIDATION_FAILED`)                                   | **Anglais**        | Ce sont des identifiants techniques, comparés par égalité dans le code client (`if (error.code === 'TASK_NOT_FOUND')`) — jamais affichés tels quels à l'utilisateur ; l'anglais est la convention universelle pour ce type d'identifiant stable.                                                                                             |

### 3.2. Règle de synthèse

**Tout ce qui est un contrat technique (code, schéma, format d'échange, identifiants machine) est en anglais. Tout ce qui est un contenu destiné à un lecteur humain du projet (documentation, interface, messages) est en français.** Cette règle unique permet de trancher n'importe quel cas non explicitement couvert par le tableau ci-dessus.

---

## 4. Conventions de nommage

### 4.1. Tableau de référence

| Élément                   | Convention                                                             | Exemple                                                  |
| ------------------------- | ---------------------------------------------------------------------- | -------------------------------------------------------- |
| Classes                   | `PascalCase`, nom singulier                                            | `TaskService`, `Event`, `TaskRepository`                 |
| Interfaces                | `PascalCase` + suffixe `Interface`                                     | `TaskRepositoryInterface`, `ClockInterface`              |
| Enums (PHP 8.1+)          | `PascalCase` pour le type, `UPPER_SNAKE_CASE` pour chaque cas          | `enum EventType { case CREATE_TASK; ... }`               |
| Traits                    | `PascalCase` + suffixe `Trait`                                         | `TimestampableTrait`                                     |
| Variables                 | `camelCase`                                                            | `$dailyTarget`, `$taskUuid`                              |
| Constantes (PHP `const`)  | `UPPER_SNAKE_CASE`                                                     | `const DEFAULT_DAILY_TARGET = 1;`                        |
| Méthodes et fonctions     | `camelCase`, verbe explicite                                           | `createTask()`, `computeCurrentStreak()`, `findByUuid()` |
| Tables SQL                | `snake_case`, nom **pluriel**                                          | `tasks`, `events`, `task_target_days`                    |
| Colonnes SQL              | `snake_case`                                                           | `daily_target`, `created_at`, `event_type`               |
| Routes API                | `snake_case` ou mots simples, ressource au **pluriel**                 | `/tasks`, `/tasks/{uuid}/checks`                         |
| Fichiers PHP              | `PascalCase.php`, identique au nom de la classe qu'il contient (PSR-4) | `TaskService.php`                                        |
| Dossiers du code source   | `PascalCase`                                                           | `Controllers/`, `Services/`, `Repositories/`             |
| En-têtes de colonnes CSV  | `snake_case`                                                           | `task_id`, `daily_target`, `occurred_at`                 |
| Clés JSON                 | `snake_case`                                                           | `"daily_target"`, `"target_days"`                        |
| Fichiers de migration SQL | `AAAAMMJJHHMMSS_description_en_snake_case.sql`                         | `20260115090000_create_tasks_table.sql`                  |

### 4.2. Cas particulier : pourquoi les méthodes commencent toujours par un verbe

Une méthode représente une action ou une question ; son nom doit le refléter sans ambiguïté :

- `find...()` : peut retourner une valeur absente (`null` ou tableau vide), ne lève jamais d'exception pour une absence de résultat ;
- `get...()` : retourne toujours une valeur, lève une exception si elle est absente (ex. si utilisé après une vérification d'existence déjà faite) ;
- `create...()` / `save...()` : opération d'écriture ;
- `compute...()` : opération de calcul pur, sans effet de bord, cohérent avec `ProgressCalculationService` (`README_ARCHITECTURE.md`, section 5.4) ;
- `validate...()` : lève une exception (`ValidationException`) en cas d'échec, ne retourne rien en cas de succès.

Cette convention de préfixe rend le comportement d'une méthode prévisible avant même d'en lire le corps.

---

## 5. Identifiants

### 5.1. Identifiants concernés

| Identifiant | Entité               | Statut                                                                          |
| ----------- | -------------------- | ------------------------------------------------------------------------------- |
| `task_id`   | Tâche                | Implémenté (`README_DATABASE.md`, `README_IMPORT_EXPORT.md`)                    |
| `event_id`  | Événement            | Implémenté                                                                      |
| `user_id`   | Utilisateur          | Réservé pour l'évolution multi-utilisateur (`README_DATABASE.md`, section 12.5) |
| `series_id` | Série de progression | **Volontairement absent** — voir section 5.4                                    |

### 5.2. Pourquoi des identifiants textuels stables (UUID) plutôt qu'auto-incrémentés

Ce choix a déjà été justifié en détail dans `README_DATABASE.md` (section 3.1) ; il est rappelé ici comme **convention transversale du projet**, applicable à toute future entité identifiable publiquement :

| Critère                             | Entier auto-incrémenté                                   | UUID v4 (retenu comme identifiant public)                            |
| ----------------------------------- | -------------------------------------------------------- | -------------------------------------------------------------------- |
| Stabilité entre installations       | Non — dépend de l'ordre d'insertion propre à chaque base | Oui — valable universellement                                        |
| Génération anticipée hors-ligne     | Impossible                                               | Possible (utile pour Flutter, `README_ARCHITECTURE.md` section 16.3) |
| Devinable / énumérable              | Oui (un attaquant peut itérer `1, 2, 3...`)              | Non                                                                  |
| Performance en clé primaire interne | Excellente                                               | Mauvaise (fragmentation InnoDB)                                      |

**Convention retenue, valable pour toute nouvelle entité du projet :** un identifiant **interne** (`BIGINT UNSIGNED AUTO_INCREMENT`), jamais exposé, couplé à un identifiant **public** (`UUID v4`, colonne `uuid`), seul exposé à l'extérieur (API, export). Cette convention n'est pas spécifique à `tasks` et `events` : elle s'applique par défaut à toute table future qui nécessiterait un identifiant public (par exemple, la future table `users`).

### 5.3. Format des UUID

- Version 4 (aléatoire), générée via `Helpers\UuidGenerator` (`README_ARCHITECTURE.md`, section 13) — jamais via une fonction ad hoc dupliquée dans une autre classe.
- Représentation textuelle canonique en minuscules, avec tirets : `b3f2c9a0-6d3e-4f1a-9e2b-8c7d5a1f0e4d`.
- Stockage SQL en `CHAR(36)` (voir `README_DATABASE.md` pour la justification de ce choix plutôt que `BINARY(16)`, retenu ici pour sa lisibilité directe en base, utile en phase de développement et de débogage).

### 5.4. Pourquoi il n'existe pas de `series_id` stocké

Le cahier des charges évoque la nécessité de "gérer plusieurs séries de progression" et de "conserver toutes les anciennes séries". Il serait tentant d'en déduire une table `series` avec un identifiant `series_id` dédié. **Ce choix est délibérément écarté**, car il contredirait directement le principe "calcul plutôt que stockage" (section 2) : une série de progression est un **segment dérivé** de la liste des événements d'une tâche, délimité par des événements `RESET` (ou par le début de l'historique). Elle n'a pas d'existence propre en base de données — elle est reconstituée à la demande par `ProgressCalculationService` (`README_ARCHITECTURE.md`, section 5.4).

Si un besoin futur nécessitait de référencer une série précise (par exemple, "partager le lien de ma série de 30 jours de janvier"), la convention retenue serait de dériver un identifiant **de façon déterministe** à partir de données déjà existantes (par exemple, l'`event_id` de l'événement qui a démarré la série), plutôt que d'introduire un nouveau compteur stocké. Cette piste est documentée ici à titre de convention par anticipation, non implémentée en V1.

---

## 6. Conventions de dates

### 6.1. Règle unique

**Toute date stockée ou échangée par le système est exprimée en UTC, au format ISO 8601.** Cette règle s'applique identiquement en base de données (`README_DATABASE.md`, section 3.2), dans le format d'export (`README_IMPORT_EXPORT.md`, section 8.2) et dans l'API (`README_API.md`, section 4.3). Il n'existe **aucune exception** à cette règle pour une donnée stockée ou transmise entre systèmes.

### 6.2. Représentations par contexte

| Contexte                                            | Format                                                                           | Exemple                                                             |
| --------------------------------------------------- | -------------------------------------------------------------------------------- | ------------------------------------------------------------------- |
| Colonne SQL (`DATETIME`)                            | `AAAA-MM-JJ HH:MM:SS`, toujours interprétée comme UTC par convention applicative | `2026-07-22 09:14:03`                                               |
| JSON (API, `manifest.json`)                         | ISO 8601 avec suffixe `Z`                                                        | `"2026-07-22T09:14:03Z"`                                            |
| CSV (`events.csv`, `tasks.csv`)                     | ISO 8601 avec suffixe `Z`                                                        | `2026-07-22T09:14:03Z`                                              |
| Affichage utilisateur (interface web, PWA, Flutter) | Format localisé, dans le fuseau horaire de l'utilisateur, **jamais** en UTC brut | `22 juillet 2026 à 11h14` (heure d'Afrique de l'Ouest, par exemple) |

### 6.3. Où s'effectue la conversion UTC ↔ heure locale

La conversion n'a lieu **qu'à la frontière de présentation** (interface utilisateur), jamais dans les Services ni dans les Repositories (`README_ARCHITECTURE.md`, section 5.2). Le champ `timezone` du manifeste d'export (`README_IMPORT_EXPORT.md`, section 4.2) existe exactement dans ce but : permettre à une interface de re-convertir une date UTC pour l'affichage, sans jamais réinterpréter la donnée stockée elle-même.

---

## 7. Conventions CSV

Cette section consolide, sous forme de convention transversale, les règles déjà détaillées dans `README_IMPORT_EXPORT.md` (section 8) — tout nouveau fichier CSV introduit par le projet doit s'y conformer, pas seulement `tasks.csv` et `events.csv`.

| Règle                                                   | Valeur retenue                                                                          | Justification (rappel)                                                                                                                                                                                                   |
| ------------------------------------------------------- | --------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Séparateur de colonnes                                  | Virgule (`,`)                                                                           | Standard RFC 4180, le mieux supporté par PHP (`fgetcsv`) et les tableurs                                                                                                                                                 |
| Séparateur interne (valeurs multiples dans une cellule) | Pipe (`\|`)                                                                             | Évite tout conflit avec le séparateur de colonnes                                                                                                                                                                        |
| Encodage                                                | UTF-8 **sans BOM**                                                                      | Certains lecteurs non-Windows interprètent mal un BOM en tête de fichier                                                                                                                                                 |
| Retours à la ligne                                      | LF (`\n`)                                                                               | Convention Unix/PHP, compatible avec les principaux tableurs                                                                                                                                                             |
| Guillemets                                              | Doubles (`"`), doublés pour échapper un guillemet interne                               | RFC 4180                                                                                                                                                                                                                 |
| Valeurs nulles / absentes                               | Chaîne vide (`""`), **jamais** la chaîne littérale `"NULL"` ou `"null"`                 | Une chaîne `"NULL"` serait ambiguë avec une valeur textuelle légitime portant ce contenu ; la cellule vide est la convention CSV standard pour l'absence de valeur                                                       |
| Ordre des colonnes                                      | Fixe par version de format (`format_version`), documenté dans `README_IMPORT_EXPORT.md` | Un lecteur strict peut se fier à la position des colonnes en plus de leur nom d'en-tête ; toute nouvelle colonne s'ajoute **en fin de fichier**, jamais insérée au milieu, pour rester rétrocompatible (voir section 16) |
| Première ligne                                          | Toujours une ligne d'en-tête, avec les noms exacts de colonnes en `snake_case`          | Permet une lecture par nom de colonne, plus robuste qu'une lecture par position seule                                                                                                                                    |

### 7.1. Exemple

```csv
task_id,name,description,daily_target,target_days,status,created_at
a1e4d2f0-1111-4a2b-9c3d-000000000001,Faire du sport,,1,MON|WED|FRI,ACTIVE,2026-01-15T08:00:00Z
```

La colonne `description` vide illustre la convention "valeur nulle = chaîne vide".

---

## 8. Conventions JSON

### 8.1. Casse des clés : `snake_case`, décision motivée

| Option                    | Avantage                                                                                                     | Inconvénient                                                                                                                                                                                          |
| ------------------------- | ------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `camelCase`               | Convention dominante dans l'écosystème JavaScript/Flutter (Dart utilise aussi le camelCase par convention)   | Nécessiterait une couche de traduction entre les colonnes SQL (`snake_case`), les en-têtes CSV (`snake_case`) et le JSON — source d'erreurs et de code de conversion superflu                         |
| **`snake_case` (retenu)** | Identique à la base de données et au format CSV — aucune conversion nécessaire à aucune frontière du système | Moins idiomatique pour un client JavaScript/Dart pur, qui devra adapter ses propres conventions internes (acceptable : la conversion, si souhaitée, reste du ressort du client, pas du contrat d'API) |

**Décision retenue : `snake_case` partout — SQL, CSV, JSON.** Cette uniformité totale entre les trois représentations d'une même donnée est jugée plus précieuse que le confort idiomatique d'un client particulier, d'autant que plusieurs clients aux conventions différentes (JavaScript pour la PWA, Dart pour Flutter) consommeront la même API : aligner le JSON sur l'une des deux conventions client créerait une incohérence pour l'autre de toute façon.

### 8.2. Indentation

- **Réponses de l'API** : JSON compact (sans indentation), pour minimiser la bande passante — pertinent notamment pour un client mobile (Flutter) en connexion limitée.
- **`manifest.json` généré lors d'un export** : indenté sur 2 espaces, car ce fichier a vocation à être ouvert et lu directement par un humain en cas de besoin de diagnostic (`README_IMPORT_EXPORT.md`, section 2.2, sur l'auditabilité du format).

### 8.3. Valeurs absentes : `null` explicite, jamais de clé omise

Une clé optionnelle et non renseignée est toujours présente dans le JSON, avec la valeur `null` — elle n'est jamais simplement omise de l'objet.

```json
{ "description": null }
```

plutôt que

```json
{ }
```

**Justification :** un client qui lit `data.description` reçoit `null` de façon prévisible dans les deux cas grâce à cette convention, alors qu'une omission obligerait chaque client à vérifier au préalable l'existence de la clé avant d'y accéder — une source d'erreurs classique (`undefined is not an object` côté JavaScript, par exemple).

### 8.4. Booléens

Toujours les valeurs natives JSON `true` / `false` — jamais `"true"` / `"false"` en chaîne, ni `1` / `0`.

### 8.5. Ordre des propriétés

L'ordre des clés dans un objet JSON n'est **jamais significatif** et ne doit jamais être présumé par un client (conformité stricte à la spécification JSON). Par convention de lisibilité uniquement, l'ordre généré suit celui du DTO de réponse correspondant (`README_ARCHITECTURE.md`, section 8.3) : identifiant, champs métier principaux, champs de statut, dates.

---

## 9. Types d'événements

Liste officielle et exhaustive, reprise à l'identique de `README_IMPORT_EXPORT.md` (section 7) et `README_DATABASE.md` (section 6.2) — ce tableau est la référence unique à consulter, les autres documents doivent y rester strictement synchronisés.

| `event_type`   | Rôle                                          | Impact métier                                                                          | Exemple                                                    |
| -------------- | --------------------------------------------- | -------------------------------------------------------------------------------------- | ---------------------------------------------------------- |
| `CREATE_TASK`  | Création d'une tâche                          | Point de départ de tout historique pour la tâche                                       | Émis automatiquement à la création                         |
| `CHECK`        | Validation d'une tâche par l'utilisateur      | Alimente le calcul de série de progression                                             | `{ "value": 1, "note": "Course à pied" }`                  |
| `RESET`        | Interruption volontaire d'une série en cours  | Démarre un nouveau segment de série (section 5.4), sans effacer l'historique           | `{ "note": "Blessure légère" }`                            |
| `ARCHIVE_TASK` | Retrait du suivi actif                        | La tâche n'apparaît plus dans le tableau de bord quotidien, mais reste consultable     | —                                                          |
| `RESTORE_TASK` | Réactivation d'une tâche archivée             | Symétrique de `ARCHIVE_TASK`                                                           | —                                                          |
| `UPDATE_TASK`  | Modification de la configuration d'une tâche  | Trace précisément ce qui a changé (champ `metadata`, `README_DATABASE.md` section 6.3) | `{"field":"daily_target","old_value":"1","new_value":"2"}` |
| `DELETE_TASK`  | Suppression logique exceptionnelle (ex. RGPD) | Voir section 11.3 pour son lien avec le statut `DELETED`                               | Usage rare, jamais déclenché depuis l'interface standard   |

**Réservé pour une évolution future (non implémenté) :** `PAUSE_TASK` / `UNPAUSE_TASK`, en lien avec le statut `PAUSED` (section 11.4). Toute future implémentation devra suivre la procédure décrite en section 16 (évolution `MINOR` du format, mise à jour coordonnée de ce document et de `README_IMPORT_EXPORT.md`).

---

## 10. Sources d'événements

### 10.1. Convention introduite dans ce document

Chaque événement peut désormais être associé à sa **source** — l'origine technique de l'action qui l'a déclenché. Cette information n'existait pas dans les documents précédents ; elle est introduite ici en anticipation directe des besoins de synchronisation multi-appareils déjà identifiés (`README_IMPORT_EXPORT.md`, section 1.1 et section 14 ; `README_DATABASE.md`, section 6.4).

| Valeur    | Signification                                                                                                                             |
| --------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| `MANUAL`  | Action manuelle dont l'origine technique précise n'est pas distinguée (valeur par défaut en V1, tant qu'une seule interface existe)       |
| `WEB`     | Action effectuée depuis l'interface web serveur (V1)                                                                                      |
| `PWA`     | Action effectuée depuis la Progressive Web App (V2)                                                                                       |
| `ANDROID` | Action effectuée depuis l'application Flutter sur Android                                                                                 |
| `IOS`     | Action effectuée depuis l'application Flutter sur iOS                                                                                     |
| `API`     | Action effectuée par un appel direct à l'API, hors des clients officiels ci-dessus (script personnel, intégration tierce)                 |
| `IMPORT`  | Événement recréé lors d'un import de fichier `.ptracker` (conserve `occurred_at` d'origine, mais signale sa provenance)                   |
| `SYSTEM`  | Événement généré automatiquement par le système lui-même, sans action utilisateur directe (réservé à un usage futur, ex. tâche planifiée) |

### 10.2. Utilité

- **Débogage et audit** : en cas d'incohérence constatée après une synchronisation entre appareils, savoir qu'un événement donné provient de `ANDROID` plutôt que de `PWA` permet de cibler la recherche d'un éventuel bug côté client.
- **Statistiques d'usage** (facultatif, futur) : savoir quelle interface est réellement utilisée au quotidien.
- **Fiabilité de l'import** : la valeur `IMPORT` permet de distinguer, dans un historique fusionné, les événements réimportés depuis une sauvegarde des événements produits nativement — utile pour diagnostiquer un import dupliqué par erreur.

### 10.3. Plan d'intégration (extension actée, non encore implémentée)

Cette convention nécessite une évolution coordonnée de plusieurs documents déjà rédigés, listée ici explicitement pour ne pas être oubliée :

| Document à faire évoluer  | Changement requis                                                                                                                                                                                                                                                          |
| ------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `README_DATABASE.md`      | Ajout d'une colonne `source ENUM('MANUAL','WEB','PWA','ANDROID','IOS','API','IMPORT','SYSTEM') NOT NULL DEFAULT 'MANUAL'` sur la table `events`. La valeur par défaut `'MANUAL'` garantit la rétrocompatibilité avec les lignes déjà existantes au moment de la migration. |
| `README_IMPORT_EXPORT.md` | Ajout d'une colonne optionnelle `source` en fin de `events.csv` (respecte la règle "ajout en fin de fichier", section 7.1) ; passage de `format_version` de `1.0` à `1.1` (évolution `MINOR`, rétrocompatible selon la règle déjà posée dans ce document, section 12.1)    |
| `README_ARCHITECTURE.md`  | Ajout de la propriété `source` sur `Domain\Event` et sur `DTO\Response\EventResponse`                                                                                                                                                                                      |
| `README_API.md`           | Le corps des requêtes `POST /tasks/{uuid}/checks` etc. peut désormais accepter un champ optionnel `source` (par défaut déduit automatiquement du contexte de la requête plutôt que déclaré par le client, pour éviter qu'un client ne mente sur son origine)               |

Tant que cette migration n'a pas été effectuée, aucune valeur `source` n'existe dans le système — cette section documente la convention à respecter **au moment où** cette extension sera implémentée.

---

## 11. Statuts

### 11.1. Statuts actuellement implémentés

| Statut     | Signification                                                        | Table concernée |
| ---------- | -------------------------------------------------------------------- | --------------- |
| `ACTIVE`   | Tâche suivie normalement, apparaît dans le tableau de bord quotidien | `tasks.status`  |
| `ARCHIVED` | Tâche retirée du suivi actif, historique conservé et consultable     | `tasks.status`  |

### 11.2. Diagramme de transition (V1 implémentée)

```
        ┌─────────────┐
        │             │
   CREATE_TASK        │ RESTORE_TASK
        │             │
        ▼             │
   ┌─────────┐   ARCHIVE_TASK   ┌───────────┐
   │  ACTIVE   │ ───────────────▶│  ARCHIVED   │
   └─────────┘                  └───────────┘
```

Chaque transition correspond exactement à un type d'événement (section 9) — il n'existe aucune transition de statut qui ne soit pas tracée par un événement explicite, conformément à la philosophie du projet (section 2).

### 11.3. `DELETED` — statut réservé, extension actée

Le statut `DELETED` n'existe pas dans le schéma actuel (`tasks.status` n'accepte que `ACTIVE` et `ARCHIVED`, `README_DATABASE.md` section 4.3). Il est néanmoins documenté ici comme convention à appliquer **le jour où** l'événement `DELETE_TASK` serait effectivement utilisé (`README_IMPORT_EXPORT.md`, section 7.7, qui le qualifie déjà d'exceptionnel).

**Convention retenue pour cette évolution future :** `DELETED` sera un statut **terminal** — aucune transition ne permet d'en sortir (contrairement à `ARCHIVED`, réversible via `RESTORE_TASK`). Ce choix reflète la nature volontairement exceptionnelle et irréversible de la suppression logique, réservée à des cas comme une demande RGPD explicite.

```
   ┌─────────┐                  ┌───────────┐
   │  ACTIVE   │◀──────────────▶│  ARCHIVED   │
   └────┬────┘                  └─────┬─────┘
        │                              │
        │         DELETE_TASK          │
        └──────────────┬───────────────┘
                       ▼
                 ┌───────────┐
                 │  DELETED    │  (terminal, irréversible)
                 └───────────┘
```

Migration requise le jour de l'implémentation : `ALTER TABLE tasks MODIFY status ENUM('ACTIVE','ARCHIVED','DELETED') NOT NULL DEFAULT 'ACTIVE';` — une évolution additive de l'`ENUM`, sans impact sur les valeurs existantes.

### 11.4. `PAUSED` — statut réservé, non implémenté

Distinct de `ARCHIVED` : une tâche `ARCHIVED` sort de l'attention active de l'utilisateur pour une durée indéterminée, sans intention de reprise à court terme (`README_IMPORT_EXPORT.md`, section 7.4). Une tâche `PAUSED` exprimerait au contraire une **suspension temporaire et volontaire**, avec une intention de reprise proche (ex. "je mets ma routine de sport en pause pendant mes vacances"). Cette distinction sémantique justifierait un statut dédié plutôt qu'un simple réemploi d'`ARCHIVED` — mais reste, à ce stade, une piste documentée et non implémentée, en attendant sa validation dans une future révision de `CAHIER_DES_CHARGES.md`.

---

## 12. Conventions SQL

Cette section consolide, en tant que règles transversales applicables à toute nouvelle table, les choix déjà justifiés en détail dans `README_DATABASE.md`.

| Règle                                          | Convention                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| ---------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Clé primaire                                   | `BIGINT UNSIGNED AUTO_INCREMENT`, jamais l'`uuid` public (section 5.2)                                                                                                                                                                                                                                                                                                                                                                                                           |
| Clé étrangère                                  | Toujours vers la clé primaire interne (`BIGINT`) de la table référencée, jamais vers sa colonne `uuid`                                                                                                                                                                                                                                                                                                                                                                           |
| Nommage des contraintes de clé étrangère       | `fk_<table>_<table_référencée>`, ex. `fk_events_task`                                                                                                                                                                                                                                                                                                                                                                                                                            |
| Nommage des index                              | `idx_<table>_<colonne(s)>`, ex. `idx_events_task_occurred`                                                                                                                                                                                                                                                                                                                                                                                                                       |
| Nommage des contraintes d'unicité              | `uk_<table>_<colonne>`, ex. `uk_tasks_uuid`                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| Transactions                                   | Obligatoires pour toute opération touchant plusieurs tables de façon atomique (`README_DATABASE.md`, section 10.1)                                                                                                                                                                                                                                                                                                                                                               |
| Moteur de stockage                             | `InnoDB` exclusivement (`README_DATABASE.md`, section 3.3)                                                                                                                                                                                                                                                                                                                                                                                                                       |
| Jeu de caractères                              | `utf8mb4` / `utf8mb4_0900_ai_ci` exclusivement (`README_DATABASE.md`, section 3.4)                                                                                                                                                                                                                                                                                                                                                                                               |
| Suppression physique de lignes                 | Interdite sur `events` (immuabilité, section 2) ; sur `tasks`, réservée au cas exceptionnel `DELETE_TASK` (section 11.3)                                                                                                                                                                                                                                                                                                                                                         |
| "Soft delete" générique (colonne `deleted_at`) | **Explicitement écartée comme convention par défaut.** Une colonne `deleted_at` implicite masquerait une suppression logique sans laisser de trace explicite dans l'historique — contraire au principe "historique comme source de vérité" (section 2). La convention du projet est de représenter un changement d'état (y compris une suppression logique) par un statut explicite (`status`) **et** un événement associé (section 9), jamais par une colonne technique isolée. |
| Normalisation                                  | Cible 3NF pour toute nouvelle table, sauf redondance explicitement justifiée et documentée (cf. `README_DATABASE.md`, section 1.4, sur la redondance assumée de `tasks` comme projection)                                                                                                                                                                                                                                                                                        |

---

## 13. Conventions PHP

| Règle                  | Convention                                                                                                                                                                                                     |
| ---------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Norme de style de code | PSR-12                                                                                                                                                                                                         |
| Autoloading            | PSR-4, espace de noms racine `ProgressTracker\` (`README_ARCHITECTURE.md`, section 3.1)                                                                                                                        |
| Typage strict          | `declare(strict_types=1);` en première ligne de **chaque** fichier PHP, sans exception                                                                                                                         |
| Visibilité par défaut  | Toujours explicite (`private`, `protected`, `public`) — jamais omise                                                                                                                                           |
| Immutabilité           | Propriétés `readonly` pour tout objet `Domain` et tout DTO (`README_ARCHITECTURE.md`, sections 7 et 8), sauf lorsque la mutation est le rôle même de l'objet (ex. un futur agrégat mutable, hors périmètre V1) |
| Classes                | `final` par défaut, sauf si l'héritage est un besoin explicitement identifié (ex. la classe abstraite `DomainException`, `README_ARCHITECTURE.md` section 9.2)                                                 |
| Exceptions             | Toujours une sous-classe de `DomainException` pour une erreur métier ; jamais une exception PHP générique (`\Exception`, `\RuntimeException`) levée directement dans une couche applicative                    |
| Services               | Suffixe `Service`, un Service par domaine fonctionnel cohérent (`TaskService`, `EventService`, `ProgressCalculationService`) — jamais un unique "Service" fourre-tout                                          |
| Repositories           | Suffixe `Repository` pour l'implémentation, suffixe `RepositoryInterface` pour le contrat ; toujours les deux, jamais une classe concrète utilisée sans interface (`README_ARCHITECTURE.md`, section 2.2)      |
| Validators             | Suffixe `Validator`, une méthode par contexte de validation (`validateForCreation()`, `validateForUpdate()`) plutôt qu'une méthode générique `validate()` ambiguë                                              |
| Helpers                | Nom de rôle clair, sans suffixe imposé, mais toujours sans état ni dépendance métier (`README_ARCHITECTURE.md`, section 13.2)                                                                                  |
| DTO                    | Sous `DTO/Request/` ou `DTO/Response/` selon le sens du flux (`README_ARCHITECTURE.md`, section 8.1) — jamais un DTO utilisé dans les deux sens                                                                |

---

## 14. Conventions API

Rappel synthétique des règles détaillées dans `README_API.md`, applicables à tout nouvel endpoint :

| Règle                | Convention                                                                                                                                                              |
| -------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Format               | JSON exclusivement, `snake_case` (section 8.1)                                                                                                                          |
| Style d'architecture | REST — ressources identifiées par URL, verbes HTTP standards (`README_API.md`, section 6)                                                                               |
| Versionnement        | Dans l'URL, `/api/v1/...` (`README_API.md`, section 2)                                                                                                                  |
| Identifiants exposés | Toujours l'`uuid` public, jamais l'identifiant interne (section 5)                                                                                                      |
| Enveloppe de réponse | `{ "data": ... }` en succès, `{ "error": {...} }` en échec (`README_API.md`, sections 4.2 et 8.1)                                                                       |
| Codes HTTP           | Voir la table complète de `README_API.md`, section 7 — toujours le code le plus précis disponible, jamais un `200` générique pour un succès de création (`201` attendu) |
| Pagination           | Curseur pour `/tasks/{uuid}/events`, aucune pour les listes de taille bornée (`README_API.md`, section 10)                                                              |

---

## 15. Conventions d'import/export

Rappel synthétique de `README_IMPORT_EXPORT.md`, dont ce document ne redéfinit aucune règle mais consolide les points de vigilance transversaux :

- Le format officiel est **`ProgressTrackerExport.ptracker`**, une archive ZIP contenant `manifest.json`, `tasks.csv`, `events.csv` (section 3).
- **`manifest.json` est toujours lu et validé en premier**, avant tout autre fichier de l'archive (section 4.1).
- **`format_version` pilote seule la compatibilité** — jamais `application_version` (section 4.3).
- **Un import est "tout ou rien"** : la moindre ligne invalide dans `tasks.csv` ou `events.csv` annule l'intégralité de l'import, sans écriture partielle en base (`README_IMPORT_EXPORT.md`, section 9.4).
- **Toute erreur de validation est précise et actionnable** (fichier, ligne, champ concerné), jamais un message générique (`README_IMPORT_EXPORT.md`, section 10.2).
- Toute évolution du format suit la règle de compatibilité sémantique déjà posée (section 16 de ce document) : ajout de champ optionnel = version `MINOR`, changement de structure existante = version `MAJOR`.

---

## 16. Conventions de développement

Règles à respecter par tout développeur ajoutant une fonctionnalité au projet, qu'il s'agisse d'une nouvelle route API, d'un nouveau champ, ou d'une nouvelle table.

### 16.1. Ne jamais casser la compatibilité sans décision explicite

Toute modification d'un contrat existant (structure d'une réponse API, colonne CSV, colonne SQL déjà exposée) doit être **additive** par défaut : ajouter un champ optionnel plutôt que d'en modifier ou d'en supprimer un existant. Une rupture de compatibilité volontaire ne peut avoir lieu qu'à l'occasion d'un changement de version `MAJOR` explicitement documenté (`README_IMPORT_EXPORT.md`, section 12 ; `README_API.md`, section 2.2).

### 16.2. Privilégier l'ajout à la modification

Illustré directement par les sections 10 et 11 de ce document : l'ajout de la source d'un événement ou d'un nouveau statut est conçu comme une extension du modèle existant (nouvelle colonne avec valeur par défaut, nouvelle valeur d'`ENUM`), jamais comme une redéfinition de ce qui existe déjà.

### 16.3. Documenter chaque évolution du format avant de l'implémenter

Aucune évolution de schéma, de format d'export ou de contrat d'API ne doit être codée avant d'avoir été **d'abord documentée** dans le(s) document(s) de référence concerné(s) — dans l'esprit de la section 10.3 de ce document, qui documente une extension avant son implémentation plutôt qu'après.

### 16.4. Préserver les anciennes versions du format d'échange

Un import doit toujours rester capable de lire une archive `.ptracker` produite par une version antérieure et compatible du format (`README_IMPORT_EXPORT.md`, section 12.2) — la suppression du support d'une ancienne `format_version` est un changement `MAJOR` à part entière, jamais une conséquence secondaire d'une autre évolution.

### 16.5. Toute migration SQL est écrite une fois, jamais modifiée après application

Une migration déjà appliquée sur une installation (répertoriée dans `schema_migrations`, `README_DATABASE.md` section 11) ne doit **jamais** être éditée rétroactivement. Une correction ou une évolution ultérieure s'exprime toujours par une **nouvelle** migration. Cette règle applique, au niveau du schéma lui-même, exactement le même principe d'immuabilité déjà appliqué à la table `events` (section 2) — une cohérence de principe volontaire entre la gestion du schéma et la gestion des données.

### 16.6. Tests

Tout nouveau Service doit être accompagné de tests unitaires (`tests/Unit/Services/`, `README_ARCHITECTURE.md` section 3), rendus possibles précisément par l'indépendance technique des Services vis-à-vis de HTTP et de PDO (section 2). Un Service qui ne peut pas être testé sans base de données réelle est un signal que la séparation des couches n'a pas été respectée.

---

## 17. Décisions d'architecture (ADR)

Ce registre retrace, sous forme condensée, les décisions structurantes déjà prises et justifiées en détail dans les autres documents. Il ne les redémontre pas ; il les rend **traçables** dans le temps.

### ADR-001 — Identifiants publics UUID v4 + clé interne `BIGINT`

- **Contexte :** besoin d'identifiants stables entre installations, performants en clé primaire InnoDB.
- **Décision :** clé interne `BIGINT AUTO_INCREMENT` + colonne publique `uuid CHAR(36)`.
- **Justification :** voir section 5.2 de ce document et `README_DATABASE.md`, section 3.1.
- **Conséquences :** toute nouvelle entité publiquement identifiable suit cette même convention (section 5.2) ; les FK internes utilisent toujours l'`id`, jamais l'`uuid`.

### ADR-002 — Immuabilité stricte de la table `events`

- **Contexte :** l'historique des événements est la source de vérité unique du système.
- **Décision :** aucune opération `UPDATE`/`DELETE` n'est possible sur `events`, ni au niveau applicatif (`EventRepositoryInterface`), ni au niveau des privilèges MySQL.
- **Justification :** `README_DATABASE.md`, section 3.5.
- **Conséquences :** toute correction d'un événement passe par un nouvel événement compensatoire, jamais par une modification rétroactive.

### ADR-003 — Architecture en couches sans framework complet

- **Contexte :** objectif pédagogique explicite du projet (`CAHIER_DES_CHARGES.md`).
- **Décision :** architecture faite main (Controllers/Services/Repositories), routage léger sans framework applicatif complet.
- **Justification :** `README_ARCHITECTURE.md`, section 3.2.
- **Conséquences :** certaines briques usuelles (validation, injection de dépendances) doivent être conçues explicitement plutôt que fournies automatiquement.

### ADR-004 — Authentification par jeton porteur plutôt que session ou OAuth2

- **Contexte :** l'API doit être consommée par des clients non-navigateurs (Flutter).
- **Décision :** authentification par Bearer Token, jeton unique en V1 mono-utilisateur.
- **Justification :** `README_API.md`, section 3.
- **Conséquences :** l'évolution vers OAuth2 ou un système de connexion complet est différée à l'ouverture multi-utilisateur (section 19).

### ADR-005 — Pagination par curseur pour l'historique des événements

- **Contexte :** `events` est la seule table dont le volume peut croître sans limite sur plusieurs années.
- **Décision :** pagination par curseur (*keyset*) pour `GET /tasks/{uuid}/events` uniquement.
- **Justification :** `README_API.md`, section 10.1.
- **Conséquences :** aucune page arbitraire n'est directement accessible sur cet endpoint, seulement une lecture séquentielle.

### ADR-006 — Introduction de la source d'un événement (`source`)

- **Contexte :** anticipation des besoins de synchronisation multi-appareils.
- **Décision :** ajout d'une colonne `source` à `events`, additive et rétrocompatible (`DEFAULT 'MANUAL'`).
- **Justification :** section 10 de ce document.
- **Conséquences :** mise à jour coordonnée de `README_DATABASE.md`, `README_IMPORT_EXPORT.md` (`format_version` → `1.1`) et `README_ARCHITECTURE.md`, listée en section 10.3.

### ADR-007 — `snake_case` uniforme entre SQL, CSV et JSON

- **Contexte :** trois représentations différentes d'une même donnée (base de données, fichier d'échange, API).
- **Décision :** `snake_case` partout, aucune couche de traduction de casse entre elles.
- **Justification :** section 8.1 de ce document.
- **Conséquences :** un client JavaScript ou Dart adapte lui-même la casse à ses propres conventions internes s'il le souhaite ; ce n'est pas la responsabilité du contrat d'API.

---

## 18. Glossaire

| Terme                                      | Définition                                                                                                                                                                                                          |
| ------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Task (Tâche)**                           | Une habitude ou une routine que l'utilisateur souhaite suivre régulièrement (ex. "faire du sport"). Entité définie par une configuration (nom, objectif quotidien, jours actifs) et un historique d'événements.     |
| **Event (Événement)**                      | Un fait immuable et horodaté concernant une tâche (création, validation, reset, archivage...). L'unité de base de la source de vérité du système.                                                                   |
| **Validation**                             | Terme métier désignant l'action de l'utilisateur qui accomplit une tâche ; techniquement représentée par un événement de type `CHECK`.                                                                              |
| **Reset**                                  | Interruption volontaire d'une série de progression en cours, sans effacer l'historique des validations passées ; techniquement représentée par un événement de type `RESET`.                                        |
| **Series (Série de progression / Streak)** | Segment continu de jours où une tâche a été validée conformément à son objectif, délimité par des événements `RESET` ou par le début de l'historique. Concept **dérivé**, jamais stocké (section 5.4).              |
| **Progression**                            | Ensemble des indicateurs calculés à partir de l'historique d'une tâche : série courante, plus longue série, taux de complétion. Toujours recalculée, jamais stockée.                                                |
| **Projection**                             | Représentation de l'état courant d'une entité (ex. la table `tasks`), maintenue à jour à partir des événements mais reconstructible intégralement à partir d'eux si nécessaire (`README_DATABASE.md`, section 1.2). |
| **Source de vérité**                       | La donnée dont dérivent toutes les autres ; dans Progress Tracker, c'est exclusivement la table `events`.                                                                                                           |
| **Source (d'un événement)**                | L'origine technique d'un événement (interface web, application mobile, import, etc.) — voir section 10.                                                                                                             |
| **Import**                                 | Processus de lecture et d'intégration d'un fichier `.ptracker` dans la base de données de l'application (`README_IMPORT_EXPORT.md`, section 10).                                                                    |
| **Export**                                 | Processus de génération d'un fichier `.ptracker` à partir des données de l'application (`README_IMPORT_EXPORT.md`, section 11).                                                                                     |
| **Manifest**                               | Fichier `manifest.json` décrivant le contenu et les versions d'un export ; point d'entrée obligatoire de toute lecture d'une archive `.ptracker`.                                                                   |
| **Archive (archivage)**                    | Action de retirer une tâche du suivi actif sans en supprimer l'historique ; techniquement représentée par un événement `ARCHIVE_TASK` et le statut `ARCHIVED`.                                                      |
| **UUID**                                   | *Universally Unique Identifier* — identifiant standardisé sur 128 bits, utilisé dans Progress Tracker comme identifiant public stable de toute entité (section 5.2).                                                |
| **DTO**                                    | *Data Transfer Object* — objet dont le seul rôle est de transporter des données entre deux couches, sans logique métier (`README_ARCHITECTURE.md`, section 8).                                                      |
| **Repository**                             | Couche exclusivement responsable de la traduction entre les objets métier et leur représentation en base de données (`README_ARCHITECTURE.md`, section 6).                                                          |
| **Service**                                | Couche contenant l'intégralité de la logique métier de l'application, indépendante de toute technologie externe (`README_ARCHITECTURE.md`, section 5).                                                              |

---

## 19. Évolutions futures

Cette section liste les conventions déjà établies dans ce document qui doivent impérativement être **préservées**, et non recommencées, lors des évolutions majeures anticipées pour le projet.

| Évolution anticipée                    | Conventions de ce document déjà prêtes à l'accueillir                                                                                                                                                                                                          |
| -------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **API REST complète**                  | Déjà spécifiée intégralement (`README_API.md`) ; ce document en rappelle les règles transversales (section 14)                                                                                                                                                 |
| **Application Flutter**                | Convention `snake_case` uniforme (section 8.1) évite toute couche de traduction ; convention d'identifiants UUID (section 5) permet la génération hors-ligne côté client                                                                                       |
| **Progressive Web App**                | Aucune convention spécifique supplémentaire : la PWA consomme la même API que tout autre client (`README_ARCHITECTURE.md`, section 16.2)                                                                                                                       |
| **Notifications**                      | Devront suivre la même règle de langue que l'interface utilisateur (français, section 3) ; tout événement système déclenchant une notification future devra utiliser la source `SYSTEM` (section 10)                                                           |
| **Synchronisation multi-appareils**    | Anticipée directement par la convention `source` (section 10) et par le choix d'identifiants UUID génération-anticipée-compatible (section 5.2)                                                                                                                |
| **Cloud / hébergement multi-instance** | Aucune convention actuelle ne présuppose une installation unique, à l'exception du mono-utilisateur (ligne suivante), volontairement isolée                                                                                                                    |
| **Multi-utilisateur**                  | Convention d'identifiant `user_id` déjà réservée (section 5.1) ; chemin d'extension du schéma déjà documenté (`README_DATABASE.md`, section 12.5) ; authentification par jeton déjà compatible avec une émission par utilisateur (`README_API.md`, section 14) |

**Règle de fond, valable pour toute évolution non listée ci-dessus :** toute nouvelle convention doit être ajoutée à ce document **avant** d'être implémentée dans le code, jamais après (section 16.3) — ce document doit rester, à tout moment du projet, un reflet exact et à jour des règles réellement appliquées, jamais un vestige de décisions passées.

---

*Fin du document — `README_CONVENTIONS.md`, référence normative pour `README_ARCHITECTURE.md`, `README_DATABASE.md`, `README_IMPORT_EXPORT.md` et `README_API.md`.*
