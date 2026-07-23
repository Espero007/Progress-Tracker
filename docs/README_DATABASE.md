# README_DATABASE.md — Spécification de la base de données

**Projet :** Progress Tracker
**SGBD cible :** MySQL 8.0+
**Statut du document :** Spécification de référence

Ce document décrit intégralement le schéma relationnel de Progress Tracker : sa philosophie, ses tables, ses contraintes, ses index et ses requêtes types. Il est cohérent avec `CAHIER_DES_CHARGES.md` (vocabulaire métier), `README_ARCHITECTURE.md` (couches Service/Repository) et `README_IMPORT_EXPORT.md` (format d'échange `.ptracker`).

**Hypothèse de portée retenue pour ce document :** la V1 de Progress Tracker est **mono-utilisateur** — aucune table `users` n'existe à ce stade, et aucune colonne `user_id` n'est présente dans le schéma. Cette hypothèse découle de l'absence de gestion multi-utilisateur dans `CAHIER_DES_CHARGES.md`. Le chemin d'évolution vers le multi-utilisateur est documenté en section 12, pour garantir qu'il ne remette pas en cause les choix faits ici.

---

## Sommaire

1. Philosophie de conception
2. Vue d'ensemble
3. Choix transversaux
4. Table `tasks`
5. Table `task_target_days`
6. Table `events`
7. Diagramme relationnel détaillé
8. Règles d'intégrité
9. Index recommandés (récapitulatif)
10. Requêtes SQL d'exemple
11. Table `schema_migrations`
12. Optimisations et évolutions futures

---

## 1. Philosophie de conception

### 1.1. Pourquoi une base de données relationnelle

Progress Tracker manipule des données fortement structurées et reliées entre elles (des tâches, des événements qui leur sont rattachés, des règles d'intégrité claires comme "un événement doit référencer une tâche existante"). Une base relationnelle comme MySQL apporte exactement ce dont ce projet a besoin :

- des **contraintes d'intégrité déclaratives** (clés étrangères, contraintes `CHECK`, unicité) qui empêchent des données incohérentes d'exister, plutôt que de reporter cette responsabilité uniquement sur le code applicatif ;
- des **transactions ACID**, indispensables pour l'import décrit dans `README_IMPORT_EXPORT.md` (principe "tout ou rien", section 9.4 de ce document) ;
- un **langage de requête mature (SQL)**, largement documenté, parfaitement supporté par PHP via PDO.

Une base NoSQL (document ou clé-valeur) a été écartée : elle aurait demandé de réimplémenter manuellement, côté applicatif, des garanties que MySQL offre nativement (intégrité référentielle, contraintes de type), pour un gain de flexibilité que ce projet ne nécessite pas — le schéma de données de Progress Tracker est stable et bien connu dès la conception.

### 1.2. Le couple `tasks` / `events` : projection et source de vérité

Ce schéma matérialise directement la philosophie déjà établie dans `CAHIER_DES_CHARGES.md` et `README_IMPORT_EXPORT.md` : **l'historique des événements est la seule source de vérité.**

Concrètement, cela se traduit en base de données par deux rôles bien distincts :

```
┌─────────────────────────┐         ┌──────────────────────────────┐
│   events                  │         │   tasks                         │
│   (source de vérité)       │ ──────▶ │   (projection / état courant)   │
│   • append-only             │  met à  │   • name, daily_target,         │
│   • jamais modifiée          │  jour   │     target_days, status          │
│   • jamais supprimée         │         │   • mutable, mais toujours       │
│                             │         │     reconstructible à partir      │
│                             │         │     de events                     │
└─────────────────────────┘         └──────────────────────────────┘
```

- **`events` est une table append-only** : en conditions normales, l'application n'exécute jamais d'instruction `UPDATE` ou `DELETE` sur cette table. Une fois un événement inséré, il reste tel quel pour toujours (voir section 3.5 pour l'application stricte de cette règle).
- **`tasks` est une projection (un "read model")** : elle contient l'état courant d'une tâche (son nom, sa cible quotidienne, ses jours actifs, son statut). Cet état est mis à jour à chaque événement pertinent (`UPDATE_TASK`, `ARCHIVE_TASK`, `RESTORE_TASK`...), mais **il doit toujours pouvoir être intégralement reconstruit en rejouant les événements de `events`**. Le fait de le stocker directement dans une table dédiée est une optimisation de lecture assumée (éviter de rejouer tout l'historique à chaque affichage de la liste des tâches), pas une entorse à la philosophie du projet.
- **Aucun compteur, aucune série de progression, aucune statistique n'est stockée dans le schéma.** Ces valeurs sont systématiquement calculées à la volée par la couche Service à partir des lignes de `events` (voir section 10.6).

### 1.3. Pourquoi normaliser

La normalisation (au sens des formes normales relationnelles, 1NF à 3NF principalement) est le principe directeur de ce schéma, pour plusieurs raisons directement alignées avec les objectifs du projet énoncés en préambule (robustesse, maintenabilité, évolutivité) :

1. **Éliminer les anomalies de mise à jour.** Une donnée stockée à un seul endroit ne peut pas devenir incohérente avec elle-même. C'est le prolongement, au niveau du schéma physique, du principe déjà appliqué au niveau métier (section 1.2) : une seule source de vérité par donnée.
2. **Faciliter l'évolution du schéma.** Une table normalisée isole chaque responsabilité (voir par exemple `task_target_days`, section 5), ce qui permet de faire évoluer une partie du modèle sans effet de bord sur le reste.
3. **Garantir l'intégrité par construction.** Une contrainte de clé étrangère bien placée empêche des états incohérents (ex. un événement pointant vers une tâche inexistante) sans qu'aucune ligne de code applicatif n'ait à le vérifier explicitement à chaque insertion.

### 1.4. Pourquoi éviter les redondances — et où ce projet en accepte une, consciemment

La redondance de données est en général un défaut de conception : une même information stockée à deux endroits peut diverger, silencieusement, au fil du temps.

Ce schéma accepte cependant **une redondance délibérée et documentée** : la table `tasks` duplique, sous forme d'état courant, des informations qui sont en théorie déjà entièrement dérivables de `events` (voir section 1.2). Cette redondance est jugée acceptable car :

- elle est **unidirectionnelle et contrôlée** : `tasks` est toujours dérivée de `events`, jamais l'inverse ; le sens de la dépendance est clair et unique ;
- elle est **reconstructible** : en cas de doute ou de bug, `tasks` peut être régénérée intégralement en rejouant `events`, ce qui constitue un filet de sécurité qu'une redondance "sauvage" n'offrirait pas ;
- elle est **justifiée par un besoin de performance légitime** (lecture fréquente de l'état courant des tâches, potentiellement plusieurs fois par affichage de l'interface), documenté explicitement plutôt que découvert a posteriori.

C'est la différence essentielle entre une redondance *subie* (un bug de conception) et une redondance *choisie* (une projection assumée, avec sa règle de reconstruction documentée).

### 1.5. Portée V1 : application mono-utilisateur

Comme indiqué en introduction, ce schéma ne comporte pas de notion d'utilisateur. Toutes les tâches et tous les événements appartiennent implicitement à l'unique utilisateur de l'installation. Ce choix simplifie considérablement le schéma (aucune table `users`, aucune colonne `user_id`, aucune contrainte d'appartenance à vérifier dans chaque requête), ce qui est cohérent avec le principe de simplicité mis en avant pour ce projet. Le chemin d'évolution vers le multi-utilisateur, s'il devient nécessaire, est décrit en section 12.5 : il est conçu pour ne pas nécessiter de refonte du schéma actuel, seulement une extension.

---

## 2. Vue d'ensemble

```
┌───────────────────┐        1,n        ┌───────────────────────┐
│      tasks           │ ─────────────────▶ │   task_target_days       │
│                     │                    │  (jours actifs)          │
└─────────┬─────────┘                    └───────────────────────┘
          │ 1
          │
          │ n
          ▼
┌───────────────────┐
│      events          │
│  (journal complet)   │
└───────────────────┘
```

Trois tables composent le cœur du schéma :

| Table | Rôle | Nature |
|---|---|---|
| `tasks` | État courant de chaque tâche suivie | Projection mutable |
| `task_target_days` | Jours de la semaine où une tâche est active | Table de normalisation (1 tâche → plusieurs jours) |
| `events` | Journal complet et immuable de tout ce qui s'est produit | Source de vérité, append-only |

Une quatrième table technique, `schema_migrations`, est recommandée pour la maintenabilité du projet sur plusieurs années (section 11), sans faire partie du modèle métier.

---

## 3. Choix transversaux

Ces décisions s'appliquent à l'ensemble du schéma. Elles sont regroupées ici pour éviter de les répéter table par table, et pour qu'elles soient facilement retrouvables en cas de doute pendant l'implémentation.

### 3.1. Identifiants : clé interne `BIGINT` + identifiant public `UUID`

C'est la décision la plus structurante du schéma. Deux approches ont été comparées.

| Critère | UUID comme clé primaire | `BIGINT AUTO_INCREMENT` interne + `UUID` en colonne unique (retenu) |
|---|---|---|
| Correspondance directe avec `task_id` / `event_id` du format `.ptracker` | Directe, aucune colonne supplémentaire | Nécessite une colonne dédiée en plus de la clé primaire |
| Performance d'insertion (InnoDB, clé primaire = index clusterisé) | Mauvaise : un UUID v4 est aléatoire, chaque insertion provoque des réorganisations de pages dans l'index clusterisé (fragmentation), un problème qui s'aggrave avec des années d'historique dans `events` | Excellente : `AUTO_INCREMENT` est strictement croissant, les insertions se font toujours en fin d'index, sans fragmentation |
| Taille des clés étrangères et des index | 36 octets (`CHAR(36)`) ou 16 octets (`BINARY(16)`) par référence | 8 octets (`BIGINT`) par référence — jointures plus rapides, index plus compacts |
| Génération d'identifiants côté client hors-ligne (utile pour la future application Flutter) | Possible nativement | Possible également, mais uniquement via la colonne `uuid` — la clé interne reste toujours attribuée par MySQL |

**Décision retenue : clé primaire interne `BIGINT UNSIGNED AUTO_INCREMENT`, doublée d'une colonne `uuid CHAR(36) NOT NULL UNIQUE`.**

Justification : sur une table comme `events`, destinée à accumuler potentiellement des dizaines de milliers de lignes sur plusieurs années d'usage (l'objectif affiché du projet), la dégradation de performance d'un UUID en clé primaire InnoDB est un problème documenté et bien connu, qui s'aggraverait avec le temps — exactement le genre de piège que ce projet doit éviter en anticipant dès aujourd'hui. La colonne `uuid` conserve néanmoins tous les bénéfices d'un identifiant public stable, indépendant de la base de données, utilisé pour :

- la correspondance directe avec `task_id` / `event_id` du format d'export (`README_IMPORT_EXPORT.md`, section 8.6) ;
- l'exposition dans la future API REST (les identifiants internes `BIGINT` ne doivent jamais être exposés publiquement — voir `README_API.md` à venir) ;
- la génération anticipée côté client hors-ligne (Flutter).

Les clés étrangères internes (`events.task_id → tasks.id`) utilisent la clé `BIGINT`, jamais la colonne `uuid`, pour bénéficier de jointures optimales.

### 3.2. Dates : `DATETIME` en UTC, jamais `TIMESTAMP`

| Critère | `TIMESTAMP` | `DATETIME` (retenu) |
|---|---|---|
| Plage de valeurs | Limité à 2038 (dépassement de capacité 32 bits) | Jusqu'à l'an 9999 |
| Conversion automatique de fuseau horaire | Oui, basée sur la variable de session `time_zone` du serveur MySQL — source d'erreurs si cette variable change ou diffère entre environnements | Non — la valeur stockée est exactement celle écrite, aucune conversion implicite |
| Cohérence avec le format d'export | — | Directe : `README_IMPORT_EXPORT.md` impose des dates UTC explicites (section 8.2) ; `DATETIME` sans conversion automatique garantit qu'aucune ambiguïté ne peut être introduite par une différence de configuration serveur |

**Décision retenue : toutes les colonnes de date métier (`created_at`, `occurred_at`) sont de type `DATETIME`, et la convention stricte est que toute valeur qui y est écrite est déjà exprimée en UTC.** Cette convention doit être respectée par la couche Repository (voir `README_ARCHITECTURE.md`), qui est responsable de la conversion entre l'heure locale de l'utilisateur (affichage) et l'UTC (stockage) — jamais MySQL.

### 3.3. Moteur de stockage : InnoDB

InnoDB est utilisé pour toutes les tables, sans exception. C'est le moteur par défaut de MySQL 8, et le seul qui offre :

- le support des clés étrangères (indispensable, voir section 8) ;
- les transactions ACID (indispensables pour l'import "tout ou rien", `README_IMPORT_EXPORT.md` section 9.4) ;
- le verrouillage au niveau ligne plutôt qu'au niveau table (important dès que l'application effectue des écritures concurrentes, par exemple de futurs clients synchronisés).

### 3.4. Jeu de caractères et collation

Toutes les tables et colonnes textuelles utilisent `utf8mb4` avec la collation `utf8mb4_0900_ai_ci` (collation par défaut de MySQL 8, insensible à la casse et aux accents).

- `utf8mb4` (plutôt que `utf8`, qui ne couvre en réalité qu'un sous-ensemble d'Unicode sur 3 octets) est nécessaire pour stocker sans perte tout caractère Unicode valide, y compris les emojis que des utilisateurs pourraient inclure dans le nom d'une tâche ou une note.
- La variante `ai_ci` (*accent-insensitive, case-insensitive*) facilite une éventuelle recherche de tâches par nom (ex. "Mediter" retrouve "Méditer"), sans effort applicatif supplémentaire.

### 3.5. Immutabilité de la table `events`

L'immuabilité de `events` est **le principe le plus important de tout ce schéma** ; elle mérite une garantie plus forte qu'une simple convention de code. Deux niveaux de protection sont recommandés, cumulatifs :

1. **Niveau applicatif (obligatoire) :** la couche Repository ne doit exposer, pour la table `events`, que des méthodes `insert()` et `findBy...()`. Aucune méthode `update()` ni `delete()` ne doit exister pour cette table dans le code — l'absence de la possibilité au niveau de l'interface du Repository est une protection plus fiable qu'une simple discipline de ne pas l'utiliser.
2. **Niveau base de données (recommandé, renforcement défense en profondeur) :** l'utilisateur MySQL applicatif (celui utilisé par la connexion PDO de production) peut se voir accorder uniquement les privilèges `SELECT` et `INSERT` sur la table `events`, à l'exclusion de `UPDATE` et `DELETE` :

```sql
REVOKE UPDATE, DELETE ON progress_tracker.events FROM 'app_user'@'%';
GRANT SELECT, INSERT ON progress_tracker.events TO 'app_user'@'%';
```

Cette seconde protection a un avantage décisif : elle rend une modification accidentelle de l'historique **impossible même en cas de bug applicatif**, et pas seulement improbable. C'est cohérent avec le niveau d'exigence de robustesse affiché pour ce projet.

---

## 4. Table `tasks`

### 4.1. Rôle

Contient l'état courant de chaque tâche suivie par l'utilisateur (voir section 1.2 : c'est une projection, pas une source de vérité).

### 4.2. Colonnes

| Colonne | Type SQL | Contraintes | Justification |
|---|---|---|---|
| `id` | `BIGINT UNSIGNED` | `PRIMARY KEY`, `AUTO_INCREMENT` | Clé interne, voir section 3.1. |
| `uuid` | `CHAR(36)` | `NOT NULL`, `UNIQUE` | Identifiant public, correspond à `task_id` dans le format d'export. |
| `name` | `VARCHAR(255)` | `NOT NULL` | Nom affiché à l'utilisateur. 255 caractères est une limite large mais raisonnable pour un champ de titre, cohérente avec la contrainte déjà posée dans `README_IMPORT_EXPORT.md` (section 5.2). |
| `description` | `TEXT` | `NULL` | Champ libre, potentiellement long ; `TEXT` plutôt que `VARCHAR` pour ne pas imposer de limite arbitraire. |
| `daily_target` | `SMALLINT UNSIGNED` | `NOT NULL`, `DEFAULT 1`, `CHECK (daily_target >= 1)` | Nombre d'occurrences attendues par jour actif. `SMALLINT` est largement suffisant (aucun cas d'usage réaliste ne dépasse quelques dizaines) et plus compact qu'un `INT`. |
| `status` | `ENUM('ACTIVE','ARCHIVED')` | `NOT NULL`, `DEFAULT 'ACTIVE'` | Statut courant. L'usage d'un `ENUM` plutôt que d'un `VARCHAR` garantit au niveau du schéma qu'aucune valeur invalide ne peut être insérée, sans dépendre uniquement d'une validation applicative. |
| `created_at` | `DATETIME` | `NOT NULL` | Date de création métier de la tâche, en UTC (section 3.2). Correspond à l'événement `CREATE_TASK` associé dans `events`. |
| `updated_at` | `DATETIME` | `NOT NULL`, `DEFAULT CURRENT_TIMESTAMP`, `ON UPDATE CURRENT_TIMESTAMP` | Colonne purement technique de traçabilité interne (dernière modification de la ligne), **non exportée** dans le format `.ptracker` — à ne pas confondre avec `created_at`, qui est une donnée métier. |

> **Pourquoi `target_days` n'apparaît pas ici :** contrairement au fichier `tasks.csv` du format d'export (qui stocke `target_days` comme une chaîne `MON|WED|FRI` pour rester simple à lire dans un tableur — voir `README_IMPORT_EXPORT.md` section 5.2), le schéma de base de données n'est **pas obligé de reproduire cette structure**. Le format d'export est volontairement découplé du schéma physique (`README_IMPORT_EXPORT.md`, section 3.3). En base de données, cette information multivaluée est normalisée dans une table séparée : `task_target_days` (section 5). La conversion entre les deux représentations est une responsabilité de la couche Service/Repository dédiée à l'import/export, documentée dans `README_IMPORT_EXPORT.md`.

### 4.3. `CREATE TABLE`

```sql
CREATE TABLE tasks (
    id            BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    uuid          CHAR(36)        NOT NULL,
    name          VARCHAR(255)    NOT NULL,
    description   TEXT            NULL,
    daily_target  SMALLINT UNSIGNED NOT NULL DEFAULT 1,
    status        ENUM('ACTIVE', 'ARCHIVED') NOT NULL DEFAULT 'ACTIVE',
    created_at    DATETIME        NOT NULL,
    updated_at    DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP
                                   ON UPDATE CURRENT_TIMESTAMP,

    PRIMARY KEY (id),
    UNIQUE KEY uk_tasks_uuid (uuid),
    KEY idx_tasks_status (status),

    CONSTRAINT chk_tasks_daily_target CHECK (daily_target >= 1)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8mb4
  COLLATE = utf8mb4_0900_ai_ci;
```

### 4.4. Index

| Index | Colonnes | Justification |
|---|---|---|
| `PRIMARY` | `id` | Clé primaire, index clusterisé InnoDB. |
| `uk_tasks_uuid` | `uuid` | Recherche d'une tâche par identifiant public (import, API future) en temps constant, tout en garantissant l'unicité. |
| `idx_tasks_status` | `status` | La liste des tâches actives est affichée en permanence dans l'interface (tableau de bord quotidien) ; cet index rend ce filtre immédiat même avec un grand nombre de tâches archivées accumulées au fil des années. |

---

## 5. Table `task_target_days`

### 5.1. Rôle

Normalise l'information "jours de la semaine où une tâche est active", qui est par nature une donnée multivaluée (une tâche peut être active plusieurs jours).

### 5.2. Comparaison des approches possibles

| Approche | Description | Avantages | Inconvénients |
|---|---|---|---|
| Chaîne `VARCHAR` avec séparateur | `target_days = "MON\|WED\|FRI"` dans une colonne de `tasks` | Simple, correspond directement au format CSV | Viole la 1NF (valeur non atomique) ; impossible d'indexer efficacement ; une requête "quelles tâches sont actives lundi" nécessite un `LIKE` sur une sous-chaîne, lent et fragile |
| Type `SET` de MySQL | `target_days SET('MON','TUE',...)` | Compact (stocké sur un entier), fonctions dédiées (`FIND_IN_SET`) | Reste conceptuellement une violation de la 1NF ; type propriétaire à MySQL, peu portable ; moins lisible dans les outils d'administration génériques |
| **Table de normalisation dédiée (retenu)** | `task_target_days (task_id, day_of_week)` | Pleinement conforme à la 1NF ; index natif et requêtes simples (`WHERE day_of_week = 'MON'`) ; extensible sans effort (ajouter une colonne à cette table n'affecte pas `tasks`) | Une jointure supplémentaire est nécessaire pour reconstituer les jours actifs d'une tâche |

**Décision retenue : table de normalisation dédiée.** Pour un projet dont l'ambition affichée est de servir de référence pendant plusieurs années, la conformité au modèle relationnel classique est jugée préférable à l'économie d'une jointure, d'autant que cette jointure reste extrêmement bon marché (une poignée de lignes par tâche, indexées).

### 5.3. Colonnes

| Colonne | Type SQL | Contraintes | Justification |
|---|---|---|---|
| `task_id` | `BIGINT UNSIGNED` | `NOT NULL`, `FOREIGN KEY → tasks(id)` | Référence la tâche concernée via la clé interne (section 3.1). |
| `day_of_week` | `ENUM('MON','TUE','WED','THU','FRI','SAT','SUN')` | `NOT NULL` | Jour actif. L'`ENUM` reprend exactement les valeurs autorisées par `README_IMPORT_EXPORT.md` (section 5.2), garantissant qu'aucune valeur invalide ne peut exister en base. |

### 5.4. `CREATE TABLE`

```sql
CREATE TABLE task_target_days (
    task_id      BIGINT UNSIGNED NOT NULL,
    day_of_week  ENUM('MON', 'TUE', 'WED', 'THU', 'FRI', 'SAT', 'SUN') NOT NULL,

    PRIMARY KEY (task_id, day_of_week),

    CONSTRAINT fk_task_target_days_task
        FOREIGN KEY (task_id) REFERENCES tasks (id)
        ON DELETE CASCADE
        ON UPDATE RESTRICT
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8mb4
  COLLATE = utf8mb4_0900_ai_ci;
```

> **Pourquoi `ON DELETE CASCADE` ici, mais pas sur `events` (section 6) :** les lignes de `task_target_days` n'ont **aucune valeur historique propre** — elles décrivent uniquement la configuration *courante* d'une tâche. Si une tâche venait à être supprimée physiquement (cas exceptionnel, voir `README_IMPORT_EXPORT.md` section 7.7 sur `DELETE_TASK`), ses lignes de configuration n'ont plus de sens et peuvent disparaître avec elle sans perte d'information réelle. C'est l'exact opposé de `events`, dont chaque ligne constitue un fait historique qui ne doit jamais disparaître silencieusement.

### 5.5. Index

| Index | Colonnes | Justification |
|---|---|---|
| `PRIMARY` | `(task_id, day_of_week)` | Clé composite naturelle : une tâche ne peut être active qu'une seule fois pour un jour donné. Sert également d'index pour retrouver rapidement tous les jours actifs d'une tâche donnée. |

Aucun index supplémentaire n'est nécessaire en V1 ; une requête "quelles tâches sont actives le lundi" (peu fréquente, typiquement utilisée pour un futur tableau de bord hebdomadaire) reste rapide sur une table de cette taille sans index inversé dédié. Un index `(day_of_week, task_id)` pourra être ajouté si ce type de requête devient un point chaud (section 12).

---

## 6. Table `events`

### 6.1. Rôle

Le journal complet, immuable et append-only de tout ce qui s'est produit dans l'application. C'est la table la plus importante du schéma — la source de vérité unique (section 1.2).

### 6.2. Colonnes

| Colonne | Type SQL | Contraintes | Justification |
|---|---|---|---|
| `id` | `BIGINT UNSIGNED` | `PRIMARY KEY`, `AUTO_INCREMENT` | Clé interne, voir section 3.1. Pour cette table en particulier, qui est vouée à croître sans limite pendant toute la durée de vie du projet, le choix d'une clé strictement croissante (plutôt qu'un UUID en clé primaire) est le plus déterminant du schéma en matière de performance. |
| `uuid` | `CHAR(36)` | `NOT NULL`, `UNIQUE` | Identifiant public, correspond à `event_id` dans le format d'export. |
| `task_id` | `BIGINT UNSIGNED` | `NOT NULL`, `FOREIGN KEY → tasks(id)` | Tâche concernée par l'événement. |
| `event_type` | `ENUM('CREATE_TASK','CHECK','RESET','ARCHIVE_TASK','RESTORE_TASK','UPDATE_TASK','DELETE_TASK')` | `NOT NULL` | Reprend exactement la liste des types d'événements définie dans `README_IMPORT_EXPORT.md` (section 7), garantissant qu'aucun type inconnu ne peut être inséré en base sans une migration de schéma explicite. |
| `occurred_at` | `DATETIME` | `NOT NULL` | Date et heure à laquelle l'événement s'est produit *du point de vue métier*, en UTC (section 3.2). Nommée `occurred_at` plutôt que `datetime` pour éviter toute confusion avec le mot-clé de type SQL `DATETIME`, et pour son sens explicite. |
| `value` | `SMALLINT UNSIGNED` | `NULL` | Quantité associée, pertinente surtout pour `CHECK`. `NULL` pour les types d'événements où une quantité n'a pas de sens (voir `README_IMPORT_EXPORT.md`, section 7.8). |
| `note` | `TEXT` | `NULL` | Commentaire libre saisi par l'utilisateur. |
| `metadata` | `JSON` | `NULL` | Données structurées associées à certains types d'événements (notamment `UPDATE_TASK`, voir section 6.3). |
| `recorded_at` | `DATETIME` | `NOT NULL`, `DEFAULT CURRENT_TIMESTAMP` | Date et heure d'**insertion en base** de l'événement — distincte de `occurred_at`. Voir section 6.4. |

### 6.3. Pourquoi séparer `note` et `metadata` (divergence assumée avec le CSV d'export)

`README_IMPORT_EXPORT.md` (section 6.3) documente qu'une seule colonne `note` du fichier `events.csv` sert à la fois de commentaire libre et de conteneur JSON échappé pour les événements structurés comme `UPDATE_TASK`. Cette contrainte s'explique par le format CSV lui-même, qui a un nombre de colonnes fixe et doit rester simple à lire dans un tableur.

Le schéma de base de données **n'a pas cette contrainte** — et comme rappelé en section 4.2, le format d'export est explicitement découplé du schéma physique. Il est donc préférable, en base, de séparer :

- `note` (`TEXT`) : texte libre, non structuré, saisi par l'utilisateur ;
- `metadata` (`JSON`) : données structurées, propres à certains types d'événements.

**Avantages du type `JSON` natif de MySQL 8** par rapport à un simple `TEXT` contenant du JSON sérialisé :

- validation automatique du contenu à l'écriture (MySQL rejette un JSON syntaxiquement invalide) ;
- possibilité d'interroger des champs internes via les fonctions `JSON_EXTRACT()` / l'opérateur `->>`, utile pour du débogage ou des statistiques futures sur les modifications de tâches (ex. "combien de fois `daily_target` a-t-il été modifié ?") ;
- stockage optimisé en interne (format binaire), plus compact qu'une chaîne texte équivalente.

**Conséquence pour la cohérence inter-documents :** la couche Service/Repository responsable de l'export (`README_IMPORT_EXPORT.md`) est chargée de fusionner `note` et `metadata` dans l'unique colonne `note` du CSV au moment de l'export, et inversement de les séparer à l'import. Cette règle de correspondance doit être documentée explicitement dans le code de ce module (et pourra être rappelée dans une future mise à jour de `README_IMPORT_EXPORT.md` si ce niveau de détail d'implémentation y est jugé utile).

### 6.4. `occurred_at` vs `recorded_at`

Cette distinction, absente du format d'export V1 (qui n'expose que `datetime`, équivalent à `occurred_at`), est ajoutée dès la conception du schéma car elle est peu coûteuse à prévoir maintenant et potentiellement précieuse plus tard :

- `occurred_at` répond à la question *"quand cela s'est-il passé ?"* (point de vue métier) ;
- `recorded_at` répond à la question *"quand cela a-t-il été enregistré dans le système ?"* (point de vue technique).

En usage normal, ces deux valeurs sont identiques. Elles peuvent diverger dans des scénarios déjà anticipés par ce projet, notamment la **synchronisation hors-ligne** (`README_IMPORT_EXPORT.md`, section 1.1 et section 14) : un événement créé sur un téléphone sans connexion, puis synchronisé plusieurs heures plus tard, doit conserver son horodatage métier d'origine (`occurred_at`) tout en permettant de savoir a posteriori quand il a effectivement atteint le serveur (`recorded_at`), par exemple pour diagnostiquer un problème de synchronisation.

### 6.5. `CREATE TABLE`

```sql
CREATE TABLE events (
    id           BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    uuid         CHAR(36)        NOT NULL,
    task_id      BIGINT UNSIGNED NOT NULL,
    event_type   ENUM('CREATE_TASK', 'CHECK', 'RESET', 'ARCHIVE_TASK',
                       'RESTORE_TASK', 'UPDATE_TASK', 'DELETE_TASK') NOT NULL,
    occurred_at  DATETIME        NOT NULL,
    value        SMALLINT UNSIGNED NULL,
    note         TEXT            NULL,
    metadata     JSON            NULL,
    recorded_at  DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP,

    PRIMARY KEY (id),
    UNIQUE KEY uk_events_uuid (uuid),
    KEY idx_events_task_occurred (task_id, occurred_at),
    KEY idx_events_occurred (occurred_at),
    KEY idx_events_task_type (task_id, event_type),

    CONSTRAINT fk_events_task
        FOREIGN KEY (task_id) REFERENCES tasks (id)
        ON DELETE RESTRICT
        ON UPDATE RESTRICT
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8mb4
  COLLATE = utf8mb4_0900_ai_ci;
```

> **Pourquoi `ON DELETE RESTRICT` et pas `CASCADE` :** contrairement à `task_target_days` (section 5.4), une suppression physique d'une tâche ne doit **jamais** entraîner la suppression silencieuse de ses événements — ce serait une perte d'historique irréversible, en contradiction directe avec la philosophie du projet (section 1.2). `RESTRICT` empêche MySQL de supprimer une tâche tant qu'il lui reste des événements associés. Si une suppression complète et volontaire devient un jour nécessaire (cas RGPD évoqué dans `README_IMPORT_EXPORT.md`, section 7.7), elle doit être une opération applicative explicite, auditée, qui supprime d'abord consciemment les événements avant la tâche — jamais un effet de bord automatique d'une contrainte de schéma.

### 6.6. Index

| Index | Colonnes | Justification |
|---|---|---|
| `PRIMARY` | `id` | Index clusterisé, croissance strictement séquentielle (section 3.1). |
| `uk_events_uuid` | `uuid` | Recherche par identifiant public (import, API future), avec garantie d'unicité. |
| `idx_events_task_occurred` | `(task_id, occurred_at)` | **L'index le plus important de tout le schéma.** Il correspond exactement au besoin de recalcul décrit en section 1.2 : récupérer, pour une tâche donnée, tous ses événements triés chronologiquement. C'est la requête exécutée à chaque fois que la couche Service doit reconstruire l'état ou les statistiques d'une tâche. |
| `idx_events_occurred` | `occurred_at` | Requêtes transverses à toutes les tâches, triées dans le temps — utilisées par l'export complet (`README_IMPORT_EXPORT.md`, section 11) et par un futur flux d'activité globale. |
| `idx_events_task_type` | `(task_id, event_type)` | Requêtes ciblant un type d'événement précis pour une tâche (ex. "tous les `RESET` de cette tâche", utile pour l'affichage de l'historique des interruptions d'une série). |

---

## 7. Diagramme relationnel détaillé

```
┌────────────────────────────────────┐
│ tasks                                 │
├────────────────────────────────────┤
│ PK  id              BIGINT UNSIGNED   │
│ UQ  uuid             CHAR(36)          │
│     name             VARCHAR(255)      │
│     description      TEXT              │
│     daily_target     SMALLINT UNSIGNED │
│     status           ENUM(...)         │
│     created_at       DATETIME          │
│     updated_at       DATETIME          │
└──────────────┬─────────────┬────────┘
               │ 1            │ 1
               │              │
               │ n            │ n
┌──────────────▼───────┐  ┌───▼──────────────────────────────┐
│ task_target_days        │  │ events                              │
├──────────────────────┤  ├──────────────────────────────────┤
│ PK,FK task_id  BIGINT UN │  │ PK  id              BIGINT UNSIGNED  │
│ PK    day_of_week ENUM   │  │ UQ  uuid             CHAR(36)         │
└──────────────────────┘  │ FK  task_id           BIGINT UNSIGNED  │
                          │     event_type        ENUM(...)        │
                          │     occurred_at        DATETIME         │
                          │     value              SMALLINT UNSIGNED│
                          │     note                TEXT             │
                          │     metadata            JSON             │
                          │     recorded_at         DATETIME         │
                          └──────────────────────────────────┘
```

**Cardinalités :**

- Une tâche (`tasks`) possède zéro, un ou plusieurs jours actifs (`task_target_days`) — relation `1, n`.
- Une tâche (`tasks`) possède un ou plusieurs événements (`events`) — relation `1, n`. En pratique, une tâche possède toujours *au moins* un événement `CREATE_TASK`, créé au même moment que la ligne `tasks` elle-même (voir section 10.1 pour l'implication transactionnelle de cette règle).
- Un événement (`events`) appartient à exactement une tâche — relation `n, 1`, obligatoire (`task_id NOT NULL`).

---

## 8. Règles d'intégrité

### 8.1. Règles appliquées directement par le schéma (déclaratives)

| Règle | Mécanisme |
|---|---|
| Un événement doit référencer une tâche existante | `FOREIGN KEY events.task_id → tasks.id` |
| Une tâche ne peut être supprimée tant qu'elle a des événements | `ON DELETE RESTRICT` sur `fk_events_task` |
| Un jour actif doit référencer une tâche existante | `FOREIGN KEY task_target_days.task_id → tasks.id` |
| `daily_target` ne peut jamais être inférieur à 1 | `CHECK (daily_target >= 1)` sur `tasks` |
| `status` ne peut prendre qu'une valeur parmi `ACTIVE`, `ARCHIVED` | `ENUM` sur `tasks.status` |
| `event_type` ne peut prendre qu'une valeur parmi les 7 types reconnus | `ENUM` sur `events.event_type` |
| `day_of_week` ne peut prendre qu'une valeur parmi les 7 jours | `ENUM` sur `task_target_days.day_of_week` |
| Une tâche ne peut être active deux fois le même jour | Clé primaire composite `(task_id, day_of_week)` sur `task_target_days` |
| Les identifiants publics (`uuid`) sont uniques | `UNIQUE KEY` sur `tasks.uuid` et `events.uuid` |
| La table `events` est en lecture-écriture-ajout seule (jamais de modification) | Privilèges MySQL restreints (section 3.5) |

### 8.2. Règles qui restent de la responsabilité de la couche Service

Certaines règles métier ne peuvent pas — ou ne doivent pas — être portées par le schéma relationnel, car elles dépendent d'une logique qui doit rester indépendante de la base de données, conformément à `README_ARCHITECTURE.md`. Elles sont documentées ici pour mémoire, à titre de passerelle entre ce document et le futur document d'architecture applicative :

| Règle | Pourquoi elle n'est pas dans le schéma |
|---|---|
| `occurred_at` d'un nouvel événement ne devrait normalement pas être antérieur à `created_at` de la tâche concernée | Règle métier avec des exceptions possibles (ex. import de données historiques) ; mieux gérée comme validation applicative avec message d'erreur explicite qu'avec une contrainte SQL rigide. |
| Le contenu de `metadata` doit respecter une structure précise selon `event_type` (ex. `{"field", "old_value", "new_value"}` pour `UPDATE_TASK`) | MySQL peut valider qu'une valeur *est* un JSON valide, mais pas qu'elle respecte un schéma JSON métier précis dépendant d'une autre colonne de la même ligne. Cette validation est effectuée par la couche Service avant insertion. |
| Le calcul des séries de progression (*streaks*) et des statistiques | **Volontairement absent du schéma** : conformément à la philosophie du projet (section 1.2), ces valeurs ne sont jamais stockées ni calculées par une requête SQL complexe unique. Elles sont calculées en PHP, dans la couche Service, à partir des événements bruts récupérés en base (voir section 10.6). Ce choix garantit que la logique de calcul des séries reste testable indépendamment de MySQL et réutilisable telle quelle par la future API REST. |

---

## 9. Index recommandés (récapitulatif)

| Table | Index | Type | Usage principal |
|---|---|---|---|
| `tasks` | `PRIMARY (id)` | Clusterisé | Accès direct par clé interne |
| `tasks` | `uk_tasks_uuid (uuid)` | Unique | Résolution d'un identifiant public |
| `tasks` | `idx_tasks_status (status)` | Secondaire | Liste des tâches actives |
| `task_target_days` | `PRIMARY (task_id, day_of_week)` | Clusterisé composite | Jours actifs d'une tâche |
| `events` | `PRIMARY (id)` | Clusterisé | Accès direct par clé interne |
| `events` | `uk_events_uuid (uuid)` | Unique | Résolution d'un identifiant public |
| `events` | `idx_events_task_occurred (task_id, occurred_at)` | Secondaire composite | **Recalcul des projections** (requête la plus fréquente du système) |
| `events` | `idx_events_occurred (occurred_at)` | Secondaire | Export complet, flux d'activité global |
| `events` | `idx_events_task_type (task_id, event_type)` | Secondaire composite | Filtrage par type d'événement pour une tâche |

---

## 10. Requêtes SQL d'exemple

### 10.1. Créer une tâche (et son événement `CREATE_TASK`, dans une même transaction)

```sql
START TRANSACTION;

INSERT INTO tasks (uuid, name, description, daily_target, status, created_at)
VALUES (UUID(), 'Faire du sport', '30 minutes minimum', 1, 'ACTIVE', UTC_TIMESTAMP());

SET @task_id = LAST_INSERT_ID();

INSERT INTO task_target_days (task_id, day_of_week)
VALUES (@task_id, 'MON'), (@task_id, 'WED'), (@task_id, 'FRI');

INSERT INTO events (uuid, task_id, event_type, occurred_at)
VALUES (UUID(), @task_id, 'CREATE_TASK', UTC_TIMESTAMP());

COMMIT;
```

*Justification de la transaction :* les trois insertions doivent réussir ou échouer ensemble. Sans transaction, un échec après l'insertion de `tasks` mais avant celle de l'événement `CREATE_TASK` laisserait une tâche sans aucune trace de sa création dans l'historique — une incohérence directement contraire à la philosophie du projet (section 1.2).

### 10.2. Enregistrer une validation (`CHECK`)

```sql
INSERT INTO events (uuid, task_id, event_type, occurred_at, value, note)
VALUES (UUID(), :task_id, 'CHECK', UTC_TIMESTAMP(), 1, 'Course à pied 5km');
```

### 10.3. Récupérer l'historique complet d'une tâche, trié chronologiquement (requête de recalcul)

```sql
SELECT id, uuid, event_type, occurred_at, value, note, metadata
FROM events
WHERE task_id = :task_id
ORDER BY occurred_at ASC;
```

Cette requête s'appuie directement sur l'index `idx_events_task_occurred` (section 6.6). C'est la requête fondamentale du système : son résultat est ensuite transmis tel quel à la couche Service, qui reconstruit en PHP les statistiques et séries de progression (aucune agrégation métier complexe n'est effectuée en SQL, conformément à la section 8.2).

### 10.4. Lister les tâches actives, avec leurs jours actifs

```sql
SELECT
    t.id,
    t.uuid,
    t.name,
    t.daily_target,
    GROUP_CONCAT(ttd.day_of_week ORDER BY FIELD(ttd.day_of_week,
        'MON','TUE','WED','THU','FRI','SAT','SUN') SEPARATOR '|') AS target_days
FROM tasks t
LEFT JOIN task_target_days ttd ON ttd.task_id = t.id
WHERE t.status = 'ACTIVE'
GROUP BY t.id, t.uuid, t.name, t.daily_target
ORDER BY t.created_at ASC;
```

*Remarque :* `GROUP_CONCAT` avec `FIELD()` permet de reconstituer, si besoin, une représentation proche de celle utilisée dans `tasks.csv` (`README_IMPORT_EXPORT.md`, section 5.2) — utile notamment pour le module d'export, qui peut réutiliser cette requête telle quelle plutôt que de reconstruire la chaîne en PHP.

### 10.5. Quelles tâches sont actives aujourd'hui ?

```sql
SELECT t.id, t.uuid, t.name, t.daily_target
FROM tasks t
INNER JOIN task_target_days ttd ON ttd.task_id = t.id
WHERE t.status = 'ACTIVE'
  AND ttd.day_of_week = ELT(WEEKDAY(UTC_DATE()) + 1, 'MON','TUE','WED','THU','FRI','SAT','SUN');
```

*Utilisation typique :* alimente le tableau de bord quotidien affiché à l'utilisateur à l'ouverture de l'application.

### 10.6. Ce qui n'est volontairement PAS fait en SQL : le calcul d'une série de progression

Une tentation fréquente serait d'écrire une requête SQL calculant directement une série de progression (streak) en cours, par exemple avec des fonctions de fenêtrage (`ROW_NUMBER()`, `LAG()`). Ce projet **déconseille délibérément cette approche**, pour rester cohérent avec `README_ARCHITECTURE.md` :

- une telle requête serait complexe, difficile à tester unitairement, et coupler fortement la logique métier (définition d'une "série valide") à une syntaxe SQL spécifique à MySQL ;
- elle devrait être dupliquée ou réécrite dans un tout autre langage le jour où un mode hors-ligne (Flutter, sans MySQL local) devra effectuer le même calcul.

**Approche retenue :** la requête 10.3 récupère les événements bruts ; la couche Service (PHP, indépendante de MySQL) applique ensuite l'algorithme de calcul de série. Ce même algorithme, écrit une seule fois, pourra être porté tel quel (ou réimplémenté à l'identique en Dart) pour un usage hors-ligne, puisqu'il ne dépend que d'une liste d'événements en entrée — pas d'une base de données particulière.

### 10.7. Modifier une tâche et tracer la modification

```sql
START TRANSACTION;

UPDATE tasks
SET daily_target = :new_value
WHERE id = :task_id;

INSERT INTO events (uuid, task_id, event_type, occurred_at, metadata)
VALUES (
    UUID(),
    :task_id,
    'UPDATE_TASK',
    UTC_TIMESTAMP(),
    JSON_OBJECT('field', 'daily_target', 'old_value', :old_value, 'new_value', :new_value)
);

COMMIT;
```

### 10.8. Flux d'activité récent (toutes tâches confondues)

```sql
SELECT e.uuid, e.event_type, e.occurred_at, e.value, e.note, t.name AS task_name
FROM events e
INNER JOIN tasks t ON t.id = e.task_id
ORDER BY e.occurred_at DESC
LIMIT 20;
```

---

## 11. Table `schema_migrations`

### 11.1. Rôle

Bien qu'absente du modèle métier, cette table est **fortement recommandée** dès la V1, pour la maintenabilité à long terme explicitement visée par ce projet. Elle permet de savoir précisément, à tout moment et sur toute installation, quelles évolutions du schéma ont été appliquées.

### 11.2. `CREATE TABLE`

```sql
CREATE TABLE schema_migrations (
    version      VARCHAR(14)  NOT NULL COMMENT 'Format AAAAMMJJHHMMSS',
    description  VARCHAR(255) NULL,
    applied_at   DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,

    PRIMARY KEY (version)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8mb4
  COLLATE = utf8mb4_0900_ai_ci;
```

### 11.3. Convention

Chaque évolution du schéma (ajout de colonne, nouvel index, nouvelle table) doit être accompagnée d'un script SQL numéroté par un horodatage (`20260722091400_add_metadata_to_events.sql`, par exemple), et d'une ligne insérée dans `schema_migrations` une fois ce script appliqué. Cette convention, simple, évite qu'une installation se retrouve avec un schéma dans un état inconnu ou partiellement à jour — un risque réel sur un projet destiné à vivre plusieurs années et potentiellement plusieurs environnements (développement local, production, futurs tests automatisés).

---

## 12. Optimisations et évolutions futures

Cette section documente, sans les implémenter dès la V1 (conformément au principe de simplicité), les évolutions déjà anticipées pour ce schéma.

### 12.1. Partitionnement de la table `events` par année

À mesure que l'historique s'accumule sur plusieurs années, un partitionnement `RANGE` sur `occurred_at` (une partition par année) pourrait être introduit :

```sql
-- Illustration, non appliquée en V1
ALTER TABLE events
PARTITION BY RANGE (YEAR(occurred_at)) (
    PARTITION p2026 VALUES LESS THAN (2027),
    PARTITION p2027 VALUES LESS THAN (2028),
    PARTITION pmax  VALUES LESS THAN MAXVALUE
);
```

*Bénéfice attendu :* les requêtes portant sur une période récente (l'immense majorité des requêtes applicatives) n'ont besoin de parcourir que la partition correspondante, améliorant les performances au fur et à mesure que la table grossit. Cette évolution n'est pas nécessaire en V1 (le volume de données d'un usage personnel reste, même après plusieurs années, largement dans les capacités confortables d'InnoDB sans partitionnement), mais le choix d'`occurred_at` comme colonne de date métier bien identifiée (section 6.4) rend cette évolution simple à mettre en œuvre le jour venu.

### 12.2. Archivage à froid des anciennes années

Complémentaire à la piste précédente : les événements de plus de N années pourraient être déplacés vers une table (ou une base) d'archive en lecture seule, allégeant la table `events` active sans perdre l'historique — cohérent avec le principe "conserver toutes les anciennes séries" du cahier des charges.

### 12.3. Index de recherche plein texte sur `tasks.name` / `tasks.description`

Si le nombre de tâches créées (actives et archivées cumulées) devient important, un index `FULLTEXT` sur `name` et `description` permettrait une recherche plus riche qu'un simple `LIKE '%...%'`.

### 12.4. Index inversé sur `task_target_days`

Comme mentionné en section 5.5, un index `(day_of_week, task_id)` pourra être ajouté si les requêtes du type "quelles tâches sont actives tel jour, tous utilisateurs confondus" (pertinent uniquement après une éventuelle évolution multi-utilisateur, section 12.5) deviennent fréquentes.

### 12.5. Chemin d'évolution vers le multi-utilisateur

Bien que hors-périmètre de la V1 (section 1.5), le schéma actuel a été conçu pour ne pas bloquer cette évolution :

1. Ajout d'une table `users (id, uuid, email, password_hash, created_at, ...)`.
2. Ajout d'une colonne `user_id BIGINT UNSIGNED NOT NULL` sur `tasks`, avec `FOREIGN KEY → users(id)`.
3. Aucune modification requise sur `events` ni `task_target_days` : ces tables restent liées à une tâche, qui est elle-même désormais liée à un utilisateur — la relation d'appartenance se propage naturellement par transitivité, sans dupliquer `user_id` partout.
4. Ajout d'un index `idx_tasks_user_status (user_id, status)` pour remplacer l'actuel `idx_tasks_status`, devenu insuffisant seul dans un contexte multi-utilisateur.

Cette évolution reste une **extension additive** du schéma (nouvelles tables, nouvelles colonnes, nouveaux index) et ne nécessite la modification d'aucune contrainte existante — un signe, a posteriori, que la normalisation appliquée dès la V1 (section 1.3) porte ses fruits.

### 12.6. Cohérence à surveiller avec `README_IMPORT_EXPORT.md` en cas d'évolution des identifiants

Une piste théorique d'optimisation consisterait à remplacer les UUID v4 (aléatoires) par des UUID v7 (ordonnés temporellement), qui limiteraient la fragmentation même si la colonne `uuid` était un jour promue en clé primaire. **Cette piste n'est pas retenue ici**, car `README_IMPORT_EXPORT.md` (section 8.6) impose explicitement des UUID v4 dans le format d'échange `.ptracker` en `format_version 1.0`. Adopter unilatéralement un autre format d'identifiant côté base de données casserait la correspondance directe entre `uuid` et les identifiants exportés. Si cette piste devait un jour être suivie, elle devrait être coordonnée avec une évolution `MINOR` ou `MAJOR` du format d'échange (voir `README_IMPORT_EXPORT.md`, section 12), et documentée dans les deux fichiers à la fois — un exemple concret de la vigilance de cohérence inter-documents demandée pour ce projet.

---

*Fin du document — `README_DATABASE.md`, cohérent avec le schéma MySQL 8 décrit ci-dessus et avec `format_version: "1.0"` du format `.ptracker` défini dans `README_IMPORT_EXPORT.md`.*
