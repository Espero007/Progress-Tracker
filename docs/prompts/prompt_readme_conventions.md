Tu es un Software Architect senior, expert en conception logicielle, en architecture d'entreprise, en Domain Driven Design, en PHP, en bases de données relationnelles et en rédaction de documentation technique.

Tu participes à la conception d'un projet appelé **Progress Tracker**.

Le projet possède déjà :

- un cahier des charges fonctionnel ;
- une architecture générale en couches ;
- un système d'import/export basé sur des fichiers CSV et un manifest.json.

Ta mission est maintenant de rédiger un document appelé :

README_CONVENTIONS.md

Ce document ne décrit ni l'architecture ni les fonctionnalités.

Il définit les conventions officielles du projet.

Il servira de référence pendant toute la durée du développement afin que tous les composants de l'application respectent les mêmes règles.

Le document devra être extrêmement détaillé.

Il devra justifier chacune des décisions prises.

Il devra être rédigé en Markdown.

Le lecteur est un développeur qui rejoint le projet.

Le document devra comporter au minimum les sections suivantes.

# 1. Présentation

Expliquer l'objectif du document.

Expliquer qu'il centralise toutes les conventions utilisées dans le projet.

Expliquer pourquoi il est important de documenter les conventions plutôt que de les laisser implicites.

Présenter les bénéfices :

- cohérence
- maintenabilité
- évolutivité
- réduction des erreurs
- facilité d'intégration de nouveaux développeurs

---

# 2. Philosophie générale

Expliquer les grands principes qui guident tout le projet.

Notamment :

- simplicité
- modularité
- séparation des responsabilités
- architecture en couches
- logique métier indépendante
- historique comme source de vérité
- calcul plutôt que stockage des compteurs
- forte évolutivité

Expliquer pourquoi ces choix ont été retenus.

---

# 3. Langue du projet

Définir précisément quelles parties du projet sont rédigées :

- en anglais
- en français

Par exemple :

Code :

anglais

BDD :

anglais

CSV :

anglais

Documentation :

français

Interface utilisateur :

français

Messages d'erreur :

français

etc.

Justifier ce choix.

---

# 4. Conventions de nommage

Décrire les conventions de nommage :

Classes

Interfaces

Enums

Traits

Variables

Constantes

Méthodes

Fonctions

Colonnes SQL

Tables SQL

Routes

Fichiers

Dossiers

CSV

JSON

Clés JSON

Présenter plusieurs exemples.

---

# 5. Identifiants

Décrire les conventions concernant les identifiants.

task_id

event_id

series_id

user_id

etc.

Expliquer pourquoi utiliser des identifiants textuels stables.

Présenter plusieurs formats possibles.

Comparer avec les identifiants auto-incrémentés.

Conclure sur la solution retenue.

---

# 6. Conventions de dates

Décrire :

- format des dates
- format des heures
- timezone
- ISO 8601
- datetime SQL
- affichage utilisateur

Présenter plusieurs exemples.

---

# 7. Conventions CSV

Décrire :

- séparateur
- encodage
- BOM
- ordre des colonnes
- valeurs nulles
- caractères spéciaux
- guillemets
- retours à la ligne

Présenter plusieurs exemples.

---

# 8. Conventions JSON

Décrire :

- indentation
- encodage
- ordre des propriétés
- camelCase ou snake_case
- null
- booléens

Présenter plusieurs exemples.

---

# 9. Types d'événements

Lister officiellement tous les événements du projet.

Par exemple :

CREATE_TASK

CHECK

RESET

UPDATE_TASK

ARCHIVE_TASK

RESTORE_TASK

DELETE_TASK

Ajouter éventuellement d'autres événements jugés pertinents.

Pour chacun :

- description
- rôle
- impact métier
- exemple

---

# 10. Sources d'événements

Décrire la colonne source.

Par exemple :

MANUAL

WEB

PWA

ANDROID

IOS

API

IMPORT

SYSTEM

Présenter leur signification.

Expliquer leur utilité.

---

# 11. Statuts

Décrire les différents statuts :

ACTIVE

ARCHIVED

DELETED

PAUSED

etc.

Présenter les règles de transition.

---

# 12. Conventions SQL

Décrire :

- clés primaires
- clés étrangères
- index
- contraintes
- transactions
- soft delete
- normalisation

---

# 13. Conventions PHP

Décrire :

- PSR
- namespaces
- autoloading
- exceptions
- DTO
- Services
- Repositories
- Validators
- Helpers

Présenter plusieurs exemples.

---

# 14. Conventions API

Même si l'API n'existe pas encore.

Décrire :

- JSON
- REST
- versionnement
- endpoints
- codes HTTP

---

# 15. Conventions d'import/export

Faire un rappel du format officiel.

manifest.json

tasks.csv

events.csv

Décrire les validations.

Décrire les erreurs.

Décrire les comportements attendus.

---

# 16. Conventions de développement

Présenter toutes les règles à respecter lors de l'ajout d'une nouvelle fonctionnalité.

Par exemple :

- ne jamais casser la compatibilité
- privilégier l'ajout plutôt que la modification
- documenter chaque évolution du format
- préserver les anciennes versions

---

# 17. Décisions d'architecture

Créer une section retraçant les décisions importantes (Architecture Decision Record simplifié).

Pour chaque décision :

- contexte
- décision
- justification
- conséquences

Présenter plusieurs ADR.

---

# 18. Glossaire

Définir précisément les principaux termes du projet :

Task

Series

Event

Validation

Reset

Progression

Import

Export

Manifest

Source

Archive

etc.

---

# 19. Évolutions futures

Présenter les conventions qui devront être conservées lors des futures évolutions :

API

Flutter

PWA

Notifications

Synchronisation

Cloud

Multi-utilisateur

etc.

---

Le document devra être extrêmement professionnel.

Il devra être rédigé comme une véritable charte technique d'entreprise.

Chaque convention devra être justifiée.

Le résultat attendu est un README_CONVENTIONS.md complet, pédagogique et directement exploitable pendant tout le développement du projet.
