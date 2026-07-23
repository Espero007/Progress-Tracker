Tu es un architecte logiciel senior et un expert en conception de formats d'échange de données.

Tu m'assistes dans la conception d'une application appelée **Progress Tracker**, une application web (PHP/MySQL) destinée à suivre des habitudes personnelles.

Je souhaite que tu rédiges un document **README_IMPORT_EXPORT.md** extrêmement complet en Markdown.

Ce document ne doit pas être un simple tutoriel.

Il doit constituer la **spécification officielle du système d'import/export** de l'application.

Le document servira de référence pendant toute la durée du développement.

Il doit donc être très pédagogique, très détaillé et justifier les choix effectués.

Le document devra comporter au minimum les sections suivantes :

# 1. Objectifs

- Pourquoi un système d'import/export ?
- Cas d'utilisation
- Sauvegarde
- Migration
- Synchronisation
- Partage

# 2. Philosophie

Expliquer que l'application est basée sur un historique d'événements (Event Log).

Montrer que les compteurs ne sont jamais sauvegardés mais toujours recalculés.

Expliquer pourquoi cette approche est plus robuste.

# 3. Format officiel d'échange

Présenter le futur format officiel :

ProgressTrackerExport.ptracker

Expliquer qu'il s'agit en réalité d'une archive ZIP.

Présenter sa structure.

Exemple :

ProgressTrackerExport.ptracker

├── manifest.json
├── tasks.csv
└── events.csv

Expliquer pourquoi ce choix.

# 4. manifest.json

Décrire entièrement son rôle.

Présenter tous les champs possibles.

Expliquer chacun d'eux.

Par exemple :

- application
- application_version
- format
- format_version
- exported_at
- timezone
- files

Donner plusieurs exemples.

Expliquer pourquoi application_version est différente de format_version.

# 5. tasks.csv

Décrire précisément chaque colonne.

Par exemple :

- task_id
- name
- description
- daily_target
- target_days
- status
- created_at

Pour chaque champ :

- description
- type
- contraintes
- exemples
- justification

Donner plusieurs exemples de contenu.

# 6. events.csv

Décrire précisément chaque colonne.

- event_id
- datetime
- event_type
- task_id
- value
- note

Même niveau de détail.

# 7. Types d'événements

Décrire chaque événement.

Au minimum :

CREATE_TASK

CHECK

RESET

ARCHIVE_TASK

RESTORE_TASK

UPDATE_TASK

DELETE_TASK (si pertinent)

Présenter leur rôle.

Donner des exemples.

Expliquer leur impact métier.

# 8. Conventions

Définir toutes les conventions.

Format des dates.

Encodage.

Fuseaux horaires.

UTF-8.

Séparateur CSV.

Gestion des retours à la ligne.

Gestion des caractères spéciaux.

Identifiants.

# 9. Validation avant import

Décrire toutes les vérifications réalisées.

Exemples :

manifest absent

CSV manquant

colonnes invalides

identifiants dupliqués

task_id inconnu

format de date invalide

etc.

# 10. Processus d'import

Décrire pas à pas le déroulement.

Lecture du manifest.

Validation.

Lecture des tâches.

Lecture des événements.

Transactions.

Rollback.

Messages d'erreurs.

# 11. Processus d'export

Décrire pas à pas.

# 12. Compatibilité

Décrire la stratégie de compatibilité entre versions.

Anciennes versions.

Nouvelles versions.

Migration.

# 13. Exemples complets

Fournir plusieurs exports complets.

manifest.json

tasks.csv

events.csv

# 14. Évolutions futures

Expliquer comment le format pourra évoluer sans casser les anciennes versions.

Le document doit être rédigé comme une documentation professionnelle destinée à une équipe de développement.

Il doit être extrêmement détaillé.

Chaque choix technique doit être expliqué.

Produire directement un document Markdown exploitable.