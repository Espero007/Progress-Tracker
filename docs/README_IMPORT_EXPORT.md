# README_IMPORT_EXPORT.md — Spécification du système d'import/export

**Projet :** Progress Tracker
**Statut du document :** Spécification de référence
**Portée :** Ce document est la source de vérité pour tout ce qui concerne l'échange de données (export, import, sauvegarde, migration) dans Progress Tracker. Toute implémentation (backend PHP, futurs clients Flutter, outils tiers) doit s'y conformer.

Ce document est cohérent avec les concepts définis dans `README_ARCHITECTURE.md` (couches Service/Repository) et `README_DATABASE.md` (schéma MySQL). Le vocabulaire employé ici — *tâche*, *événement*, *validation*, *série de progression* — est le vocabulaire officiel du projet et doit rester identique dans tous les autres documents.

---

## Sommaire

1. Objectifs
2. Philosophie
3. Format officiel d'échange
4. `manifest.json`
5. `tasks.csv`
6. `events.csv`
7. Types d'événements
8. Conventions
9. Validation avant import
10. Processus d'import
11. Processus d'export
12. Compatibilité
13. Exemples complets
14. Évolutions futures

---

## 1. Objectifs

Un système d'import/export n'est pas une fonctionnalité annexe dans Progress Tracker : c'est une conséquence directe de la philosophie métier du projet (voir section 2). Puisque l'historique des événements est la seule donnée qui compte réellement, il est naturel — et même nécessaire — de pouvoir faire circuler cet historique en dehors de la base de données MySQL qui l'héberge.

### 1.1. Cas d'usage couverts

| Cas d'usage         | Description                                                                                                                                                                                                                 | Priorité           |
| ------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------ |
| **Sauvegarde**      | L'utilisateur télécharge une copie complète de ses données, à conserver localement ou sur un support externe, indépendamment de tout serveur.                                                                               | Essentielle (V1)   |
| **Migration**       | L'utilisateur change d'installation (nouveau serveur, nouvelle base de données, changement d'hébergeur) et souhaite retrouver l'intégralité de son historique.                                                              | Essentielle (V1)   |
| **Synchronisation** | À terme, l'application Flutter et l'application web devront pouvoir échanger des données sans connexion permanente à une API centrale (mode hors-ligne). Le format d'échange sert de base à cette synchronisation différée. | Prévue (V2+)       |
| **Partage**         | Un utilisateur souhaite partager la structure de ses tâches (mais pas nécessairement son historique de validations) avec un autre utilisateur, par exemple pour dupliquer une routine.                                      | Envisagée (future) |

### 1.2. Pourquoi ne pas se contenter d'un simple dump SQL ?

Un dump `mysqldump` serait plus rapide à produire, mais il présente des inconvénients rédhibitoires pour ce projet :

| Critère                                                        | Dump SQL brut                          | Format `.ptracker`                                           |
| -------------------------------------------------------------- | -------------------------------------- | ------------------------------------------------------------ |
| Lisible par un humain                                          | Non                                    | Oui (CSV + JSON)                                             |
| Portable entre moteurs de BDD                                  | Non (lié à MySQL)                      | Oui                                                          |
| Résistant aux évolutions de schéma                             | Non (couplé à la structure des tables) | Oui (le format est une abstraction, pas un miroir du schéma) |
| Exploitable par un futur client Flutter sans base MySQL locale | Non                                    | Oui                                                          |
| Permet un import partiel ou une relecture manuelle             | Difficile                              | Facile                                                       |

Cette comparaison justifie le choix d'un format d'échange **propriétaire mais ouvert et documenté**, découplé du schéma physique de la base de données.

---

## 2. Philosophie

Progress Tracker repose sur un principe fondamental, déjà énoncé dans `CAHIER_DES_CHARGES.md` : **les compteurs ne sont jamais une source de vérité**. Ils sont systématiquement recalculés à partir de l'historique des événements.

### 2.1. Ce que cela signifie concrètement pour l'export

Exporter les données de Progress Tracker, ce n'est **pas** exporter :

- des totaux de validations ;
- des séries (*streaks*) en cours ou passées ;
- des statistiques agrégées (moyennes, taux de complétion, etc.).

C'est exporter :

- la définition des tâches (leur configuration) ;
- **l'intégralité du journal d'événements** (*event log*) qui les concerne.

```
┌─────────────────────┐        ┌──────────────────────────┐
│   tasks.csv          │        │   events.csv               │
│   (définitions)       │   +    │   (journal d'événements)   │
└─────────────────────┘        └──────────────────────────┘
                    │                          │
                    └────────────┬─────────────┘
                                 ▼
                    Moteur de recalcul (Service)
                                 ▼
                  Compteurs, séries, statistiques
                  (reconstruits, jamais stockés)
```

### 2.2. Pourquoi cette approche est plus robuste

1. **Aucune divergence possible entre les données et leurs statistiques.** Un compteur stocké peut se désynchroniser de son historique (bug, écriture concurrente, migration incomplète). Un compteur recalculé ne le peut pas, par construction : il est une fonction pure de l'historique.
2. **Le format d'export reste stable même si les règles de calcul évoluent.** Si demain la définition d'une "série" change (par exemple, on autorise un jour de repos par semaine sans casser la série), il suffit de rejouer l'historique existant avec les nouvelles règles. Aucune migration de données n'est nécessaire, seulement une évolution du code de recalcul.
3. **L'export est auditable.** Un utilisateur — ou un développeur — peut ouvrir `events.csv` dans un tableur et comprendre exactement, événement par événement, comment un total a été obtenu. C'est impossible avec un simple compteur.
4. **Cela prépare naturellement la synchronisation multi-appareils** (section 1.1) : un journal d'événements horodaté se fusionne (par ordre chronologique) beaucoup plus proprement que des compteurs qui s'écraseraient mutuellement.

Ce principe est **non négociable** dans la conception du format d'export : `tasks.csv` et `events.csv` ne contiennent donc, par construction, aucune colonne de total, de série ou de statistique.

---

## 3. Format officiel d'échange

### 3.1. Présentation

Le format officiel d'échange de Progress Tracker est un fichier portant l'extension **`.ptracker`**.

```
ProgressTrackerExport.ptracker
```

Ce fichier est, techniquement, **une archive ZIP standard** dont l'extension a été renommée. Ce choix est similaire à celui adopté par de nombreux formats de fichiers modernes (`.docx`, `.xlsx`, `.epub`, `.apk` sont tous des archives ZIP déguisées).

### 3.2. Structure de l'archive

```
ProgressTrackerExport.ptracker
├── manifest.json
├── tasks.csv
└── events.csv
```

| Fichier         | Rôle                                                                                                                      |
| --------------- | ------------------------------------------------------------------------------------------------------------------------- |
| `manifest.json` | Décrit le contenu de l'export : versions, date, fichiers présents. C'est le point d'entrée de toute lecture de l'archive. |
| `tasks.csv`     | Contient la définition de chaque tâche suivie par l'utilisateur.                                                          |
| `events.csv`    | Contient l'intégralité du journal d'événements (la source de vérité, voir section 2).                                     |

### 3.3. Justification du choix "ZIP renommé"

| Alternative envisagée                   | Pourquoi elle a été écartée                                                                                                                                                                                                                                                                                                                                                              |
| --------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Un fichier JSON unique contenant tout   | Peu lisible pour de gros volumes d'événements ; pas de séparation claire entre métadonnées et données ; difficile à ouvrir partiellement dans un tableur.                                                                                                                                                                                                                                |
| Une archive `.tar.gz`                   | Moins répandue et moins bien supportée nativement par les systèmes d'exploitation grand public (Windows en particulier) que le ZIP.                                                                                                                                                                                                                                                      |
| Un format binaire propriétaire          | Illisible sans outil dédié ; contraire à l'esprit "auditable" défendu en section 2.2 ; plus risqué en cas de corruption partielle.                                                                                                                                                                                                                                                       |
| **ZIP renommé en `.ptracker`** (retenu) | Lisible par tout outil ZIP standard (y compris en renommant manuellement l'extension en `.zip`) ; permet de séparer proprement métadonnées (JSON) et données tabulaires (CSV) ; les CSV restent ouvrables individuellement dans un tableur pour inspection ou correction manuelle ; compression native, ce qui réduit la taille des sauvegardes contenant plusieurs années d'historique. |

L'extension personnalisée `.ptracker` (plutôt que `.zip`) a deux intérêts : elle permet une association de fichier dédiée dans l'application (double-clic → ouverture directe dans Progress Tracker) et elle signale clairement qu'il s'agit d'un format applicatif spécifique, même si sa nature ZIP reste documentée et assumée — il ne s'agit pas d'obscurcir le format, seulement de l'identifier.

### 3.4. Ce que l'archive NE contient PAS (V1)

Pour rester simple et prévisible, la V1 du format n'inclut pas :

- de fichiers binaires (images, pièces jointes) ;
- de données de compte utilisateur (identifiants, mots de passe) — l'export porte uniquement sur les données métier ;
- de compteurs ou statistiques pré-calculés (voir section 2).

Ces exclusions sont volontaires et documentées en section 14 comme axes d'évolution possibles.

---

## 4. `manifest.json`

### 4.1. Rôle

`manifest.json` est le **point d'entrée obligatoire** de toute lecture d'un fichier `.ptracker`. Aucun import ne doit commencer par la lecture directe de `tasks.csv` ou `events.csv` : le manifeste doit toujours être lu et validé en premier, car il indique comment interpréter le reste de l'archive.

### 4.2. Champs

| Champ                 | Type                      | Obligatoire | Description                                                                                                                                                                                                            |
| --------------------- | ------------------------- | ----------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `application`         | string                    | Oui         | Nom de l'application ayant généré l'export. Toujours `"progress-tracker"`. Permet de rejeter immédiatement un fichier `.ptracker` qui ne proviendrait pas de l'application (protection basique).                       |
| `application_version` | string (semver)           | Oui         | Version de l'application au moment de l'export (ex. `"1.4.2"`). Informative : utile pour le support et le débogage, mais **jamais utilisée pour décider de la compatibilité d'un import**.                             |
| `format`              | string                    | Oui         | Nom du format d'échange. Toujours `"ptracker"`.                                                                                                                                                                        |
| `format_version`      | string (semver)           | Oui         | Version du format d'échange (ex. `"1.0"`). **C'est ce champ, et lui seul, qui pilote la logique de compatibilité à l'import** (voir section 12).                                                                       |
| `exported_at`         | string (ISO 8601, UTC)    | Oui         | Date et heure de génération de l'export, toujours exprimée en UTC (voir section 8.2).                                                                                                                                  |
| `timezone`            | string (identifiant IANA) | Oui         | Fuseau horaire de l'utilisateur au moment de l'export (ex. `"Europe/Paris"`, `"Africa/Porto-Novo"`). Sert uniquement à l'affichage lors d'une relecture humaine du manifeste ; les données elles-mêmes restent en UTC. |
| `export_id`           | string (UUID v4)          | Oui         | Identifiant unique de cet export précis. Permet de tracer un import ("cet import provient bien de cet export"), utile pour le débogage et pour éviter les imports accidentels en double.                               |
| `files`               | objet                     | Oui         | Liste des fichiers attendus dans l'archive, avec leur nombre de lignes de données (hors en-tête). Sert à une première vérification d'intégrité avant même d'ouvrir les CSV (voir section 9).                           |
| `task_count`          | integer                   | Oui         | Nombre de tâches exportées. Redondant avec `files.tasks_csv.row_count` par lisibilité, mais conservé pour un contrôle rapide.                                                                                          |
| `event_count`         | integer                   | Oui         | Nombre d'événements exportés. Idem.                                                                                                                                                                                    |

### 4.3. Pourquoi `application_version` est différent de `format_version`

C'est une distinction essentielle du système, et une source d'erreurs fréquente si elle n'est pas comprise :

- **`application_version`** décrit *le logiciel* qui a produit le fichier (ex. Progress Tracker 1.4.2). Elle évolue à chaque nouvelle version de l'application, potentiellement très souvent (corrections de bugs, nouvelles fonctionnalités d'interface, etc.), **sans que cela n'impacte forcément la structure des données échangées**.
- **`format_version`** décrit *la structure du fichier `.ptracker` lui-même* — les colonnes des CSV, les champs du manifeste, les types d'événements reconnus. Elle n'évolue que lorsque cette structure change réellement.

**Exemple concret :** la version 1.4.2 de l'application peut corriger un bug d'affichage sans toucher au format d'export ; le fichier généré aura alors `application_version: "1.4.2"` mais `format_version: "1.0"`, identique à celui généré par la version 1.3.0. Inversement, si une future version 2.0.0 de l'application introduit un nouveau type d'événement, elle générera des exports en `format_version: "1.1"` ou `"2.0"` selon l'ampleur du changement (voir section 12), indépendamment du fait que l'application soit en version majeure 2.

Cette séparation permet à la logique d'import de ne raisonner **que sur `format_version`**, ce qui la rend beaucoup plus simple et stable dans le temps : elle n'a jamais besoin de connaître l'historique complet des versions de l'application, seulement celui du format.

### 4.4. Exemple

```json
{
  "application": "progress-tracker",
  "application_version": "1.4.2",
  "format": "ptracker",
  "format_version": "1.0",
  "exported_at": "2026-07-22T09:14:03Z",
  "timezone": "Africa/Porto-Novo",
  "export_id": "b3f2c9a0-6d3e-4f1a-9e2b-8c7d5a1f0e4d",
  "task_count": 5,
  "event_count": 812,
  "files": {
    "tasks_csv": {
      "filename": "tasks.csv",
      "row_count": 5
    },
    "events_csv": {
      "filename": "events.csv",
      "row_count": 812
    }
  }
}
```

---

## 5. `tasks.csv`

### 5.1. Rôle

`tasks.csv` contient la **définition** de chaque tâche : sa configuration telle qu'elle existe au moment de l'export. Il ne contient aucune statistique ni aucun historique — cela relève exclusivement de `events.csv`.

### 5.2. Colonnes

| Colonne        | Type                   | Contraintes                                  | Exemple                                     | Justification                                                                                                                                                                                                                                                                                                                   |
| -------------- | ---------------------- | -------------------------------------------- | ------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `task_id`      | string (UUID v4)       | Obligatoire, unique dans le fichier          | `a1e4d2f0-...`                              | Un identifiant UUID (plutôt qu'un entier auto-incrémenté MySQL) garantit que l'identifiant reste valide indépendamment de la base de données cible. C'est indispensable pour un import sur une nouvelle installation, où un entier auto-incrémenté entrerait en conflit avec des identifiants déjà attribués. Voir section 8.6. |
| `name`         | string                 | Obligatoire, non vide, 255 caractères max    | `Faire du sport`                            | Nom affiché à l'utilisateur.                                                                                                                                                                                                                                                                                                    |
| `description`  | string                 | Optionnel, peut être vide                    | `30 minutes minimum, cardio ou musculation` | Champ libre pour préciser la tâche.                                                                                                                                                                                                                                                                                             |
| `daily_target` | integer                | Obligatoire, ≥ 1                             | `1` (ou `8` pour "boire 8 verres d'eau")    | Nombre d'occurrences attendues par jour actif. Permet de gérer aussi bien les tâches "une fois par jour" que les tâches "plusieurs fois par jour" (ex. hydratation) avec le même modèle.                                                                                                                                        |
| `target_days`  | string                 | Obligatoire, liste de jours séparés par `\|` | `MON\|TUE\|WED\|THU\|FRI`                   | Jours de la semaine où la tâche est active. Le séparateur `\|` (pipe) est utilisé plutôt qu'une virgule pour éviter tout conflit avec le séparateur de colonnes CSV (voir section 8.4).                                                                                                                                         |
| `status`       | string (enum)          | Obligatoire : `ACTIVE` ou `ARCHIVED`         | `ACTIVE`                                    | Statut courant de la tâche. Une tâche archivée n'apparaît plus dans le suivi actif mais son historique reste exporté.                                                                                                                                                                                                           |
| `created_at`   | string (ISO 8601, UTC) | Obligatoire                                  | `2026-01-15T08:00:00Z`                      | Date de création de la tâche. Redondant avec l'événement `CREATE_TASK` correspondant dans `events.csv` (voir section 7.1), mais dupliqué ici volontairement pour permettre une lecture de `tasks.csv` **seul**, sans devoir parcourir tout `events.csv` pour connaître l'ancienneté d'une tâche.                                |

> **Note sur la redondance `created_at` / événement `CREATE_TASK`** : cette duplication est assumée et documentée. `tasks.csv` est une **vue de confort** sur l'état courant, reconstruite à partir des événements au moment de l'export. En cas de divergence détectée entre `tasks.csv` et le rejeu de `events.csv`, c'est **toujours `events.csv` qui fait foi** (voir section 2). L'export garantit cette cohérence par construction, mais un import doit rester défensif face à un fichier corrompu ou modifié manuellement (voir section 9).

### 5.3. Exemple de contenu

```csv
task_id,name,description,daily_target,target_days,status,created_at
a1e4d2f0-1111-4a2b-9c3d-000000000001,Faire du sport,30 minutes minimum,1,MON|WED|FRI,ACTIVE,2026-01-15T08:00:00Z
a1e4d2f0-1111-4a2b-9c3d-000000000002,Boire de l'eau,Objectif 8 verres par jour,8,MON|TUE|WED|THU|FRI|SAT|SUN,ACTIVE,2026-01-15T08:05:00Z
a1e4d2f0-1111-4a2b-9c3d-000000000003,Lire,,1,MON|TUE|WED|THU|FRI|SAT|SUN,ARCHIVED,2026-02-01T19:30:00Z
```

---

## 6. `events.csv`

### 6.1. Rôle

`events.csv` est **le cœur du système d'export**. C'est le journal complet, chronologique et immuable de tout ce qui s'est produit dans l'application pour les tâches exportées. Conformément à la philosophie décrite en section 2, c'est la seule source à partir de laquelle toute statistique doit être reconstruite.

### 6.2. Colonnes

| Colonne      | Type                   | Contraintes                                                         | Exemple                                | Justification                                                                                                                                                                                                                                                                                                                                           |
| ------------ | ---------------------- | ------------------------------------------------------------------- | -------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `event_id`   | string (UUID v4)       | Obligatoire, unique dans le fichier                                 | `e7c1a900-...`                         | Comme pour `task_id`, un UUID garantit l'absence de collision lors d'une fusion future de plusieurs journaux (cas d'usage "synchronisation", section 1.1). Un entier auto-incrémenté serait inutilisable dans ce scénario.                                                                                                                              |
| `datetime`   | string (ISO 8601, UTC) | Obligatoire                                                         | `2026-07-20T18:32:10Z`                 | Horodatage exact de l'événement, toujours en UTC (voir section 8.2). C'est ce champ qui définit l'ordre chronologique du journal — un ordre essentiel puisque le recalcul des séries dépend de la succession des événements.                                                                                                                            |
| `event_type` | string (enum)          | Obligatoire, voir section 7                                         | `CHECK`                                | Type de l'événement.                                                                                                                                                                                                                                                                                                                                    |
| `task_id`    | string (UUID v4)       | Obligatoire, doit référencer un `task_id` existant dans `tasks.csv` | `a1e4d2f0-1111-4a2b-9c3d-000000000001` | Tous les types d'événements du système sont rattachés à une tâche (voir section 7) ; il n'existe pas, en V1, d'événement "global" indépendant d'une tâche.                                                                                                                                                                                              |
| `value`      | integer ou vide        | Optionnel selon `event_type`                                        | `1`                                    | Quantité associée à l'événement, pertinente essentiellement pour `CHECK` (ex. "2 verres d'eau bus en une validation"). Vide (`""`) pour les types d'événements où une quantité n'a pas de sens (ex. `ARCHIVE_TASK`).                                                                                                                                    |
| `note`       | string                 | Optionnel                                                           | `Séance en salle`                      | Commentaire libre de l'utilisateur. **Également utilisé comme extension structurée** pour certains types d'événements qui nécessitent plus qu'une simple valeur numérique (typiquement `UPDATE_TASK`, voir section 6.3) — dans ce cas, `note` contient un fragment JSON échappé selon les règles CSV standard (section 8.5), plutôt que du texte libre. |

### 6.3. Le champ `note` comme extension structurée

Le choix de limiter `events.csv` à six colonnes fixes (plutôt que d'ajouter des colonnes optionnelles par type d'événement, ce qui produirait un fichier très creux) a une conséquence : certains événements, comme `UPDATE_TASK`, doivent transporter plus d'information qu'une simple valeur numérique (par exemple : quel champ a changé, ancienne valeur, nouvelle valeur).

Pour ces cas, le champ `note` contient un objet JSON sérialisé sous forme de chaîne, correctement échappé selon les règles CSV (guillemets doublés, voir section 8.5) :

```csv
event_id,datetime,event_type,task_id,value,note
e7c1a900-...,2026-07-21T10:00:00Z,UPDATE_TASK,a1e4d2f0-...,,"{""field"":""daily_target"",""old_value"":""1"",""new_value"":""2""}"
```

Ce choix a été préféré à l'ajout de colonnes `old_value` / `new_value` dédiées, pour deux raisons :

1. Il évite d'alourdir la structure du fichier pour un besoin qui ne concerne qu'un seul type d'événement sur sept.
2. Il rend le format **extensible sans rupture** : un futur type d'événement nécessitant des métadonnées structurées différentes pourra utiliser le même mécanisme (un objet JSON dans `note`) sans qu'il soit nécessaire d'ajouter une nouvelle colonne — donc sans changer `format_version` de manière majeure (voir section 12).

### 6.4. Exemple de contenu

```csv
event_id,datetime,event_type,task_id,value,note
e7c1a900-2222-4b3c-8d4e-000000000001,2026-01-15T08:00:00Z,CREATE_TASK,a1e4d2f0-1111-4a2b-9c3d-000000000001,,
e7c1a900-2222-4b3c-8d4e-000000000002,2026-01-16T07:45:00Z,CHECK,a1e4d2f0-1111-4a2b-9c3d-000000000001,1,Course à pied 5km
e7c1a900-2222-4b3c-8d4e-000000000003,2026-01-18T07:50:00Z,CHECK,a1e4d2f0-1111-4a2b-9c3d-000000000001,1,
e7c1a900-2222-4b3c-8d4e-000000000004,2026-01-22T09:00:00Z,RESET,a1e4d2f0-1111-4a2b-9c3d-000000000001,,Série interrompue volontairement
e7c1a900-2222-4b3c-8d4e-000000000005,2026-02-01T19:30:00Z,ARCHIVE_TASK,a1e4d2f0-1111-4a2b-9c3d-000000000003,,
```

---

## 7. Types d'événements

Chaque ligne de `events.csv` représente un fait immuable. Voici la liste exhaustive des types reconnus en `format_version: "1.0"`.

### 7.1. `CREATE_TASK`

- **Rôle :** marque la création d'une tâche.
- **`value` :** vide.
- **`note` :** vide ou commentaire libre.
- **Impact métier :** point de départ de toute progression pour cette tâche ; aucune validation antérieure à cet événement ne doit être considérée comme valide lors d'un recalcul.

### 7.2. `CHECK`

- **Rôle :** marque la validation d'une tâche par l'utilisateur — l'événement central de l'application.
- **`value` :** quantité validée (par défaut `1`, ou un nombre supérieur pour les tâches à `daily_target` multiple, ex. verres d'eau).
- **`note` :** commentaire libre optionnel.
- **Impact métier :** c'est cet événement, agrégé par jour et comparé à `daily_target`, qui détermine si une journée est "complète" pour une tâche donnée — et donc si elle compte dans le calcul d'une série de progression.

### 7.3. `RESET`

- **Rôle :** interrompt volontairement une série de progression en cours, sans supprimer l'historique des validations passées.
- **`value` :** vide.
- **`note` :** raison du reset, optionnelle mais recommandée pour la lisibilité de l'historique.
- **Impact métier :** lors du recalcul, aucune série ne doit "traverser" un événement `RESET` : la série repart de zéro à partir de la date du reset, mais les `CHECK` antérieurs restent visibles dans les statistiques globales (nombre total de validations, par exemple).

### 7.4. `ARCHIVE_TASK`

- **Rôle :** retire une tâche du suivi actif sans la supprimer.
- **`value` :** vide.
- **`note` :** optionnelle.
- **Impact métier :** une tâche archivée n'apparaît plus dans les listes de suivi quotidien, mais son historique complet reste exporté et consultable.

### 7.5. `RESTORE_TASK`

- **Rôle :** réactive une tâche précédemment archivée.
- **`value` :** vide.
- **`note` :** optionnelle.
- **Impact métier :** symétrique de `ARCHIVE_TASK`. Une tâche peut être archivée puis restaurée plusieurs fois au cours de son existence ; chaque transition est tracée individuellement, ce qui permet de reconstituer précisément les périodes d'activité de la tâche.

### 7.6. `UPDATE_TASK`

- **Rôle :** trace une modification de la configuration d'une tâche (nom, description, `daily_target`, `target_days`).
- **`value` :** vide.
- **`note` :** objet JSON structuré décrivant le changement (voir section 6.3).
- **Impact métier :** important pour l'auditabilité — permet de savoir, par exemple, qu'une tâche dont l'objectif est passé de "1 fois par jour" à "2 fois par jour" ne doit pas être jugée avec les mêmes critères avant et après cette date lors d'un recalcul historique fin.

### 7.7. `DELETE_TASK`

- **Rôle :** trace la suppression logique d'une tâche.
- **`value` :** vide.
- **`note` :** optionnelle.
- **Impact métier :** ⚠️ Ce type d'événement est présent dans le format pour complétude, mais **son usage réel dépend d'une décision produit à trancher dans `CAHIER_DES_CHARGES.md`** : Progress Tracker privilégie l'archivage (`ARCHIVE_TASK`) plutôt que la suppression définitive, afin de ne jamais perdre d'historique. `DELETE_TASK` ne devrait donc être émis, le cas échéant, que dans des cas très spécifiques (ex. suppression RGPD explicite demandée par l'utilisateur), et jamais comme action courante de l'interface. Il est documenté ici pour que le format soit prêt le jour où ce besoin se confirme, sans qu'une évolution de `format_version` soit nécessaire à ce moment-là.

### 7.8. Tableau récapitulatif

| `event_type`   | `value` utilisé ? | `note` structurée ? | Fréquence attendue |
| -------------- | ----------------- | ------------------- | ------------------ |
| `CREATE_TASK`  | Non               | Non                 | Une fois par tâche |
| `CHECK`        | Oui               | Non                 | Très fréquent      |
| `RESET`        | Non               | Non (texte libre)   | Occasionnel        |
| `ARCHIVE_TASK` | Non               | Non                 | Rare               |
| `RESTORE_TASK` | Non               | Non                 | Rare               |
| `UPDATE_TASK`  | Non               | Oui (JSON)          | Occasionnel        |
| `DELETE_TASK`  | Non               | Non                 | Exceptionnel       |

---

## 8. Conventions

Cette section fixe les règles transversales qui s'appliquent à l'ensemble du format. Elles doivent être respectées à l'identique par tout export et tout import, quel que soit le langage ou la plateforme (PHP aujourd'hui, potentiellement Dart/Flutter demain).

### 8.1. Encodage

Tous les fichiers texte de l'archive (`manifest.json`, `tasks.csv`, `events.csv`) sont encodés en **UTF-8 sans BOM** (*Byte Order Mark*). L'absence de BOM est un choix délibéré : certains lecteurs CSV (notamment sur des environnements non-Windows) interprètent mal un BOM en tête de fichier, ce qui peut décaler la lecture de la première colonne.

### 8.2. Dates et fuseaux horaires

- Toutes les dates stockées dans les données (`created_at`, `datetime`, `exported_at`) sont exprimées en **UTC**, au format **ISO 8601** avec le suffixe `Z` : `2026-07-22T09:14:03Z`.
- Le champ `timezone` du manifeste (section 4.2) est **informatif uniquement** : il permet à l'interface de ré-afficher les dates dans le fuseau horaire d'origine de l'utilisateur, mais ne doit jamais être utilisé pour interpréter les dates elles-mêmes.
- **Justification :** stocker les dates en UTC élimine toute ambiguïté lors d'une synchronisation entre appareils situés dans des fuseaux horaires différents, et évite les bugs classiques liés aux changements d'heure été/hiver.

### 8.3. Séparateur CSV

Le séparateur de colonnes est la **virgule** (`,`), conformément au standard **RFC 4180**. Ce standard est choisi plutôt qu'un séparateur alternatif (point-virgule, tabulation) car c'est le plus largement supporté par les bibliothèques de traitement CSV, y compris la fonction native PHP `fgetcsv()` / `str_getcsv()` utilisée par le backend.

### 8.4. Séparateur interne (`target_days`)

Le champ `target_days` (section 5.2) utilise le **pipe** (`|`) comme séparateur de valeurs à l'intérieur d'une même cellule CSV. Ce choix évite tout conflit avec la virgule, séparateur de colonnes, et avec le point-virgule parfois utilisé en alternative dans certains tableurs européens.

### 8.5. Échappement des caractères spéciaux

Le format suit strictement les règles d'échappement CSV de la RFC 4180 :

- Toute valeur contenant une virgule, un guillemet double ou un retour à la ligne doit être **encadrée de guillemets doubles** (`"`).
- Un guillemet double présent dans une valeur doit être **doublé** (`""`).

**Exemple :** une note utilisateur `Séance de sport, 45 min "intense"` doit être écrite ainsi dans le CSV :

```csv
"Séance de sport, 45 min ""intense"""
```

Cette règle est particulièrement critique pour le champ `note`, à la fois parce qu'il contient du texte libre saisi par l'utilisateur (susceptible de contenir virgules et guillemets) et parce qu'il peut contenir du JSON échappé (section 6.3), qui contient lui-même des guillemets doubles en grand nombre.

### 8.6. Identifiants

Tous les identifiants (`task_id`, `event_id`, `export_id`) sont des **UUID version 4**.

Ce choix est préféré à des entiers auto-incrémentés pour trois raisons :

1. **Portabilité :** un UUID généré côté client ou serveur reste valide quelle que soit la base de données cible, contrairement à un entier qui dépend de l'état d'une séquence `AUTO_INCREMENT` propre à chaque installation.
2. **Absence de collision lors d'une fusion :** dans la perspective de la synchronisation multi-appareils (section 1.1), deux appareils peuvent générer des événements simultanément ; des UUID garantissent qu'ils ne s'entreront jamais en collision, contrairement à des entiers auto-incrémentés qui produiraient inévitablement des doublons.
3. **Génération anticipée :** un UUID peut être généré côté client (par exemple dans une future application Flutter fonctionnant hors-ligne) avant même que l'événement soit transmis au serveur, ce qui est impossible avec un identifiant attribué par la base de données.

L'inconvénient (UUID plus long et moins lisible qu'un entier) est jugé acceptable au regard de ces bénéfices, d'autant que ces identifiants ne sont pas destinés à être lus ou saisis manuellement par l'utilisateur final.

### 8.7. Retours à la ligne

Les fichiers CSV utilisent **LF** (`\n`) comme séparateur de ligne, conformément aux conventions Unix/PHP, plutôt que CRLF (`\r\n`). Les lecteurs CSV standards (y compris Excel et Google Sheets) gèrent correctement ce format à l'ouverture.

### 8.8. Ordre des lignes

Les lignes de `events.csv` doivent toujours être écrites **triées par `datetime` croissant**. Ce n'est pas une contrainte strictement nécessaire à la validité du fichier (un import doit rester tolérant à un fichier non trié, voir section 9), mais c'est une convention d'export qui facilite grandement la lecture humaine et le débogage.

---

## 9. Validation avant import

Avant toute écriture en base de données, un import doit être validé de bout en bout. Aucune donnée ne doit être insérée tant que la validation complète n'a pas réussi (voir également le principe de transaction, section 10).

### 9.1. Vérifications sur l'archive

| Vérification                                                        | Comportement en cas d'échec                                     |
| ------------------------------------------------------------------- | --------------------------------------------------------------- |
| Le fichier `.ptracker` est bien une archive ZIP valide              | Rejet immédiat, message "fichier corrompu ou invalide"          |
| `manifest.json` est présent à la racine de l'archive                | Rejet immédiat                                                  |
| `manifest.json` est un JSON valide                                  | Rejet immédiat                                                  |
| `manifest.json` contient tous les champs obligatoires (section 4.2) | Rejet immédiat, liste des champs manquants                      |
| `application` vaut `"progress-tracker"`                             | Rejet immédiat                                                  |
| `format` vaut `"ptracker"`                                          | Rejet immédiat                                                  |
| `format_version` est une version connue et supportée (section 12)   | Rejet, ou déclenchement d'une migration si une stratégie existe |
| `tasks.csv` est présent si annoncé dans `files`                     | Rejet immédiat                                                  |
| `events.csv` est présent si annoncé dans `files`                    | Rejet immédiat                                                  |

### 9.2. Vérifications sur `tasks.csv`

| Vérification                                                                                 | Comportement en cas d'échec                                                             |
| -------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------- |
| L'en-tête contient exactement les colonnes attendues (section 5.2), dans un ordre quelconque | Rejet immédiat                                                                          |
| Chaque `task_id` est un UUID v4 syntaxiquement valide                                        | Rejet de l'import, ligne fautive indiquée                                               |
| Aucun `task_id` n'est dupliqué dans le fichier                                               | Rejet immédiat, identifiants en doublon listés                                          |
| `name` n'est jamais vide                                                                     | Rejet, ligne fautive indiquée                                                           |
| `daily_target` est un entier ≥ 1                                                             | Rejet, ligne fautive indiquée                                                           |
| `target_days` ne contient que des valeurs parmi `MON, TUE, WED, THU, FRI, SAT, SUN`          | Rejet, valeur invalide indiquée                                                         |
| `status` vaut `ACTIVE` ou `ARCHIVED`                                                         | Rejet, valeur invalide indiquée                                                         |
| `created_at` est une date ISO 8601 valide                                                    | Rejet, ligne fautive indiquée                                                           |
| Le nombre de lignes correspond à `files.tasks_csv.row_count` du manifeste                    | Avertissement (non bloquant), car cette incohérence peut indiquer une archive corrompue |

### 9.3. Vérifications sur `events.csv`

| Vérification                                                                                 | Comportement en cas d'échec                                                               |
| -------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| L'en-tête contient exactement les colonnes attendues (section 6.2)                           | Rejet immédiat                                                                            |
| Chaque `event_id` est un UUID v4 syntaxiquement valide et unique dans le fichier             | Rejet immédiat                                                                            |
| Chaque `event_type` fait partie des types reconnus (section 7)                               | Rejet, type inconnu indiqué — sauf stratégie de compatibilité ascendante, voir section 12 |
| Chaque `task_id` référencé existe bien dans `tasks.csv`                                      | Rejet immédiat, référence orpheline signalée                                              |
| `datetime` est une date ISO 8601 valide                                                      | Rejet, ligne fautive indiquée                                                             |
| `value` est vide ou un entier positif                                                        | Rejet, ligne fautive indiquée                                                             |
| Pour `UPDATE_TASK`, `note` est un JSON valide respectant la structure attendue (section 6.3) | Rejet, ligne fautive indiquée                                                             |

### 9.4. Principe général

**Un import est tout ou rien.** Si une seule ligne d'un seul fichier échoue à la validation, **aucune donnée n'est importée**. Ce choix, plus strict qu'un import "au mieux" qui ignorerait les lignes fautives, est justifié par la nature de l'historique d'événements : un import partiel produirait un journal incomplet, ce qui fausserait silencieusement tous les recalculs de séries et de statistiques futurs — une corruption silencieuse est jugée bien plus dangereuse qu'un échec explicite et bloquant.

---

## 10. Processus d'import

Le processus d'import suit une séquence stricte, décrite ci-dessous.

```
┌──────────────────────────┐
│ 1. Réception du fichier   │
│    .ptracker              │
└────────────┬─────────────┘
             ▼
┌──────────────────────────┐
│ 2. Ouverture de l'archive  │
│    ZIP                    │
└────────────┬─────────────┘
             ▼
┌──────────────────────────┐
│ 3. Lecture et validation   │
│    de manifest.json        │
└────────────┬─────────────┘
             ▼
┌──────────────────────────┐
│ 4. Vérification de         │
│    format_version          │
│    (compatible ? migration │
│    nécessaire ? rejet ?)   │
└────────────┬─────────────┘
             ▼
┌──────────────────────────┐
│ 5. Lecture et validation   │
│    complète de tasks.csv   │
│    (en mémoire)            │
└────────────┬─────────────┘
             ▼
┌──────────────────────────┐
│ 6. Lecture et validation   │
│    complète de events.csv  │
│    (en mémoire)            │
└────────────┬─────────────┘
             ▼
┌──────────────────────────┐
│ 7. Validation croisée      │
│    (task_id référencés     │
│    existent bien)          │
└────────────┬─────────────┘
             ▼
      Tout est valide ?
        │           │
      Non          Oui
        │           │
        ▼           ▼
┌──────────────┐ ┌──────────────────────────┐
│ Rejet, aucune │ │ 8. Ouverture d'une         │
│ écriture en   │ │    transaction MySQL       │
│ base           │ └────────────┬─────────────┘
└──────────────┘              ▼
                  ┌──────────────────────────┐
                  │ 9. Insertion des tâches    │
                  │    (ou stratégie de fusion,│
                  │    voir ci-dessous)         │
                  └────────────┬─────────────┘
                               ▼
                  ┌──────────────────────────┐
                  │ 10. Insertion des          │
                  │     événements              │
                  └────────────┬─────────────┘
                               ▼
                  ┌──────────────────────────┐
                  │ 11. Commit de la           │
                  │     transaction             │
                  └────────────┬─────────────┘
                               ▼
                  ┌──────────────────────────┐
                  │ 12. Recalcul des           │
                  │     projections (compteurs,│
                  │     séries) pour les tâches│
                  │     importées               │
                  └──────────────────────────┘
```

### 10.1. Détail des étapes clés

- **Étape 4 (vérification de `format_version`)** : c'est ici que la logique de compatibilité décrite en section 12 intervient. Si la version est ancienne mais supportée, une étape de migration en mémoire est appliquée avant de poursuivre. Si elle est trop ancienne ou plus récente que ce que l'application sait gérer, l'import est rejeté avec un message explicite.
- **Étapes 5 et 6 (lecture en mémoire)** : l'intégralité des fichiers `tasks.csv` et `events.csv` est lue et validée en mémoire **avant** toute écriture en base. Cela permet de garantir le principe "tout ou rien" énoncé en section 9.4 sans avoir à gérer un rollback partiel complexe. Pour des exports de taille raisonnable (quelques années d'historique personnel), ceci reste parfaitement supportable en mémoire ; une stratégie de lecture en flux (*streaming*) est envisagée en section 14 si ce volume devait un jour poser problème.
- **Étape 8 à 11 (transaction)** : l'insertion des tâches puis des événements est réalisée au sein d'une unique transaction MySQL. En cas d'erreur imprévue à n'importe quel moment de cette phase (par exemple une contrainte d'intégrité violée que la validation applicative n'aurait pas anticipée), un **rollback complet** est déclenché : la base de données retrouve son état exact d'avant l'import, comme si celui-ci n'avait jamais eu lieu.
- **Gestion des conflits d'identifiants existants** : si un `task_id` importé existe déjà dans la base cible (cas d'un import répété, ou d'une fusion de deux installations), la stratégie par défaut de la V1 est de **rejeter l'import avec un message explicite**, plutôt que d'écraser silencieusement des données. Une stratégie de fusion plus fine (résolution de conflits événement par événement) est identifiée comme évolution future pour le cas d'usage "synchronisation" (section 1.1) et sera détaillée dans une version ultérieure de ce document lorsque ce cas d'usage sera implémenté.
- **Étape 12 (recalcul)** : conformément à la philosophie du projet (section 2), aucun compteur n'est importé tel quel. Une fois les événements insérés, le moteur de recalcul du Service concerné (voir `README_ARCHITECTURE.md`) est invoqué pour reconstruire les projections à partir de l'historique fraîchement importé.

### 10.2. Messages d'erreur

Chaque étape de validation échouée doit produire un message d'erreur :

- **précis** (il indique quel fichier, quelle ligne, quel champ pose problème) ;
- **actionnable** (il explique ce qui est attendu, pas seulement ce qui a été trouvé) ;
- **non technique dans sa forme utilisateur**, tout en restant loggé de façon détaillée côté serveur pour le débogage.

**Exemple de message utilisateur :** *"L'import a été annulé : la ligne 42 du fichier events.csv fait référence à une tâche inconnue (task_id absent de tasks.csv). Aucune donnée n'a été modifiée."*

---

## 11. Processus d'export

Le processus d'export est plus simple que l'import, puisqu'il part d'une base de données cohérente par construction (les contraintes d'intégrité de MySQL garantissent déjà la validité des données à ce stade — voir `README_DATABASE.md`).

```
┌──────────────────────────┐
│ 1. L'utilisateur demande   │
│    un export                │
└────────────┬─────────────┘
             ▼
┌──────────────────────────┐
│ 2. Génération de           │
│    export_id (UUID)        │
└────────────┬─────────────┘
             ▼
┌──────────────────────────┐
│ 3. Lecture des tâches       │
│    concernées depuis MySQL  │
│    → génération de          │
│    tasks.csv                │
└────────────┬─────────────┘
             ▼
┌──────────────────────────┐
│ 4. Lecture des événements   │
│    concernés depuis MySQL   │
│    → génération de          │
│    events.csv (tri par      │
│    datetime croissant)      │
└────────────┬─────────────┘
             ▼
┌──────────────────────────┐
│ 5. Génération de            │
│    manifest.json            │
│    (versions, dates,        │
│    row_count)               │
└────────────┬─────────────┘
             ▼
┌──────────────────────────┐
│ 6. Compression des 3        │
│    fichiers en une archive  │
│    ZIP                      │
└────────────┬─────────────┘
             ▼
┌──────────────────────────┐
│ 7. Renommage en             │
│    ProgressTrackerExport    │
│    .ptracker                │
└────────────┬─────────────┘
             ▼
┌──────────────────────────┐
│ 8. Mise à disposition du    │
│    fichier au téléchargement│
└──────────────────────────┘
```

### 11.1. Remarques importantes

- **Aucun verrou long sur la base de données.** La génération des CSV se fait par lecture simple (potentiellement dans une transaction en lecture seule pour garantir un instantané cohérent), sans bloquer les écritures concurrentes plus que le temps strictement nécessaire à la lecture.
- **L'export est toujours complet, jamais partiel, en V1.** Un export "partiel" (une seule tâche, une plage de dates) est identifié comme évolution possible (section 14), mais la V1 exporte systématiquement l'intégralité des tâches (actives et archivées) et de leur historique, afin de rester fidèle à la philosophie de sauvegarde complète (section 1.1).
- **`row_count` du manifeste est calculé après génération des CSV**, pas avant, pour garantir qu'il reflète exactement le contenu réel des fichiers produits (et non une estimation faite avant écriture, qui pourrait diverger en cas d'erreur d'écriture partielle).

---

## 12. Compatibilité

### 12.1. Principe général

Le format `.ptracker` doit pouvoir évoluer sans que les exports existants deviennent illisibles. Cette compatibilité repose entièrement sur `format_version`, selon une logique de **versionnage sémantique simplifié** :

```
format_version = "MAJOR.MINOR"
```

- **Incrémenter `MINOR`** (ex. `1.0` → `1.1`) : changement **rétrocompatible**. Un import capable de lire `1.1` doit toujours pouvoir lire `1.0`. Exemples : ajout d'un champ optionnel dans le manifeste, ajout d'un nouveau type d'événement que les anciens lecteurs peuvent simplement ignorer sans casser leur logique.
- **Incrémenter `MAJOR`** (ex. `1.1` → `2.0`) : changement **non rétrocompatible**. Exemples : suppression ou renommage d'une colonne existante, changement de la signification d'un champ.

### 12.2. Stratégie face à une version plus ancienne

Lorsqu'un import reçoit un fichier dont `format_version` correspond à une version `MINOR` antérieure à la version courante supportée par l'application (ex. application en `1.2`, fichier en `1.0`), l'import doit réussir : les champs absents de l'ancien format sont simplement traités avec leur valeur par défaut documentée pour cette version.

### 12.3. Stratégie face à une version plus récente

Lorsqu'un import reçoit un fichier dont `format_version` a un numéro `MAJOR` supérieur à ce que l'application sait lire, l'import doit être **rejeté explicitement**, avec un message invitant l'utilisateur à mettre à jour son application. Il est jugé préférable d'échouer clairement plutôt que de tenter une lecture partielle et potentiellement incorrecte d'un format non maîtrisé.

### 12.4. Migration

Si une évolution `MAJOR` du format devient nécessaire, une fonction de migration dédiée (ex. `migrateFromV1ToV2()`) sera ajoutée au module d'import, responsable de transformer en mémoire une structure `1.x` vers la structure `2.x` avant de poursuivre la validation standard. Cette approche isole la complexité de la migration dans un point unique du code, plutôt que de disséminer des conditions `if (format_version == ...)` dans toute la logique d'import.

### 12.5. Tableau de compatibilité (à tenir à jour)

| `format_version` | Statut           | Notes                                 |
| ---------------- | ---------------- | ------------------------------------- |
| `1.0`            | Version actuelle | Décrite intégralement par ce document |

Ce tableau devra être complété à chaque évolution du format, avec un lien vers la section de ce document (ou un document dédié `CHANGELOG_FORMAT.md`, si le nombre de versions le justifie un jour) décrivant précisément les changements.

---

## 13. Exemples complets

### 13.1. Exemple minimal (une seule tâche, quelques validations)

**`manifest.json`**

```json
{
  "application": "progress-tracker",
  "application_version": "1.0.0",
  "format": "ptracker",
  "format_version": "1.0",
  "exported_at": "2026-01-20T12:00:00Z",
  "timezone": "Africa/Porto-Novo",
  "export_id": "11111111-1111-4111-8111-111111111111",
  "task_count": 1,
  "event_count": 3,
  "files": {
    "tasks_csv": { "filename": "tasks.csv", "row_count": 1 },
    "events_csv": { "filename": "events.csv", "row_count": 3 }
  }
}
```

**`tasks.csv`**

```csv
task_id,name,description,daily_target,target_days,status,created_at
22222222-2222-4222-8222-222222222222,Méditer,10 minutes le matin,1,MON|TUE|WED|THU|FRI|SAT|SUN,ACTIVE,2026-01-15T06:00:00Z
```

**`events.csv`**

```csv
event_id,datetime,event_type,task_id,value,note
33333333-3333-4333-8333-333333333333,2026-01-15T06:00:00Z,CREATE_TASK,22222222-2222-4222-8222-222222222222,,
44444444-4444-4444-8444-444444444444,2026-01-15T06:10:00Z,CHECK,22222222-2222-4222-8222-222222222222,1,Première séance
55555555-5555-4555-8555-555555555555,2026-01-16T06:05:00Z,CHECK,22222222-2222-4222-8222-222222222222,1,
```

### 13.2. Exemple plus riche (plusieurs tâches, reset, archivage, mise à jour)

**`tasks.csv`**

```csv
task_id,name,description,daily_target,target_days,status,created_at
a1e4d2f0-1111-4a2b-9c3d-000000000001,Faire du sport,30 minutes minimum,1,MON|WED|FRI,ACTIVE,2026-01-15T08:00:00Z
a1e4d2f0-1111-4a2b-9c3d-000000000002,Boire de l'eau,Objectif journalier,8,MON|TUE|WED|THU|FRI|SAT|SUN,ACTIVE,2026-01-15T08:05:00Z
a1e4d2f0-1111-4a2b-9c3d-000000000003,Lire,,1,MON|TUE|WED|THU|FRI|SAT|SUN,ARCHIVED,2026-02-01T19:30:00Z
```

**`events.csv`**

```csv
event_id,datetime,event_type,task_id,value,note
e0000001-0000-4000-8000-000000000001,2026-01-15T08:00:00Z,CREATE_TASK,a1e4d2f0-1111-4a2b-9c3d-000000000001,,
e0000001-0000-4000-8000-000000000002,2026-01-15T08:05:00Z,CREATE_TASK,a1e4d2f0-1111-4a2b-9c3d-000000000002,,
e0000001-0000-4000-8000-000000000003,2026-01-16T07:30:00Z,CHECK,a1e4d2f0-1111-4a2b-9c3d-000000000001,1,
e0000001-0000-4000-8000-000000000004,2026-01-16T09:00:00Z,CHECK,a1e4d2f0-1111-4a2b-9c3d-000000000002,4,
e0000001-0000-4000-8000-000000000005,2026-01-18T07:20:00Z,CHECK,a1e4d2f0-1111-4a2b-9c3d-000000000001,1,
e0000001-0000-4000-8000-000000000006,2026-01-25T10:00:00Z,RESET,a1e4d2f0-1111-4a2b-9c3d-000000000001,,Blessure légère
e0000001-0000-4000-8000-000000000007,2026-02-01T19:30:00Z,CREATE_TASK,a1e4d2f0-1111-4a2b-9c3d-000000000003,,
e0000001-0000-4000-8000-000000000008,2026-02-10T20:00:00Z,ARCHIVE_TASK,a1e4d2f0-1111-4a2b-9c3d-000000000003,,
e0000001-0000-4000-8000-000000000009,2026-02-12T08:00:00Z,UPDATE_TASK,a1e4d2f0-1111-4a2b-9c3d-000000000002,,"{""field"":""daily_target"",""old_value"":""8"",""new_value"":""6""}"
```

---

## 14. Évolutions futures

Le format `.ptracker` en `1.0` a été conçu volontairement minimal, pour rester simple à implémenter et à valider dès la V1 de l'application. Plusieurs évolutions sont d'ores et déjà anticipées, sans qu'elles ne remettent en cause les principes fondamentaux décrits dans ce document :

| Évolution envisagée                                                                                       | Impact sur `format_version`                                        | Notes                                                                                                                                                                   |
| --------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Export partiel (par tâche, par plage de dates)                                                            | Aucun (nouveau paramètre d'export, structure de fichier inchangée) | Ne concerne que le processus d'export (section 11), pas le format lui-même.                                                                                             |
| Ajout de pièces jointes (images)                                                                          | `MAJOR`                                                            | Nécessiterait un nouveau dossier `attachments/` dans l'archive et de nouveaux champs de référence — changement structurel.                                              |
| Support de la synchronisation multi-appareils                                                             | `MINOR` probable                                                   | S'appuierait sur les UUID déjà présents (section 8.6) et une stratégie de fusion d'événements par horodatage ; ne devrait pas nécessiter de nouvelle colonne.           |
| Nouveaux types d'événements (ex. `PAUSE_TASK` pour une mise en pause temporaire distincte de l'archivage) | `MINOR`                                                            | Compatible avec la logique d'extension par ignorance décrite en section 12.2, tant que les anciens lecteurs savent ignorer un type d'événement inconnu sans échouer.    |
| Lecture en flux (*streaming*) pour de très gros historiques                                               | Aucun                                                              | Évolution purement technique du processus d'import (section 10), sans impact sur le format lui-même.                                                                    |
| Chiffrement optionnel de l'archive                                                                        | `MINOR` (ajout d'un champ `encrypted: true` dans le manifeste)     | Envisagé si Progress Tracker venait à stocker des données jugées sensibles ; nécessiterait une réflexion dédiée sur la gestion des clés, hors périmètre de ce document. |

Ces pistes sont documentées ici à titre de **veille architecturale** : elles ne doivent pas être implémentées de façon anticipée, mais leur simple identification garantit que les choix faits aujourd'hui (UUID plutôt qu'entiers, séparation manifeste/données, extension par le champ `note`, versionnage indépendant format/application) ne bloquent aucune d'entre elles.

---

*Fin du document — `README_IMPORT_EXPORT.md` version 1.0, cohérent avec `format_version: "1.0"` du format `.ptracker` décrit ci-dessus.*
