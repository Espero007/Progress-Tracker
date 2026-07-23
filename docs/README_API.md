# README_API.md — Documentation officielle de l'API REST

**Projet :** Progress Tracker
**Base URL (V1) :** `https://<domaine>/api/v1`
**Format :** JSON exclusivement
**Statut du document :** documentation officielle de référence, destinée aux développeurs consommant l'API (front-end PWA, application Flutter, outils tiers).

Cette API expose, via HTTP, exactement la même logique métier que décrite dans `README_ARCHITECTURE.md` (Controllers → Services → Repositories). Le vocabulaire employé ici (*tâche*, *événement*, *validation*, *série de progression*) et les identifiants utilisés (`uuid`, types d'événements) sont strictement identiques à ceux de `README_DATABASE.md` et `README_IMPORT_EXPORT.md`.

---

## Sommaire

1. Principes généraux
2. Versionnement
3. Authentification
4. Format des requêtes et réponses
5. Ressources et endpoints
6. Verbes HTTP
7. Statuts HTTP utilisés
8. Format des erreurs
9. Validation
10. Pagination
11. Export et import
12. Exemples complets de requêtes/réponses
13. Sécurité complémentaire
14. Évolutions futures

---

## 1. Principes généraux

### 1.1. Architecture REST

L'API de Progress Tracker suit les principes REST classiques :

- chaque **ressource** (tâche, événement) est identifiée par une URL stable ;
- les opérations sur cette ressource passent par les verbes HTTP standards (section 6), et non par des URL de type `/getTask` ou `/createTask` ;
- l'API est **sans état** (*stateless*) : aucune session serveur n'est maintenue entre deux requêtes ; chaque requête porte elle-même tout ce qui est nécessaire à son traitement, notamment son jeton d'authentification (section 3).

### 1.2. Un identifiant public, jamais un identifiant interne

Conformément à `README_ARCHITECTURE.md` (section 8.3) et `README_DATABASE.md` (section 3.1), **toutes les URL et tous les corps de requête/réponse de cette API utilisent exclusivement l'`uuid` public des tâches et des événements.** L'identifiant interne `BIGINT` utilisé par MySQL n'est jamais exposé, à aucun endroit de l'API.

```
✅  GET /api/v1/tasks/b3f2c9a0-6d3e-4f1a-9e2b-8c7d5a1f0e4d
❌  GET /api/v1/tasks/482
```

### 1.3. Cette API ne duplique aucune règle métier

Comme expliqué dans `README_ARCHITECTURE.md` (section 16.1), les Controllers de cette API ne font qu'appeler les mêmes Services que ceux utilisés par l'application web. Ce document décrit donc une **surface d'exposition HTTP** d'une logique métier qui existe déjà et qui est documentée ailleurs — il ne redéfinit aucune règle de calcul ou de validation, il se contente de préciser comment y accéder par le réseau.

---

## 2. Versionnement

### 2.1. Stratégie retenue : versionnement dans l'URL

| Stratégie | Exemple | Avantages | Inconvénients |
|---|---|---|---|
| **Dans l'URL (retenu)** | `/api/v1/tasks` | Explicite, visible immédiatement dans les logs et les outils de débogage ; compatible avec toute mise en cache HTTP standard (le cache distingue naturellement les URL) ; aucune configuration cliente particulière requise | Une ressource dupliquée virtuellement à chaque version majeure |
| Dans un en-tête HTTP | `Accept: application/vnd.progresstracker.v1+json` | URL stable dans le temps | Moins visible, oublié plus facilement par les clients, plus difficile à tester manuellement (ex. depuis un navigateur) |
| Par paramètre de requête | `/api/tasks?version=1` | Simple à implémenter | Peu conventionnel, facilement omis par erreur, mauvaise pratique largement déconseillée par la communauté REST |

**Décision retenue : versionnement dans l'URL (`/api/v1/...`).** Pour un projet destiné à être consommé par plusieurs clients indépendants (PWA, puis application Flutter, `CAHIER_DES_CHARGES.md`), la visibilité et la simplicité d'usage du versionnement dans l'URL l'emportent sur la légère duplication qu'il implique en cas d'évolution majeure.

### 2.2. Politique de compatibilité

- Un changement **rétrocompatible** (ajout d'un champ optionnel dans une réponse, ajout d'un nouvel endpoint) ne crée pas de nouvelle version : `v1` continue d'évoluer.
- Un changement **non rétrocompatible** (suppression ou renommage d'un champ, changement de la signification d'un endpoint existant) donne lieu à une nouvelle version (`/api/v2/...`), publiée en parallèle de `v1` pendant une période de transition documentée au moment venu.

Cette politique est directement inspirée de celle déjà adoptée pour `format_version` dans `README_IMPORT_EXPORT.md` (section 12) — la cohérence de principe entre le versionnement du format d'échange et celui de l'API est volontaire.

---

## 3. Authentification

### 3.1. Comparaison des approches

| Approche | Adaptée à un client mobile (Flutter) ? | Adaptée à une PWA ? | Complexité |
|---|---|---|---|
| Session + cookie | Non — la gestion de cookies est peu naturelle pour un client mobile natif | Oui | Faible |
| **Jeton porteur (Bearer Token / API Token, retenu)** | Oui — un jeton se transmet simplement dans un en-tête HTTP, quel que soit le client | Oui | Faible à modérée |
| OAuth2 complet | Oui | Oui | Élevée — inutile pour une application actuellement mono-utilisateur (`README_DATABASE.md`, section 1.5) |

**Décision retenue : authentification par jeton porteur (*Bearer Token*), transmis dans l'en-tête `Authorization`.**

```
Authorization: Bearer <token>
```

### 3.2. Justification

Puisque Progress Tracker est conçu dès aujourd'hui pour être consommé par un client mobile natif (Flutter, `CAHIER_DES_CHARGES.md`) et par une PWA, un mécanisme fondé sur les cookies de session serait mal adapté au premier. Un jeton porteur, simple et *stateless*, convient aux deux à la fois, sans complexité inutile. OAuth2 est écarté en V1 car il résout un problème (déléguer l'accès entre plusieurs applications tierces et plusieurs utilisateurs) que Progress Tracker ne rencontre pas encore, tant que l'application reste mono-utilisateur.

### 3.3. Émission du jeton

En V1 (mono-utilisateur), un unique jeton longue durée est généré lors de l'installation de l'application (ex. via une commande `bin/generate-token.php`) et doit être renseigné manuellement dans chaque client (PWA, Flutter). Il n'existe pas, en V1, d'endpoint public `/login` : ce serait une complexité superflue tant qu'il n'y a qu'un seul utilisateur et qu'aucun mot de passe n'est géré par l'application. Ce point est identifié comme évolution nécessaire pour le passage au multi-utilisateur (section 14, cohérent avec `README_DATABASE.md` section 12.5).

### 3.4. Requêtes non authentifiées

Toute requête sans en-tête `Authorization` valide reçoit une réponse `401 Unauthorized` (section 7), interceptée par `AuthMiddleware` (`README_ARCHITECTURE.md`, section 14.2) avant même d'atteindre un Controller.

---

## 4. Format des requêtes et réponses

### 4.1. Type de contenu

Toutes les requêtes contenant un corps (`POST`, `PATCH`) doivent porter l'en-tête `Content-Type: application/json`, à l'exception de l'import de fichier (`POST /api/v1/imports`, section 11.2), qui utilise `multipart/form-data`. Toutes les réponses portent `Content-Type: application/json; charset=utf-8`.

### 4.2. Enveloppe de réponse

Toute réponse réussie est enveloppée dans un objet `data` :

```json
{
  "data": { "...": "..." }
}
```

Pour une liste de ressources, `data` est un tableau, accompagné d'un objet `meta` (section 10) :

```json
{
  "data": [ { "...": "..." }, { "...": "..." } ],
  "meta": { "...": "..." }
}
```

**Justification de cette enveloppe** (plutôt que de renvoyer directement l'objet ou le tableau à la racine de la réponse) : elle laisse la place, dès la V1, à des métadonnées transverses (pagination, avertissements non bloquants) sans qu'une évolution future ne casse la forme générale des réponses — un changement de structure de `data` resterait localisé, sans jamais transformer un objet racine en tableau ou inversement.

### 4.3. Dates

Toutes les dates échangées par l'API sont au format ISO 8601, en UTC, identique à la convention déjà posée dans `README_IMPORT_EXPORT.md` (section 8.2) et `README_DATABASE.md` (section 3.2) :

```json
"created_at": "2026-07-22T09:14:03Z"
```

---

## 5. Ressources et endpoints

### 5.1. Vue d'ensemble

| Ressource | Endpoint | Description |
|---|---|---|
| Tâches | `/tasks` | Créer, lister, consulter, modifier, archiver, restaurer une tâche |
| Événements | `/tasks/{uuid}/checks`, `/tasks/{uuid}/reset`, `/tasks/{uuid}/events` | Enregistrer une validation, un reset, consulter l'historique |
| Progression | `/tasks/{uuid}/progress` | Consulter la série de progression courante et les statistiques dérivées |
| Export | `/exports` | Générer et télécharger un fichier `.ptracker` |
| Import | `/imports` | Importer un fichier `.ptracker` |

### 5.2. Détail des endpoints — Tâches

| Méthode | URL | Rôle | Statuts possibles |
|---|---|---|---|
| `GET` | `/tasks` | Liste des tâches (paramètre `status` optionnel, voir 5.2.1) | `200` |
| `POST` | `/tasks` | Créer une tâche | `201`, `422` |
| `GET` | `/tasks/{uuid}` | Détail d'une tâche | `200`, `404` |
| `PATCH` | `/tasks/{uuid}` | Modifier une tâche (émet un événement `UPDATE_TASK`) | `200`, `404`, `422` |
| `POST` | `/tasks/{uuid}/archive` | Archiver une tâche (émet `ARCHIVE_TASK`) | `200`, `404`, `409` |
| `POST` | `/tasks/{uuid}/restore` | Restaurer une tâche archivée (émet `RESTORE_TASK`) | `200`, `404`, `409` |

#### 5.2.1. `GET /tasks` — filtrage

```
GET /api/v1/tasks?status=ACTIVE
```

Le paramètre `status` accepte `ACTIVE` ou `ARCHIVED` ; en son absence, toutes les tâches sont renvoyées, quel que soit leur statut.

#### 5.2.2. Pourquoi `archive` / `restore` sont des endpoints dédiés, plutôt qu'un simple `PATCH { "status": "ARCHIVED" }`

Une conception naïve laisserait `PATCH /tasks/{uuid}` accepter un changement de `status` parmi d'autres champs. Ce choix est délibérément écarté :

- l'archivage et la restauration sont des **événements métier significatifs** (`ARCHIVE_TASK`, `RESTORE_TASK`, `README_IMPORT_EXPORT.md` section 7), distincts d'une simple mise à jour de champ (`UPDATE_TASK`) ; leur donner un endpoint dédié rend cette distinction explicite dans l'API, et évite qu'un `PATCH` générique n'ait besoin d'une logique conditionnelle cachée selon les champs reçus ;
- cela permet d'appliquer des règles de transition d'état strictes (ex. `409 Conflict` si on tente d'archiver une tâche déjà archivée) sans ambiguïté sur l'intention de la requête.

### 5.3. Détail des endpoints — Événements

| Méthode | URL | Rôle | Statuts possibles |
|---|---|---|---|
| `POST` | `/tasks/{uuid}/checks` | Enregistrer une validation (`CHECK`) | `201`, `404`, `422` |
| `POST` | `/tasks/{uuid}/reset` | Interrompre volontairement la série en cours (`RESET`) | `201`, `404` |
| `GET` | `/tasks/{uuid}/events` | Historique complet des événements d'une tâche (paginé, section 10) | `200`, `404` |

> **Remarque :** il n'existe **aucun endpoint pour modifier ou supprimer un événement**, conformément à l'immuabilité du journal d'événements (`README_DATABASE.md`, section 3.5 ; `README_ARCHITECTURE.md`, section 6.4). Ce n'est pas un oubli : c'est une conséquence directe et volontaire de la philosophie du projet.

### 5.4. Détail de l'endpoint — Progression

| Méthode | URL | Rôle | Statuts possibles |
|---|---|---|---|
| `GET` | `/tasks/{uuid}/progress` | Série de progression courante, plus longue série historique, taux de complétion récent | `200`, `404` |

Conformément à `README_ARCHITECTURE.md` (section 15.3) et `README_DATABASE.md` (section 10.6), cet endpoint ne renvoie **aucune donnée stockée telle quelle** : chaque appel déclenche un recalcul à partir des événements bruts. Aucune mise en cache de ce résultat n'est effectuée en V1, la volumétrie d'un usage personnel restant largement compatible avec un recalcul à la demande (une piste d'optimisation est identifiée en section 14 si cela devait changer).

### 5.5. Détail des endpoints — Export / Import

Voir section 11.

---

## 6. Verbes HTTP

| Verbe | Usage dans cette API |
|---|---|
| `GET` | Lecture seule, jamais d'effet de bord. Toujours *idempotent* et *safe* au sens HTTP. |
| `POST` | Création d'une ressource (`POST /tasks`), ou déclenchement d'une action métier qui n'est pas une simple mise à jour de champ (`POST /tasks/{uuid}/checks`, `/archive`, `/restore`, `/reset`). |
| `PATCH` | Modification partielle d'une ressource existante (`PATCH /tasks/{uuid}`). `PUT` n'est volontairement pas utilisé : cette API ne propose jamais de remplacement intégral d'une ressource, seulement des modifications ciblées de champs. |
| `DELETE` | **Non utilisé en V1.** Conformément à `README_IMPORT_EXPORT.md` (section 7.7), la suppression physique d'une tâche reste une opération exceptionnelle, hors du périmètre normal de l'API V1. |

---

## 7. Statuts HTTP utilisés

| Code | Signification | Exemple dans Progress Tracker |
|---|---|---|
| `200 OK` | Lecture ou modification réussie | `GET /tasks/{uuid}`, `PATCH /tasks/{uuid}` |
| `201 Created` | Création réussie | `POST /tasks`, `POST /tasks/{uuid}/checks` |
| `204 No Content` | Action réussie, sans corps de réponse | Réservé pour de futures actions sans donnée à retourner (aucun endpoint V1 ne l'utilise actuellement, toutes les actions renvoient l'état résultant) |
| `400 Bad Request` | Corps de requête malformé (JSON invalide) | Corps JSON illisible |
| `401 Unauthorized` | Jeton d'authentification absent ou invalide | Toute requête sans en-tête `Authorization` valide |
| `404 Not Found` | Ressource inexistante | `GET /tasks/{uuid}` avec un `uuid` inconnu |
| `409 Conflict` | La requête est valide dans sa forme, mais entre en conflit avec l'état courant de la ressource | `POST /tasks/{uuid}/archive` sur une tâche déjà archivée |
| `422 Unprocessable Entity` | Corps de requête bien formé (JSON valide) mais invalide au sens métier | `POST /tasks` avec `daily_target: 0` |
| `500 Internal Server Error` | Erreur inattendue côté serveur | Exception non prévue, toujours loggée côté serveur, jamais détaillée au client (section 8.3) |

### 7.1. Pourquoi distinguer `400` et `422`

Cette distinction, souvent négligée, est appliquée strictement dans cette API :

- **`400`** signifie *"je ne peux même pas comprendre ce que vous m'avez envoyé"* (JSON syntaxiquement invalide).
- **`422`** signifie *"j'ai bien compris votre requête, mais son contenu ne respecte pas les règles attendues"* (ex. `daily_target` négatif).

Cette distinction reflète directement, côté HTTP, la séparation déjà posée dans `README_ARCHITECTURE.md` (section 10.1) entre validation de structure et validation métier.

---

## 8. Format des erreurs

### 8.1. Structure standard

Toute réponse d'erreur (tout statut ≥ 400) suit la même structure, produite de façon centralisée par `ErrorHandlerMiddleware` (`README_ARCHITECTURE.md`, section 14.3) :

```json
{
  "error": {
    "code": "VALIDATION_FAILED",
    "message": "La validation a échoué.",
    "details": {
      "daily_target": "L'objectif quotidien doit être au moins 1."
    }
  }
}
```

### 8.2. Table de correspondance exceptions → réponses HTTP

Cette table matérialise directement, côté API, la hiérarchie d'exceptions décrite dans `README_ARCHITECTURE.md` (section 9.2) :

| Exception PHP | Code HTTP | `error.code` |
|---|---|---|
| `TaskNotFoundException` | `404` | `TASK_NOT_FOUND` |
| `ValidationException` | `422` | `VALIDATION_FAILED` |
| `ConflictException` | `409` | `CONFLICT` |
| `ImportValidationException` | `422` | `IMPORT_VALIDATION_FAILED` |
| *(non authentifié)* | `401` | `UNAUTHENTICATED` |
| *(toute autre exception non prévue)* | `500` | `INTERNAL_ERROR` |

### 8.3. Ce qui n'apparaît jamais dans une réponse d'erreur

Pour toute erreur `500`, le message renvoyé au client reste générique (`"Une erreur inattendue est survenue."`) : **aucune trace de pile, aucun message d'exception PHP brut, aucun détail d'implémentation** n'est exposé publiquement, même en cas de bug. Le détail complet de l'erreur est loggé côté serveur pour le débogage, jamais transmis au client — une règle de sécurité de base qui évite de divulguer des informations sur l'infrastructure interne.

---

## 9. Validation

### 9.1. Où la validation est appliquée

Conformément à `README_ARCHITECTURE.md` (section 10.1), toute requête entrante passe par un Validator avant d'atteindre la logique métier du Service. Le format de l'erreur renvoyée (section 8.1) inclut un objet `details` donnant, champ par champ, le message d'erreur correspondant — jamais un message global impossible à relier à un champ précis d'un formulaire côté client.

### 9.2. Exemple de réponse de validation multi-champs

```json
{
  "error": {
    "code": "VALIDATION_FAILED",
    "message": "La validation a échoué.",
    "details": {
      "name": "Le nom de la tâche est obligatoire.",
      "target_days": "Jour invalide : XYZ."
    }
  }
}
```

---

## 10. Pagination

### 10.1. Comparaison des approches

| Approche | Principe | Avantages | Inconvénients |
|---|---|---|---|
| Pagination par décalage (*offset/limit*) | `?page=2&per_page=20` | Simple à comprendre et à implémenter ; permet d'accéder directement à une page arbitraire | Devient coûteuse en performance sur de très grandes tables (MySQL doit parcourir et ignorer toutes les lignes précédentes) ; instable si de nouvelles lignes sont insérées entre deux requêtes de pagination |
| **Pagination par curseur (*keyset pagination*, retenue pour `/events`)** | `?after=<curseur opaque>&limit=20` | Performance constante quelle que soit la profondeur de pagination, car elle s'appuie directement sur l'index `idx_events_task_occurred` (`README_DATABASE.md`, section 6.6) ; stable même si des événements sont insérés en cours de consultation | Ne permet pas d'accéder directement à une page arbitraire (seulement "la suite") — non gênant ici, l'historique étant consulté chronologiquement |

**Décision retenue : pagination par curseur pour `GET /tasks/{uuid}/events`.** C'est l'endpoint le plus exposé au risque de volumétrie importante (des années d'historique accumulé, exactement la préoccupation déjà identifiée dans `README_DATABASE.md`, section 12.1). Les autres listes de l'API (`GET /tasks`) restent de taille modeste par nature (un nombre de tâches suivies simultanément reste limité) et ne nécessitent donc **aucune pagination** en V1.

### 10.2. Exemple

```
GET /api/v1/tasks/{uuid}/events?limit=20
```

```json
{
  "data": [ { "...": "..." } ],
  "meta": {
    "next_cursor": "eyJvY2N1cnJlZF9hdCI6IjIwMjYtMDctMjBUMTg6MzI6MTBaIiwiaWQiOjQ4Mn0",
    "has_more": true
  }
}
```

```
GET /api/v1/tasks/{uuid}/events?after=eyJvY2N1cnJlZF9hdCI6IjIwMjYtMDctMjBUMTg6MzI6MTBaIiwiaWQiOjQ4Mn0&limit=20
```

Le curseur (`next_cursor`) est une valeur opaque pour le client (en pratique, un identifiant + un horodatage encodés) : le client ne doit jamais tenter de le construire ou de l'interpréter lui-même, seulement le renvoyer tel quel pour obtenir la page suivante.

---

## 11. Export et import

Ces deux endpoints exposent, via HTTP, exactement le processus déjà documenté dans `README_IMPORT_EXPORT.md`.

### 11.1. `GET /exports`

```
GET /api/v1/exports
```

Déclenche le processus d'export décrit dans `README_IMPORT_EXPORT.md` (section 11), et renvoie directement le fichier `ProgressTrackerExport.ptracker` en réponse, avec les en-têtes suivants :

```
Content-Type: application/zip
Content-Disposition: attachment; filename="ProgressTrackerExport.ptracker"
```

### 11.2. `POST /imports`

```
POST /api/v1/imports
Content-Type: multipart/form-data

file=@ProgressTrackerExport.ptracker
```

| Statut | Signification |
|---|---|
| `201 Created` | Import réussi ; le corps de la réponse résume le nombre de tâches et d'événements importés |
| `422 Unprocessable Entity` | Une étape de validation a échoué (`README_IMPORT_EXPORT.md`, section 9) ; `error.details` détaille la ligne et le fichier fautifs |
| `409 Conflict` | Des identifiants de l'export entrent en collision avec des données déjà présentes (`README_IMPORT_EXPORT.md`, section 10.1) |

**Réponse de succès :**

```json
{
  "data": {
    "import_id": "c4d8e2f1-...",
    "tasks_imported": 5,
    "events_imported": 812,
    "format_version": "1.0"
  }
}
```

**Réponse d'échec de validation :**

```json
{
  "error": {
    "code": "IMPORT_VALIDATION_FAILED",
    "message": "L'import a été annulé : le fichier contient des erreurs.",
    "details": {
      "events.csv:42": "task_id inconnu (absent de tasks.csv)."
    }
  }
}
```

Ce format reprend directement l'exemple de message d'erreur déjà présenté dans `README_IMPORT_EXPORT.md` (section 10.2), traduit ici dans la structure JSON standard de l'API.

---

## 12. Exemples complets de requêtes/réponses

### 12.1. Créer une tâche

**Requête**

```
POST /api/v1/tasks
Authorization: Bearer <token>
Content-Type: application/json

{
  "name": "Boire de l'eau",
  "description": "Objectif journalier",
  "daily_target": 8,
  "target_days": ["MON", "TUE", "WED", "THU", "FRI", "SAT", "SUN"]
}
```

**Réponse — `201 Created`**

```json
{
  "data": {
    "id": "a1e4d2f0-1111-4a2b-9c3d-000000000002",
    "name": "Boire de l'eau",
    "description": "Objectif journalier",
    "daily_target": 8,
    "target_days": ["MON", "TUE", "WED", "THU", "FRI", "SAT", "SUN"],
    "status": "ACTIVE",
    "created_at": "2026-07-22T09:14:03Z"
  }
}
```

### 12.2. Enregistrer une validation

**Requête**

```
POST /api/v1/tasks/a1e4d2f0-1111-4a2b-9c3d-000000000002/checks
Authorization: Bearer <token>
Content-Type: application/json

{
  "value": 2,
  "note": "2 verres au réveil"
}
```

**Réponse — `201 Created`**

```json
{
  "data": {
    "id": "e7c1a900-2222-4b3c-8d4e-000000000010",
    "task_id": "a1e4d2f0-1111-4a2b-9c3d-000000000002",
    "event_type": "CHECK",
    "occurred_at": "2026-07-22T07:05:00Z",
    "value": 2,
    "note": "2 verres au réveil"
  }
}
```

### 12.3. Consulter la progression

**Requête**

```
GET /api/v1/tasks/a1e4d2f0-1111-4a2b-9c3d-000000000001/progress
Authorization: Bearer <token>
```

**Réponse — `200 OK`**

```json
{
  "data": {
    "task_id": "a1e4d2f0-1111-4a2b-9c3d-000000000001",
    "current_streak_days": 4,
    "longest_streak_days": 12,
    "last_check_at": "2026-07-21T07:30:00Z",
    "completion_rate_last_30_days": 0.73
  }
}
```

### 12.4. Erreur — tâche introuvable

**Requête**

```
GET /api/v1/tasks/00000000-0000-0000-0000-000000000000
Authorization: Bearer <token>
```

**Réponse — `404 Not Found`**

```json
{
  "error": {
    "code": "TASK_NOT_FOUND",
    "message": "Aucune tâche trouvée pour l'identifiant 00000000-0000-0000-0000-000000000000."
  }
}
```

### 12.5. Archiver une tâche déjà archivée

**Requête**

```
POST /api/v1/tasks/a1e4d2f0-1111-4a2b-9c3d-000000000003/archive
Authorization: Bearer <token>
```

**Réponse — `409 Conflict`**

```json
{
  "error": {
    "code": "CONFLICT",
    "message": "Cette tâche est déjà archivée."
  }
}
```

---

## 13. Sécurité complémentaire

| Mesure | Description |
|---|---|
| **HTTPS obligatoire** | L'API ne doit jamais être exposée en HTTP simple, y compris en développement si des jetons réels sont utilisés. Le jeton d'authentification (section 3) transite en clair dans l'en-tête `Authorization` : sans HTTPS, il serait trivialement interceptable. |
| **Limitation de débit (*rate limiting*)** | Non implémentée en V1 (application mono-utilisateur, section 3.3), mais identifiée comme nécessaire dès l'ouverture de l'API à plusieurs utilisateurs (section 14), pour éviter qu'un client mal configuré (boucle infinie, bug de synchronisation) ne surcharge le serveur. |
| **CORS** | Géré par `CorsMiddleware` (`README_ARCHITECTURE.md`, section 14.2), avec une liste blanche explicite des origines autorisées (le domaine de la PWA), plutôt qu'un `Access-Control-Allow-Origin: *` permissif. |
| **Validation systématique côté serveur** | Même si un client (PWA, Flutter) valide déjà les données côté interface pour l'expérience utilisateur, **toute donnée est revalidée côté serveur** (`README_ARCHITECTURE.md`, section 10) — une validation uniquement côté client n'offre aucune garantie de sécurité ou d'intégrité. |

---

## 14. Évolutions futures

| Évolution | Description | Lien avec les autres documents |
|---|---|---|
| Authentification multi-utilisateur | Endpoint `/auth/login`, jetons à durée de vie limitée, rafraîchissement de jeton | Suppose l'ajout de la table `users` documentée dans `README_DATABASE.md`, section 12.5 |
| Limitation de débit par jeton | Protection contre les usages abusifs ou les bugs clients | Devient nécessaire dès que l'API n'est plus mono-utilisateur |
| Mise en cache du calcul de progression | Si le volume d'événements devait un jour rendre le recalcul à la demande (section 5.4) trop coûteux | Coordonné avec le partitionnement de `events` évoqué dans `README_DATABASE.md`, section 12.1 |
| Endpoints d'export partiel | Export d'une seule tâche ou d'une plage de dates | Correspond exactement à l'évolution déjà anticipée dans `README_IMPORT_EXPORT.md`, section 14 |
| `DELETE /tasks/{uuid}` | Suppression physique explicite, cas d'usage RGPD | À n'introduire qu'en cohérence stricte avec `README_IMPORT_EXPORT.md`, section 7.7, qui documente déjà la prudence requise autour de `DELETE_TASK` |

---

*Fin du document — `README_API.md`, cohérent avec `README_ARCHITECTURE.md`, `README_DATABASE.md` et `README_IMPORT_EXPORT.md`.*
