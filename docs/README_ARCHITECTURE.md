# README_ARCHITECTURE.md — Architecture logicielle

**Projet :** Progress Tracker
**Public visé :** développeur découvrant progressivement l'architecture — aucune connaissance préalable du projet n'est supposée, si ce n'est la lecture de `CAHIER_DES_CHARGES.md`.
**Statut du document :** spécification de référence

Ce document explique **comment** le code de Progress Tracker est organisé, et surtout **pourquoi** il l'est de cette façon. Il est cohérent avec `README_DATABASE.md` (schéma MySQL) et `README_IMPORT_EXPORT.md` (format `.ptracker`), et sert de base à `README_API.md`.

---

## Sommaire

1. Philosophie générale
2. Vue d'ensemble des couches
3. Arborescence complète du projet
4. Le dossier `Controllers`
5. Le dossier `Services`
6. Le dossier `Repositories`
7. Le dossier `Domain`
8. Le dossier `DTO`
9. Le dossier `Exceptions`
10. Le dossier `Validators`
11. Le dossier `Database`
12. Le dossier `Config`
13. Le dossier `Helpers`
14. Le dossier `Http` (routage et middlewares)
15. Scénarios complets
16. Pourquoi cette architecture prépare l'API REST, la PWA et Flutter
17. Résumé des règles à ne jamais enfreindre

---

## 1. Philosophie générale

### 1.1. Pourquoi séparer les responsabilités

Un développeur débutant écrit souvent, dans un premier temps, tout son code au même endroit : un fichier PHP qui reçoit la requête HTTP, interroge directement la base de données avec du SQL en dur, applique quelques règles métier au milieu, et affiche le résultat. Ce style fonctionne pour un petit script, mais devient rapidement un problème dès qu'un projet doit évoluer sur plusieurs années — exactement l'ambition affichée pour Progress Tracker.

Le principe de **séparation des responsabilités** (*Separation of Concerns*) consiste à donner à chaque partie du code **une seule raison de changer**. Concrètement :

- si l'interface utilisateur change (nouveau design, nouvelle page), cela ne doit toucher **que** la couche Interface ;
- si une règle métier change (ex. la définition d'une série de progression valide), cela ne doit toucher **que** la couche Services ;
- si on change de moteur de base de données ou de schéma, cela ne doit toucher **que** la couche Repositories.

Sans cette séparation, une modification anodine dans un coin du projet peut avoir des effets de bord imprévisibles ailleurs — un risque que ce document cherche explicitement à éliminer par construction.

### 1.2. Pourquoi une architecture en couches

Une **architecture en couches** organise le code en niveaux empilés, où chaque couche ne communique qu'avec la couche immédiatement voisine, dans un sens précis :

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

C'est une image utile mais qu'il faut préciser tout de suite : cette architecture n'est pas une simple suite d'appels de fonctions. C'est surtout une **règle de dépendance** :

> **Une couche ne doit connaître que la couche immédiatement inférieure. Elle ne doit jamais connaître ce qui se trouve au-dessus d'elle, ni sauter une couche.**

Ainsi, un Controller peut appeler un Service, mais un Service ne doit **jamais** connaître l'existence d'un Controller, ni de la requête HTTP qui a déclenché son exécution. C'est cette règle, plus que le schéma en escalier lui-même, qui garantit que la logique métier de Progress Tracker restera réutilisable telle quelle par une future API REST, puis par une application Flutter (section 16).

### 1.3. Pourquoi les Controllers ne doivent jamais contenir de logique métier

C'est l'erreur de conception la plus fréquente chez un développeur qui découvre les architectures en couches : écrire les règles métier *dans* le Controller, parce que "c'est là que la requête arrive".

Ce document interdit explicitement cette pratique, pour trois raisons :

1. **Un Controller est lié à un protocole de transport (HTTP).** Si la logique métier y est écrite, elle devient, de fait, indissociable de HTTP. Impossible alors de la réutiliser depuis une commande en ligne, une tâche planifiée (cron), ou des tests automatisés qui n'ont pas besoin de simuler une requête HTTP complète.
2. **Un Controller est difficile à tester unitairement.** Tester une règle métier nichée dans un Controller oblige à simuler toute la mécanique HTTP (requête, session, en-têtes) juste pour vérifier une règle de calcul qui n'a, en réalité, rien à voir avec HTTP.
3. **Cela viole directement la philosophie du projet.** `CAHIER_DES_CHARGES.md` exige que la logique métier reste indépendante des interfaces, des contrôleurs, des requêtes HTTP et de la base de données. Un Controller "gras" (*fat controller*) qui contient des règles métier est une violation directe de cette exigence, pas un simple détail de style.

**Règle retenue : un Controller ne fait que trois choses.** Il traduit une requête HTTP en un appel à un Service (via un DTO, section 8), il transmet le résultat du Service à la couche de présentation (vue HTML ou réponse JSON), et il traduit les exceptions métier en réponses HTTP appropriées (codes d'erreur, messages). Rien de plus.

---

## 2. Vue d'ensemble des couches

| Couche | Dossier | Connaît | Ne doit jamais connaître |
|---|---|---|---|
| Interface | `public/`, templates, JS front | Controllers (via routes HTTP) | Services, Repositories, SQL |
| Controllers | `src/Controllers/` | Services, DTO, Exceptions | SQL, PDO, structure des tables |
| Services | `src/Services/` | Repositories (via interfaces), Domain, Exceptions | HTTP, Controllers, PDO, SQL |
| Repositories | `src/Repositories/` | Database (PDO), Domain | Services, Controllers, HTTP |
| Database | `src/Database/` | Connexion PDO, configuration | Logique métier, HTTP |
| MySQL | — (serveur externe) | — | Tout le reste |

Cette table est la référence à consulter en cas de doute : "cette classe a-t-elle le droit de connaître X ?"

### 2.1. Le rôle central des Services

Comme rappelé dans `CAHIER_DES_CHARGES.md`, **les Services constituent le cœur de l'application**. Toutes les autres couches existent pour les servir :

- les Controllers **adaptent** le monde extérieur (HTTP) pour parler aux Services ;
- les Repositories **adaptent** le monde de la persistance (MySQL) pour parler aux Services ;
- les Services, eux, ne s'adaptent à rien : ils contiennent la vérité métier de Progress Tracker, exprimée en PHP pur, sans dépendance technique.

```
                    ┌─────────────────────┐
   HTTP  ──────────▶│     Controllers        │
                    └──────────┬──────────┘
                               │ appelle
                               ▼
                    ┌─────────────────────┐
                    │      Services           │◀── cœur de l'application
                    └──────────┬──────────┘     (logique métier pure)
                               │ appelle (via interface)
                               ▼
                    ┌─────────────────────┐
                    │    Repositories         │
                    └──────────┬──────────┘
                               │ requête SQL via PDO
                               ▼
                          MySQL
```

### 2.2. Interfaces de Repository : pourquoi les Services ne dépendent-ils pas directement de PDO

Les Services ne manipulent jamais directement une classe `TaskRepository` concrète, mais une **interface** `TaskRepositoryInterface`. C'est une application du **principe d'inversion de dépendance** (le "D" de SOLID) :

```php
interface TaskRepositoryInterface
{
    public function findByUuid(string $uuid): ?Task;
    public function findAllActive(): array;
    public function save(Task $task): void;
}
```

Le Service dépend de ce contrat abstrait, jamais de son implémentation MySQL concrète. Cela apporte deux bénéfices concrets pour ce projet :

1. **Testabilité :** dans les tests unitaires d'un Service, on peut fournir une implémentation en mémoire (`InMemoryTaskRepository`) de cette même interface, sans jamais toucher à une vraie base MySQL. Les tests deviennent rapides et fiables.
2. **Évolutivité :** si Progress Tracker devait un jour changer de moteur de stockage (peu probable, mais l'architecture ne doit pas l'interdire), seule une nouvelle implémentation de l'interface serait nécessaire — aucun Service ne serait modifié.

---

## 3. Arborescence complète du projet

```
progress-tracker/
├── public/
│   ├── index.php                      # Front controller de l'application web
│   └── api.php                        # Front controller de l'API REST (voir README_API.md)
│
├── src/
│   ├── Controllers/
│   │   ├── TaskController.php
│   │   ├── EventController.php
│   │   ├── ExportController.php
│   │   └── ImportController.php
│   │
│   ├── Services/
│   │   ├── TaskService.php
│   │   ├── EventService.php
│   │   ├── ProgressCalculationService.php
│   │   ├── ExportService.php
│   │   └── ImportService.php
│   │
│   ├── Repositories/
│   │   ├── TaskRepositoryInterface.php
│   │   ├── TaskRepository.php
│   │   ├── EventRepositoryInterface.php
│   │   └── EventRepository.php
│   │
│   ├── Domain/
│   │   ├── Task.php
│   │   ├── Event.php
│   │   ├── EventType.php              # Enum PHP 8.1+
│   │   └── DayOfWeek.php              # Enum PHP 8.1+
│   │
│   ├── DTO/
│   │   ├── Request/
│   │   │   ├── CreateTaskRequest.php
│   │   │   ├── UpdateTaskRequest.php
│   │   │   └── CreateCheckRequest.php
│   │   └── Response/
│   │       ├── TaskResponse.php
│   │       ├── EventResponse.php
│   │       └── ProgressResponse.php
│   │
│   ├── Exceptions/
│   │   ├── DomainException.php
│   │   ├── TaskNotFoundException.php
│   │   ├── ValidationException.php
│   │   └── ImportValidationException.php
│   │
│   ├── Validators/
│   │   ├── TaskValidator.php
│   │   └── CheckValidator.php
│   │
│   ├── Database/
│   │   └── Connection.php             # Enveloppe autour de PDO
│   │
│   ├── Http/
│   │   ├── Router.php
│   │   ├── Request.php
│   │   ├── Response.php
│   │   └── Middlewares/
│   │       ├── ErrorHandlerMiddleware.php
│   │       ├── JsonBodyParserMiddleware.php
│   │       ├── AuthMiddleware.php
│   │       └── CorsMiddleware.php
│   │
│   ├── Config/
│   │   └── config.php
│   │
│   └── Helpers/
│       ├── UuidGenerator.php
│       └── ClockInterface.php
│
├── migrations/
│   ├── 20260115090000_create_tasks_table.sql
│   ├── 20260115090100_create_task_target_days_table.sql
│   └── 20260115090200_create_events_table.sql
│
├── bin/
│   └── migrate.php                    # Exécute les migrations en attente
│
├── tests/
│   ├── Unit/
│   │   └── Services/
│   └── Integration/
│       └── Repositories/
│
├── docs/
│   ├── README.md
│   ├── CAHIER_DES_CHARGES.md
│   ├── README_ARCHITECTURE.md
│   ├── README_DATABASE.md
│   ├── README_IMPORT_EXPORT.md
│   └── README_API.md
│
├── composer.json
└── .env.example
```

### 3.1. Espace de noms (PSR-4)

Tout le code de `src/` est déclaré sous l'espace de noms racine `ProgressTracker\`, avec une correspondance directe entre chemin de dossier et sous-espace de noms (PSR-4 standard) :

```json
{
    "autoload": {
        "psr-4": {
            "ProgressTracker\\": "src/"
        }
    }
}
```

Ainsi, `src/Services/TaskService.php` déclare `namespace ProgressTracker\Services;`. Ce nommage explicite (plutôt qu'un générique `App\`) est un choix délibéré : si ce code devait un jour être publié comme bibliothèque indépendante (par exemple, un module de calcul de séries de progression réutilisable), l'espace de noms resterait sans ambiguïté.

### 3.2. Pourquoi pas de framework complet (Laravel, Symfony)

| Approche | Avantages | Inconvénients pour ce projet |
|---|---|---|
| Framework complet (Laravel, Symfony) | Productivité immédiate, écosystème riche, ORM intégré | Masque une grande partie des mécanismes internes derrière des conventions "magiques" ; l'objectif pédagogique explicite du projet (comprendre l'architecture en profondeur) serait contrarié par une couche d'abstraction supplémentaire déjà résolue par le framework |
| **Micro-architecture faite main (retenu)** | Chaque couche est explicite et comprise dans son intégralité ; aucune dépendance lourde ; contrôle total sur les choix (identifiants, immuabilité des événements, etc.) | Certaines briques usuelles d'un framework (routage, validation, injection de dépendances) doivent être écrites ou choisies indépendamment |

Ce choix est cohérent avec l'objectif énoncé dans `CAHIER_DES_CHARGES.md` : produire une architecture propre et professionnelle **dans un but d'apprentissage de la conception logicielle**. Un framework complet résoudrait la plupart des problèmes que ce document prend justement le temps d'expliquer.

Pour le routage HTTP seul (associer une URL à un Controller), une bibliothèque légère et non intrusive (par exemple `nikic/FastRoute`) peut être utilisée sans contredire ce principe : elle ne dicte aucune convention d'architecture, elle résout uniquement un problème mécanique bien délimité (faire correspondre une URL à une fonction).

---

## 4. Le dossier `Controllers`

### 4.1. Rôle

Un Controller est un **traducteur**, dans les deux sens :

- il traduit une requête HTTP entrante en un appel de méthode de Service, via un DTO de requête (section 8) ;
- il traduit le résultat renvoyé par le Service (un objet Domain, ou une exception) en une réponse HTTP compréhensible (JSON pour l'API, HTML pour l'interface web).

### 4.2. Ce qu'un Controller ne fait jamais

- il n'exécute **aucune requête SQL**, directement ou indirectement ;
- il n'implémente **aucune règle métier** ("est-ce que cette série est valide ?", "quel est le prochain état d'une tâche archivée ?") ;
- il ne construit **aucun objet Domain complexe** lui-même — il délègue cette responsabilité au Service, qui seul connaît les règles de construction valides.

### 4.3. Exemple : `TaskController`

```php
namespace ProgressTracker\Controllers;

use ProgressTracker\Services\TaskService;
use ProgressTracker\DTO\Request\CreateTaskRequest;
use ProgressTracker\Exceptions\ValidationException;
use ProgressTracker\Http\Request;
use ProgressTracker\Http\Response;

final class TaskController
{
    public function __construct(private readonly TaskService $taskService)
    {
    }

    public function store(Request $request): Response
    {
        try {
            $dto = CreateTaskRequest::fromArray($request->jsonBody());
            $task = $this->taskService->createTask($dto);

            return Response::json(TaskResponse::fromDomain($task), 201);
        } catch (ValidationException $e) {
            return Response::json(['errors' => $e->getErrors()], 422);
        }
    }
}
```

Remarquer ce que ce Controller **ne fait pas** : il ne connaît pas la structure de la table `tasks`, il ne sait pas qu'un événement `CREATE_TASK` doit être inséré en parallèle (`README_DATABASE.md`, section 10.1) — tout cela est de la responsabilité du `TaskService`.

---

## 5. Le dossier `Services`

### 5.1. Rôle

Les Services contiennent **toute** la logique métier de Progress Tracker. Concrètement, c'est ici que sont implémentées les règles décrites dans `CAHIER_DES_CHARGES.md` et `README_DATABASE.md` (section 8.2) :

- créer une tâche **et** son événement `CREATE_TASK` de façon atomique ;
- calculer une série de progression à partir d'une liste brute d'événements (jamais en SQL, `README_DATABASE.md` section 10.6) ;
- décider si un `RESET` est nécessaire, si un archivage est autorisé, etc.

### 5.2. Caractéristique essentielle : l'indépendance technique

Un Service ne doit **jamais** importer de classe liée à HTTP (`Request`, `Response`) ni à PDO. Il ne dépend que :

- d'interfaces de Repository (section 2.2) ;
- de classes du dossier `Domain` (section 7) ;
- d'autres Services, si nécessaire (ex. `ExportService` utilise `EventService` en interne).

### 5.3. Exemple : `TaskService::createTask()`

```php
namespace ProgressTracker\Services;

use ProgressTracker\Repositories\TaskRepositoryInterface;
use ProgressTracker\Repositories\EventRepositoryInterface;
use ProgressTracker\Domain\Task;
use ProgressTracker\Domain\Event;
use ProgressTracker\Domain\EventType;
use ProgressTracker\DTO\Request\CreateTaskRequest;
use ProgressTracker\Validators\TaskValidator;
use ProgressTracker\Helpers\UuidGenerator;
use ProgressTracker\Helpers\ClockInterface;

final class TaskService
{
    public function __construct(
        private readonly TaskRepositoryInterface $taskRepository,
        private readonly EventRepositoryInterface $eventRepository,
        private readonly TaskValidator $validator,
        private readonly UuidGenerator $uuidGenerator,
        private readonly ClockInterface $clock,
    ) {
    }

    public function createTask(CreateTaskRequest $dto): Task
    {
        $this->validator->validateForCreation($dto);

        $task = new Task(
            uuid: $this->uuidGenerator->generate(),
            name: $dto->name,
            description: $dto->description,
            dailyTarget: $dto->dailyTarget,
            targetDays: $dto->targetDays,
            status: 'ACTIVE',
            createdAt: $this->clock->nowUtc(),
        );

        // La création de la tâche et de son événement CREATE_TASK
        // doivent réussir ou échouer ensemble (README_DATABASE.md, section 10.1).
        // C'est le Repository qui encapsule la transaction, voir section 6.3.
        $this->taskRepository->saveWithCreationEvent(
            $task,
            new Event(
                uuid: $this->uuidGenerator->generate(),
                taskId: $task->uuid,
                type: EventType::CREATE_TASK,
                occurredAt: $this->clock->nowUtc(),
            ),
        );

        return $task;
    }
}
```

> **Remarque sur `ClockInterface` :** l'heure courante n'est jamais lue directement via `new DateTime()` dans un Service. Elle passe par une interface injectée (`ClockInterface::nowUtc()`), pour la même raison que les Repositories sont abstraits par une interface (section 2.2) : cela permet de "figer" le temps dans les tests unitaires, en injectant une horloge factice qui renvoie toujours une date fixe et connue.

### 5.4. `ProgressCalculationService` : le calcul des séries de progression

C'est le Service qui matérialise directement la philosophie centrale du projet (voir `README_DATABASE.md`, section 10.6) :

```php
final class ProgressCalculationService
{
    public function __construct(
        private readonly EventRepositoryInterface $eventRepository,
    ) {
    }

    public function computeCurrentStreak(string $taskUuid): int
    {
        $events = $this->eventRepository->findByTaskUuidOrderedByDate($taskUuid);

        // Algorithme pur, ne dépendant que de la liste d'événements en entrée.
        // Aucune requête SQL n'est exécutée ici : les données sont déjà en mémoire.
        return $this->replayEventsIntoStreak($events);
    }

    private function replayEventsIntoStreak(array $events): int
    {
        // Parcourt les événements chronologiquement :
        // - un CHECK qui complète l'objectif du jour prolonge la série ;
        // - un RESET remet la série à zéro ;
        // - un jour actif sans CHECK complet interrompt la série.
        // (implémentation détaillée hors du périmètre de ce document d'architecture)
        // ...
    }
}
```

Le point essentiel : cette méthode ne dépend **que** de la liste d'événements qu'on lui donne en entrée. Elle pourrait aussi bien recevoir des événements lus depuis un fichier `events.csv` importé (`README_IMPORT_EXPORT.md`) que depuis MySQL — c'est précisément ce qui la rendra réutilisable telle quelle (ou portable en Dart) pour un futur mode hors-ligne dans l'application Flutter.

---

## 6. Le dossier `Repositories`

### 6.1. Rôle

Les Repositories sont la **seule** couche du projet autorisée à écrire du SQL ou à manipuler PDO directement. Ils traduisent entre le monde relationnel (lignes de tables `tasks`, `events`, `task_target_days`, décrites dans `README_DATABASE.md`) et le monde objet (`Task`, `Event` du dossier `Domain`, section 7).

### 6.2. Ce qu'un Repository ne fait jamais

- il n'implémente **aucune règle métier** — pas même une règle qui semblerait "naturelle" à mettre en SQL (ex. calculer une série de progression, voir `README_DATABASE.md` section 10.6, qui explique pourquoi ce calcul reste hors des Repositories) ;
- il ne connaît **rien** de HTTP ni des Controllers.

### 6.3. Exemple : `TaskRepository`

```php
namespace ProgressTracker\Repositories;

use ProgressTracker\Database\Connection;
use ProgressTracker\Domain\Task;
use ProgressTracker\Domain\Event;

final class TaskRepository implements TaskRepositoryInterface
{
    public function __construct(private readonly Connection $connection)
    {
    }

    public function saveWithCreationEvent(Task $task, Event $creationEvent): void
    {
        $pdo = $this->connection->pdo();
        $pdo->beginTransaction();

        try {
            $stmt = $pdo->prepare(
                'INSERT INTO tasks (uuid, name, description, daily_target, status, created_at)
                 VALUES (:uuid, :name, :description, :daily_target, :status, :created_at)'
            );
            $stmt->execute([
                'uuid'         => $task->uuid,
                'name'         => $task->name,
                'description'  => $task->description,
                'daily_target' => $task->dailyTarget,
                'status'       => $task->status,
                'created_at'   => $task->createdAt->format('Y-m-d H:i:s'),
            ]);

            $taskId = (int) $pdo->lastInsertId();
            $this->insertTargetDays($pdo, $taskId, $task->targetDays);
            $this->insertEvent($pdo, $taskId, $creationEvent);

            $pdo->commit();
        } catch (\Throwable $e) {
            $pdo->rollBack();
            throw $e;
        }
    }

    // ... insertTargetDays(), insertEvent(), findByUuid(), findAllActive() ...
}
```

Ce code met en œuvre directement la transaction décrite dans `README_DATABASE.md` (section 10.1) : la gestion de la transaction MySQL est une responsabilité du Repository, jamais du Service qui l'appelle — le Service ne sait même pas qu'une transaction a eu lieu, il sait seulement que "sauvegarder une tâche avec son événement de création" est une opération atomique du point de vue métier.

### 6.4. `EventRepository` : une interface volontairement restreinte

Conformément à la règle d'immuabilité posée dans `README_DATABASE.md` (section 3.5), l'interface `EventRepositoryInterface` **n'expose aucune méthode `update()` ni `delete()`** :

```php
interface EventRepositoryInterface
{
    public function insert(Event $event): void;
    public function findByTaskUuidOrderedByDate(string $taskUuid): array;
    public function findRecent(int $limit): array;
}
```

C'est une protection architecturale à part entière : même un développeur pressé ou distrait ne peut pas modifier un événement existant, tout simplement parce que **la méthode pour le faire n'existe pas**. Cette garantie s'ajoute (elle ne remplace pas) à la restriction de privilèges MySQL déjà recommandée dans `README_DATABASE.md`.

---

## 7. Le dossier `Domain`

### 7.1. Rôle

Le dossier `Domain` contient les objets métier "purs" de Progress Tracker : `Task`, `Event`, ainsi que les énumérations `EventType` et `DayOfWeek`. Ce sont des objets simples (parfois appelés *Plain Old PHP Objects*), sans aucune dépendance vers PDO, HTTP, ou toute autre couche technique.

### 7.2. Distinction avec les DTO (aperçu, détaillé en section 8)

Une confusion fréquente chez un développeur débutant est de se demander pourquoi `Domain\Task` et `DTO\Response\TaskResponse` sont deux classes différentes, alors qu'elles se ressemblent. La réponse tient en une phrase : **`Domain\Task` représente une tâche telle que la comprend le métier ; `DTO\Response\TaskResponse` représente une tâche telle qu'elle doit être exposée à l'extérieur (JSON de l'API, par exemple).** Ces deux représentations peuvent diverger (ex. un champ interne utile au calcul mais non pertinent pour l'API), et les faire dépendre l'une de l'autre créerait un couplage indésirable entre le métier et le format d'exposition externe.

### 7.3. Exemple : `EventType` (enum PHP 8.1+)

```php
namespace ProgressTracker\Domain;

enum EventType: string
{
    case CREATE_TASK   = 'CREATE_TASK';
    case CHECK         = 'CHECK';
    case RESET         = 'RESET';
    case ARCHIVE_TASK  = 'ARCHIVE_TASK';
    case RESTORE_TASK  = 'RESTORE_TASK';
    case UPDATE_TASK   = 'UPDATE_TASK';
    case DELETE_TASK   = 'DELETE_TASK';
}
```

Cette énumération reprend exactement les sept valeurs déjà définies dans `README_IMPORT_EXPORT.md` (section 7) et `README_DATABASE.md` (section 6.2, colonne `event_type`). L'utilisation du type natif `enum` de PHP (plutôt que de simples constantes de classe ou de chaînes libres) permet au moteur PHP lui-même de refuser toute valeur invalide dès la compilation, avant même qu'une validation applicative n'intervienne.

---

## 8. Le dossier `DTO`

### 8.1. Pourquoi des DTO (*Data Transfer Objects*)

Un DTO est un objet dont le seul rôle est de **transporter des données entre deux couches**, sans comportement métier. Deux catégories sont distinguées dans Progress Tracker :

- **DTO de requête** (`DTO/Request/`) : représentent les données entrantes, telles qu'un Controller les reçoit (ex. `CreateTaskRequest`), avant toute validation ni transformation en objet `Domain`.
- **DTO de réponse** (`DTO/Response/`) : représentent les données sortantes, telles qu'elles doivent être exposées à l'extérieur (ex. `TaskResponse`), construites à partir d'un objet `Domain`.

### 8.2. Pourquoi ne pas exposer directement les objets `Domain` à l'extérieur

Il serait tentant de renvoyer directement un objet `Domain\Task` en JSON depuis un Controller. Ce raccourci est déconseillé pour trois raisons :

1. **Stabilité de l'API :** si `Domain\Task` gagne un jour un champ interne (ex. un indicateur technique de cache), ce champ apparaîtrait automatiquement dans les réponses de l'API sans qu'on l'ait décidé explicitement. Un `TaskResponse` explicite agit comme un **contrat stable**, découplé des détails internes du métier.
2. **Sur-exposition évitée (*over-fetching*) :** certains champs internes (par exemple un identifiant interne `BIGINT`, voir `README_DATABASE.md` section 3.1) ne doivent **jamais** être exposés publiquement. Le DTO de réponse est l'endroit exact où cette règle est appliquée une fois pour toutes.
3. **Validation d'entrée centralisée :** un DTO de requête matérialise précisément la forme attendue d'une entrée, avant même que le Validator (section 10) n'entre en jeu — il documente, par sa simple existence dans le code, le contrat d'entrée d'une opération.

### 8.3. Exemple

```php
namespace ProgressTracker\DTO\Request;

final class CreateTaskRequest
{
    public function __construct(
        public readonly string $name,
        public readonly ?string $description,
        public readonly int $dailyTarget,
        public readonly array $targetDays,
    ) {
    }

    public static function fromArray(array $data): self
    {
        return new self(
            name: $data['name'] ?? '',
            description: $data['description'] ?? null,
            dailyTarget: (int) ($data['daily_target'] ?? 1),
            targetDays: $data['target_days'] ?? [],
        );
    }
}
```

```php
namespace ProgressTracker\DTO\Response;

use ProgressTracker\Domain\Task;

final class TaskResponse
{
    public static function fromDomain(Task $task): array
    {
        return [
            'id'           => $task->uuid,           // identifiant public uniquement (README_DATABASE.md, 3.1)
            'name'         => $task->name,
            'description'  => $task->description,
            'daily_target' => $task->dailyTarget,
            'target_days'  => $task->targetDays,
            'status'       => $task->status,
            'created_at'   => $task->createdAt->format(DATE_ATOM),
        ];
    }
}
```

---

## 9. Le dossier `Exceptions`

### 9.1. Rôle

Progress Tracker utilise une hiérarchie d'exceptions dédiée, plutôt que des exceptions PHP génériques (`\Exception`, `\RuntimeException`), pour permettre à la couche Controllers de traduire précisément chaque cas d'erreur métier en une réponse HTTP appropriée (voir `README_API.md` pour la correspondance complète avec les codes de statut HTTP).

### 9.2. Hiérarchie

```
\Throwable
  └── \Exception
        └── DomainException (abstraite, base commune)
              ├── TaskNotFoundException
              ├── ValidationException
              ├── ImportValidationException
              └── ConflictException
```

```php
namespace ProgressTracker\Exceptions;

abstract class DomainException extends \Exception
{
}

final class TaskNotFoundException extends DomainException
{
    public function __construct(string $uuid)
    {
        parent::__construct("Aucune tâche trouvée pour l'identifiant {$uuid}.");
    }
}

final class ValidationException extends DomainException
{
    /** @param array<string, string> $errors */
    public function __construct(private readonly array $errors)
    {
        parent::__construct('La validation a échoué.');
    }

    public function getErrors(): array
    {
        return $this->errors;
    }
}
```

### 9.3. Où ces exceptions sont levées et attrapées

- **Levées** : dans les Services (règles métier) et les Validators (règles de format) — jamais dans les Repositories, qui doivent rester agnostiques du sens métier des erreurs qu'ils peuvent rencontrer (une contrainte SQL violée est traduite, si nécessaire, en exception de plus haut niveau par le Service appelant).
- **Attrapées** : dans les Controllers (pour construire une réponse HTTP précise) ou, pour ne pas répéter cette logique dans chaque Controller, dans un `ErrorHandlerMiddleware` central (section 14.3).

---

## 10. Le dossier `Validators`

### 10.1. Deux niveaux de validation, deux responsabilités distinctes

Progress Tracker distingue explicitement deux types de validation, pour éviter la confusion fréquente entre "la donnée est-elle bien formée ?" et "la donnée est-elle acceptable pour le métier ?" :

| Niveau | Exemple | Où |
|---|---|---|
| **Validation de structure/format** | `daily_target` est-il bien un entier positif ? `target_days` ne contient-il que des valeurs parmi `MON..SUN` ? | `Validators/` |
| **Validation métier** | Peut-on vraiment archiver cette tâche maintenant ? Un `RESET` a-t-il un sens sur une tâche qui vient d'être créée ? | `Services/` |

### 10.2. Exemple : `TaskValidator`

```php
namespace ProgressTracker\Validators;

use ProgressTracker\DTO\Request\CreateTaskRequest;
use ProgressTracker\Exceptions\ValidationException;

final class TaskValidator
{
    public function validateForCreation(CreateTaskRequest $dto): void
    {
        $errors = [];

        if (trim($dto->name) === '') {
            $errors['name'] = 'Le nom de la tâche est obligatoire.';
        }

        if ($dto->dailyTarget < 1) {
            $errors['daily_target'] = 'L\'objectif quotidien doit être au moins 1.';
        }

        $validDays = ['MON', 'TUE', 'WED', 'THU', 'FRI', 'SAT', 'SUN'];
        foreach ($dto->targetDays as $day) {
            if (!in_array($day, $validDays, true)) {
                $errors['target_days'] = "Jour invalide : {$day}.";
                break;
            }
        }

        if ($errors !== []) {
            throw new ValidationException($errors);
        }
    }
}
```

Ce Validator est appelé **depuis le Service** (voir section 5.3, `TaskService::createTask()`), pas depuis le Controller. Ce choix garantit que la validation s'applique quel que soit l'appelant du Service — y compris un futur script en ligne de commande ou un job de synchronisation, qui n'auraient pas transité par un Controller HTTP.

---

## 11. Le dossier `Database`

### 11.1. Rôle

Contient exclusivement `Connection.php`, une fine enveloppe autour de PDO, responsable de :

- construire le DSN à partir de la configuration (section 12) ;
- ouvrir la connexion PDO avec les bons attributs (`PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION`, encodage `utf8mb4`, cohérent avec `README_DATABASE.md` section 3.4) ;
- exposer une méthode `pdo(): PDO` utilisée uniquement par les Repositories.

```php
namespace ProgressTracker\Database;

final class Connection
{
    private ?\PDO $pdo = null;

    public function __construct(private readonly array $config)
    {
    }

    public function pdo(): \PDO
    {
        if ($this->pdo === null) {
            $dsn = sprintf(
                'mysql:host=%s;dbname=%s;charset=utf8mb4',
                $this->config['host'],
                $this->config['database'],
            );

            $this->pdo = new \PDO($dsn, $this->config['username'], $this->config['password'], [
                \PDO::ATTR_ERRMODE            => \PDO::ERRMODE_EXCEPTION,
                \PDO::ATTR_DEFAULT_FETCH_MODE => \PDO::FETCH_ASSOC,
                \PDO::ATTR_EMULATE_PREPARES   => false,
            ]);
        }

        return $this->pdo;
    }
}
```

### 11.2. Le dossier `migrations/` et `bin/migrate.php`

Les fichiers SQL de `migrations/` correspondent directement à la table `schema_migrations` décrite dans `README_DATABASE.md` (section 11). Le script `bin/migrate.php` :

1. lit la table `schema_migrations` pour connaître les migrations déjà appliquées ;
2. compare avec les fichiers présents dans `migrations/` ;
3. exécute, dans l'ordre chronologique, les migrations manquantes, chacune dans sa propre transaction ;
4. insère une ligne dans `schema_migrations` pour chaque migration appliquée avec succès.

---

## 12. Le dossier `Config`

`Config/config.php` centralise toute configuration dépendant de l'environnement (connexion base de données, mode debug, etc.), elle-même lue depuis des variables d'environnement (fichier `.env`, jamais commité — seul `.env.example` l'est, comme référence des clés attendues).

```php
return [
    'database' => [
        'host'     => $_ENV['DB_HOST'] ?? 'localhost',
        'database' => $_ENV['DB_NAME'] ?? 'progress_tracker',
        'username' => $_ENV['DB_USER'] ?? '',
        'password' => $_ENV['DB_PASSWORD'] ?? '',
    ],
    'app' => [
        'debug'    => filter_var($_ENV['APP_DEBUG'] ?? false, FILTER_VALIDATE_BOOL),
    ],
];
```

**Règle stricte : aucun identifiant de connexion (mot de passe, hôte de production) n'est écrit en dur dans le code source**, ni dans `Config/config.php`, ni ailleurs. C'est une exigence de robustesse minimale, indépendante de la taille du projet.

---

## 13. Le dossier `Helpers`

### 13.1. Rôle, et limite volontaire de ce dossier

Les Helpers sont des fonctions ou petites classes **utilitaires, sans état, sans dépendance métier** — par exemple `UuidGenerator` (génère un UUID v4, `README_DATABASE.md` section 3.1) ou `ClockInterface` (section 5.3).

### 13.2. Ce qui ne doit jamais devenir un Helper

Un piège classique est de transformer ce dossier en fourre-tout où atterrit tout ce qui ne trouve pas sa place ailleurs. Règle simple pour éviter cette dérive : **si une fonction contient une décision métier (même petite), ce n'est pas un Helper, c'est un Service.** Exemple : "formater une date" est un Helper ; "décider si une tâche doit être considérée comme active aujourd'hui" est une règle métier, donc un Service.

---

## 14. Le dossier `Http` (routage et middlewares)

### 14.1. `Router.php`

Fait correspondre une méthode HTTP + une URL à une méthode de Controller. En V1 web, ce routeur ne dessert que quelques routes ; en V2 (API REST, `README_API.md`), il dessert l'ensemble des routes `/api/v1/...`.

### 14.2. Pourquoi des Middlewares

Un Middleware est une fonction qui s'exécute **avant** (et parfois après) qu'une requête n'atteigne un Controller, pour une préoccupation transversale à toutes les routes — c'est-à-dire une préoccupation qui n'appartient à aucune route en particulier, mais à toutes.

| Middleware | Rôle |
|---|---|
| `ErrorHandlerMiddleware` | Attrape toute exception non gérée par un Controller (notamment les `DomainException`, section 9) et la traduit en réponse JSON d'erreur standardisée (voir `README_API.md`) |
| `JsonBodyParserMiddleware` | Décode le corps JSON d'une requête entrante avant qu'un Controller n'y accède |
| `AuthMiddleware` | Vérifie la présence et la validité d'un jeton d'authentification pour les routes de l'API (voir `README_API.md`) |
| `CorsMiddleware` | Ajoute les en-têtes CORS nécessaires pour qu'un futur front-end JavaScript (PWA) ou une application Flutter puissent appeler l'API depuis une origine différente |

### 14.3. Pourquoi centraliser la gestion d'erreurs dans un Middleware plutôt que dans chaque Controller

Sans `ErrorHandlerMiddleware`, chaque Controller devrait répéter le même bloc `try/catch` pour chaque type de `DomainException` (section 9.3). Centraliser cette traduction "exception métier → réponse HTTP" dans un seul endroit garantit une cohérence totale du format d'erreur à travers toute l'API (détaillé dans `README_API.md`), et évite qu'un Controller oublié ne laisse fuiter une exception PHP brute (avec sa trace complète) vers le client — un risque de sécurité autant qu'un problème de cohérence.

---

## 15. Scénarios complets

### 15.1. Scénario A — Créer une tâche

```
Client HTTP
    │  POST /api/v1/tasks  { "name": "Lire", "daily_target": 1, "target_days": ["MON","WED"] }
    ▼
Router ──▶ AuthMiddleware ──▶ JsonBodyParserMiddleware ──▶ TaskController::store()
    │
    ▼
CreateTaskRequest::fromArray($body)              [DTO]
    │
    ▼
TaskService::createTask($dto)                     [Service]
    │
    ├──▶ TaskValidator::validateForCreation($dto)  [Validator] — échec ? → ValidationException
    │
    ├──▶ new Task(...)                             [Domain]
    ├──▶ new Event(type: CREATE_TASK, ...)          [Domain]
    │
    ▼
TaskRepository::saveWithCreationEvent($task, $event)   [Repository]
    │
    ├──▶ BEGIN TRANSACTION
    ├──▶ INSERT INTO tasks (...)
    ├──▶ INSERT INTO task_target_days (...)
    ├──▶ INSERT INTO events (...)
    └──▶ COMMIT
    │
    ▼
TaskResponse::fromDomain($task)                    [DTO]
    │
    ▼
Response::json($response, 201)
    │
    ▼
Client HTTP  ◀── 201 Created
```

**Ce que ce scénario illustre :** chaque flèche ne traverse qu'une seule couche à la fois ; en cas d'échec de validation, l'exception remonte jusqu'au Controller (ou au `ErrorHandlerMiddleware`) sans jamais atteindre le Repository — aucune transaction MySQL n'est même ouverte pour une donnée invalide.

### 15.2. Scénario B — Valider une tâche (`CHECK`)

```
Client HTTP
    │  POST /api/v1/tasks/{uuid}/checks  { "value": 1, "note": "Course à pied 5km" }
    ▼
EventController::check($uuid, $request)
    │
    ▼
EventService::recordCheck($uuid, $dto)
    │
    ├──▶ TaskRepository::findByUuid($uuid)  — introuvable ? → TaskNotFoundException (404)
    ├──▶ CheckValidator::validate($dto)
    ├──▶ new Event(type: CHECK, value: $dto->value, note: $dto->note, ...)
    │
    ▼
EventRepository::insert($event)
    │
    ▼
EventResponse::fromDomain($event)
    │
    ▼
Response::json($response, 201)
```

### 15.3. Scénario C — Consulter la progression d'une tâche

```
Client HTTP
    │  GET /api/v1/tasks/{uuid}/progress
    ▼
TaskController::progress($uuid)
    │
    ▼
ProgressCalculationService::computeCurrentStreak($uuid)
    │
    ├──▶ EventRepository::findByTaskUuidOrderedByDate($uuid)   [lecture SQL brute]
    │
    └──▶ replayEventsIntoStreak($events)                        [calcul PHP pur, aucun SQL]
    │
    ▼
ProgressResponse::fromCalculation(...)
    │
    ▼
Response::json($response, 200)
```

**Point essentiel, déjà posé dans `README_DATABASE.md` (section 10.6) :** aucune requête SQL de ce scénario ne calcule directement une série de progression. La base de données ne renvoie que des faits bruts, ordonnés chronologiquement ; toute l'intelligence du calcul réside dans `ProgressCalculationService`, en PHP pur.

### 15.4. Scénario D — Importer un fichier `.ptracker`

```
Client HTTP
    │  POST /api/v1/imports  (multipart/form-data, fichier .ptracker)
    ▼
ImportController::store($request)
    │
    ▼
ImportService::import($uploadedFilePath)
    │
    ├──▶ ouverture de l'archive ZIP
    ├──▶ lecture + validation de manifest.json           (README_IMPORT_EXPORT.md, section 9.1)
    ├──▶ lecture + validation complète de tasks.csv        (section 9.2) — tout en mémoire
    ├──▶ lecture + validation complète de events.csv        (section 9.3) — tout en mémoire
    ├──▶ validation croisée (task_id référencés existants)
    │
    ├── invalide ? ──▶ ImportValidationException            → aucune écriture en base
    │
    └── valide
         │
         ▼
    TaskRepository / EventRepository::bulkInsertWithinTransaction(...)
         │
         ├──▶ BEGIN TRANSACTION
         ├──▶ INSERT (tâches)
         ├──▶ INSERT (jours actifs)
         ├──▶ INSERT (événements)
         └──▶ COMMIT
         │
         ▼
    ProgressCalculationService — recalcul des projections concernées
```

Ce scénario reproduit exactement le processus décrit dans `README_IMPORT_EXPORT.md` (section 10), mais montre où chaque étape est implémentée dans l'architecture en couches : la validation appartient à `ImportService` (pas au Controller, pas au Repository), et la transaction reste une responsabilité exclusive du Repository (cohérent avec la section 6.3 de ce document).

---

## 16. Pourquoi cette architecture prépare l'API REST, la PWA et Flutter

### 16.1. L'API REST ne fait que rejouer les Controllers avec un autre format de sortie

`public/index.php` (application web) et `public/api.php` (API REST) partagent **exactement les mêmes Services**. Seuls les Controllers — et encore, pas toujours — et le format de sortie diffèrent. Concrètement, l'ajout de l'API REST décrite dans `README_API.md` ne nécessite **aucune duplication de logique métier** : c'est la conséquence directe de la règle de dépendance posée en section 1.2 (les Services ne connaissent jamais HTTP).

```
┌───────────────┐        ┌──────────────────┐
│  index.php       │        │   api.php            │
│  (interface web) │        │   (API REST)          │
└───────┬───────┘        └────────┬─────────┘
        │                          │
        └────────────┬────────────┘
                     ▼
             Services (partagés,
             logique métier unique)
```

### 16.2. Progressive Web App (PWA)

La transformation en PWA, prévue avant l'application Flutter (`CAHIER_DES_CHARGES.md`), consiste essentiellement à faire consommer l'API REST par le JavaScript du front-end existant, plutôt que de générer du HTML côté serveur. Comme l'API REST elle-même ne duplique aucune logique métier (section 16.1), cette transition ne demande aucune réécriture des Services ni des Repositories — seule la couche Interface évolue.

### 16.3. Application Flutter

L'application Flutter consommera la même API REST que la PWA, exactement de la même façon qu'un navigateur web. **Aucune ligne de logique métier PHP n'a besoin d'être portée vers Dart**, à une exception près et assumée : le calcul de progression en mode hors-ligne (`ProgressCalculationService`, section 5.4), si ce mode est un jour implémenté, devra être réimplémenté en Dart — mais comme cet algorithme ne dépend que d'une liste d'événements en entrée (jamais de MySQL), sa logique pourra être transposée directement, sans avoir à comprendre ou réinterpréter tout le reste de l'architecture PHP.

```
┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│  Navigateur     │   │  PWA            │   │  App Flutter    │
│  (V1 web)        │   │  (V2)             │   │  (V3)              │
└──────┬───────┘   └──────┬───────┘   └──────┬───────┘
       │                     │                     │
       └─────────────────────┴─────────────────────┘
                             ▼
                    API REST (README_API.md)
                             ▼
                    Controllers → Services → Repositories → MySQL
```

---

## 17. Résumé des règles à ne jamais enfreindre

1. Une couche ne connaît que la couche immédiatement inférieure (section 1.2).
2. Aucune logique métier dans un Controller (section 1.3).
3. Aucun SQL en dehors des Repositories (section 6.2).
4. Les Services dépendent d'interfaces de Repository, jamais d'implémentations concrètes (section 2.2).
5. `EventRepositoryInterface` n'expose jamais de méthode `update()` ni `delete()` (section 6.4).
6. Aucun calcul de série de progression en SQL — toujours en PHP, dans un Service (section 5.4, cohérent avec `README_DATABASE.md` section 10.6).
7. Les objets `Domain` ne sont jamais exposés tels quels à l'extérieur — toujours via un DTO de réponse (section 8.2).
8. Aucun identifiant interne (`BIGINT`) n'est jamais exposé publiquement — seul l'`uuid` l'est (section 8.3, cohérent avec `README_DATABASE.md` section 3.1).
9. Aucun identifiant de connexion en dur dans le code (section 12).

---

*Fin du document — `README_ARCHITECTURE.md`, cohérent avec `README_DATABASE.md` et `README_IMPORT_EXPORT.md`. Sert de base à `README_API.md`.*
