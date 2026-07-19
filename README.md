# Progress Tracker

> **Application web de suivi de progression personnelle**

Version : **1.1**  
Statut : **Cahier des charges fonctionnel et technique**

---

# Sommaire

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
- [14. Conclusion](#14-conclusion)

---

# 1. Présentation du projet

## Description

**Progress Tracker** est une application permettant de suivre des habitudes ou des objectifs personnels nécessitant une répétition régulière.

Chaque fois qu'une tâche est réalisée, l'utilisateur la valide en cliquant sur un bouton **Accompli**.

L'application enregistre alors :

- la date ;
- l'heure ;
- la progression actuelle ;
- l'historique complet.

L'utilisateur peut également remettre sa progression à zéro lorsqu'il manque une journée, sans perdre les données des séries précédentes.

L'objectif est de fournir un véritable historique de progression plutôt qu'un simple compteur.

---

# 2. Contexte

Certaines habitudes demandent une répétition quotidienne :

- se laver matin et soir ;
- faire du sport ;
- lire ;
- travailler un cours ;
- boire suffisamment d'eau ;
- prier ;
- méditer.

L'utilisateur souhaite mesurer sa régularité sur plusieurs semaines ou plusieurs mois.

L'application devra notamment permettre de répondre à des questions telles que :

- Depuis combien de jours suis-je régulier ?
- À quelle heure ai-je validé ma tâche aujourd'hui ?
- Combien de séries ai-je réalisées ?
- Quelle est ma meilleure série ?
- Quelles périodes de progression ai-je déjà réalisées ?

Toutes les données devront rester consultables.

---

# 3. Vision du projet

L'objectif n'est pas uniquement de créer une petite application web.

Le projet est pensé comme une base solide pouvant évoluer progressivement vers un véritable outil personnel de suivi.

L'application devra donc être conçue dès le départ autour des principes suivants :

- architecture propre ;
- séparation des responsabilités ;
- logique métier indépendante de l'interface ;
- forte évolutivité ;
- possibilité d'ajouter facilement de nouvelles règles métier ;
- possibilité de développer plusieurs interfaces (Web, Mobile, API) sans réécrire le cœur de l'application.

Le projet constitue également un exercice de conception logicielle et d'architecture orientée objet.

---

# 4. Objectifs

L'application devra permettre de :

- créer plusieurs tâches ;
- suivre leur progression ;
- enregistrer chaque validation ;
- conserver l'historique complet ;
- calculer automatiquement les séries de progression ;
- remettre une série à zéro sans supprimer les anciennes ;
- gérer plusieurs tâches simultanément ;
- préparer le projet à une évolution vers une application mobile.

---

# 5. Fonctionnalités

## Création d'une tâche

Chaque tâche possède :

- un nom ;
- une description (optionnelle) ;
- un objectif (optionnel).

Exemple :

```
Nom

Faire du sport

Objectif

30 jours

Description

30 minutes minimum.
```

---

## Tableau de bord

Chaque tâche affiche :

- son nom ;
- la progression actuelle ;
- l'objectif ;
- la dernière validation ;
- un bouton **Accompli** ;
- un bouton **Reset**.

Exemple :

```
SPORT

18 jours

Objectif

30 jours

Dernière validation

Aujourd'hui
08h15

[ Accompli ]

[ Reset ]
```

---

## Validation

Lorsqu'une tâche est validée :

- une seule validation est autorisée par jour ;
- la date est enregistrée ;
- l'heure est enregistrée ;
- la progression est recalculée.

---

## Historique

Toutes les validations sont conservées.

| Date | Heure |
|------|--------|
|01/04|08h15|
|02/04|08h12|
|03/04|08h08|

---

## Reset

Le Reset :

- termine la série actuelle ;
- remet le compteur à zéro ;
- crée automatiquement une nouvelle série ;
- conserve les anciennes.

---

## Historique des séries

Chaque tâche pourra afficher :

```
Progression 1

01 Avril

↓

11 Avril

11 jours

--------------------

Progression 2

13 Avril

↓

Aujourd'hui

8 jours
```

---

## Objectif

Lorsqu'un objectif est défini :

```
18 / 30 jours
```

Une barre de progression pourra être affichée.

---

# 6. Règles de gestion

## RG-01

Une tâche ne peut être validée qu'une seule fois par jour.

## RG-02

Chaque validation possède :

- une date ;
- une heure.

## RG-03

Les validations ne sont jamais supprimées.

## RG-04

Le Reset ne supprime jamais les anciennes données.

## RG-05

Chaque Reset clôt une série.

## RG-06

Une tâche peut posséder plusieurs séries.

## RG-07

Le compteur affiché correspond uniquement à la série active.

## RG-08

Le nombre de jours est calculé automatiquement.

---

# 7. Modèle métier

Le domaine métier repose sur trois entités principales.

## Tâche

Représente une habitude.

Exemples :

- Lecture
- Sport
- Douche
- Prière

---

## Série

Une série représente une période continue de réussite.

Exemple :

```
01 Avril

↓

15 Avril

15 jours
```

Une tâche possède :

- une série active ;
- plusieurs anciennes séries.

---

## Validation

Une validation représente un clic sur **Accompli**.

Elle contient :

- la date ;
- l'heure ;
- le timestamp complet.

---

# 8. Interfaces prévues

## Tableau de bord

```
Mes tâches

---------------------------------

SPORT

18 jours

[ Accompli ]

[ Reset ]

---------------------------------

Lecture

6 jours

[ Accompli ]

[ Reset ]
```

---

## Historique

```
SPORT

Progression actuelle

18 jours

Historique

Progression 1

11 jours

01 Avril

↓

11 Avril

-----------------

Progression 2

18 jours

13 Avril

↓

Aujourd'hui
```

---

## Création

```
Nom

Description

Objectif

[ Enregistrer ]
```

---

# 9. Contraintes techniques

## Technologies

Backend :

- PHP 8+

Base de données :

- MySQL 8

Frontend :

- HTML5
- CSS3
- JavaScript
- Bootstrap (optionnel)

Accès aux données :

- PDO
- Requêtes préparées

---

# 10. Architecture globale

## Philosophie

Le projet devra être conçu selon une architecture en couches.

```
Interface Web

        │

Controllers

        │

Services

        │

Repositories

        │

Database

        │

MySQL
```

Chaque couche possède une responsabilité unique.

La logique métier devra rester indépendante :

- de la base de données ;
- de l'interface graphique ;
- des appels HTTP.

Cette architecture facilitera les évolutions futures.

---

## Source de vérité

L'application ne devra pas enregistrer directement un compteur.

À la place, elle enregistrera des événements métier :

- création d'une tâche ;
- validation ;
- reset.

Le compteur sera calculé automatiquement à partir de ces événements.

Cette approche présente plusieurs avantages :

- aucune incohérence entre compteur et historique ;
- possibilité de recalculer les statistiques ;
- ajout facile de nouvelles règles ;
- meilleure évolutivité.

L'historique constitue donc la **source de vérité** du système.

---

# 11. Évolutivité vers une application mobile

L'une des exigences majeures du projet est de permettre le développement futur d'une application mobile sans devoir réécrire le backend.

Pour atteindre cet objectif, le projet devra être conçu dès le départ autour d'une architecture compatible avec une API REST.

## Première étape

Développement d'une interface Web.

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

---

## Deuxième étape

Ajout d'une API REST utilisant exactement les mêmes Services.

```
Application Web

        │

        ├───────────────┐

        ▼               ▼

 Controllers        API REST

        │               │

        └──────┬────────┘

               ▼

           Services

               ▼

         Repositories

               ▼

             MySQL
```

Les Services constituent le cœur de l'application.

Les Controllers Web et les Controllers API ne font que les appeler.

---

## Troisième étape

Développement d'une application mobile.

L'application mobile communiquera avec l'API.

```
Flutter

↓

HTTP

↓

API REST

↓

Services

↓

Repositories

↓

MySQL
```

Ainsi :

- aucune logique métier ne sera dupliquée ;
- les règles seront centralisées ;
- le backend restera unique.

---

## Progressive Web App

Avant même le développement de l'application mobile, l'application web pourra évoluer vers une **Progressive Web App (PWA)**.

Cela permettra notamment :

- une installation sur smartphone ;
- un lancement comme une application native ;
- un affichage plein écran ;
- un accès plus rapide ;
- la possibilité d'ajouter ultérieurement des fonctionnalités hors ligne et des notifications.

Cette étape constitue une évolution naturelle avant le développement d'une application mobile dédiée.

---

## Choix technologiques

Le backend restera développé en PHP.

Ce choix est motivé par :

- la simplicité de développement ;
- la maîtrise complète de l'architecture ;
- la facilité d'hébergement ;
- la possibilité de créer facilement une API REST.

Pour la future application mobile, **Flutter** est recommandé.

Flutter permettra de développer :

- Android ;
- iOS ;
- Windows ;
- Linux ;
- macOS ;
- Web.

à partir d'un code unique.

---

# 12. Évolutions futures

Le projet devra permettre l'ajout de :

- authentification ;
- multi-utilisateurs ;
- statistiques ;
- graphiques ;
- calendrier ;
- notifications ;
- rappels ;
- catégories ;
- priorités ;
- tâches hebdomadaires ;
- tâches mensuelles ;
- badges ;
- score de régularité ;
- export PDF ;
- export Excel ;
- API publique ;
- synchronisation Cloud ;
- application Flutter.

---

# 13. Critères d'acceptation

Le projet sera considéré conforme lorsque :

- plusieurs tâches peuvent être créées ;
- une tâche peut être validée quotidiennement ;
- une seule validation par jour est autorisée ;
- les validations sont historisées ;
- la date et l'heure sont enregistrées ;
- les séries sont calculées automatiquement ;
- le Reset remet uniquement la série active à zéro ;
- les anciennes séries restent consultables ;
- l'architecture permet l'ajout d'une API REST sans modification majeure ;
- une future application Flutter pourra consommer cette API.

---

# 14. Conclusion

**Progress Tracker** est conçu comme un projet d'apprentissage et de production.

À court terme, il offrira une interface web permettant de suivre efficacement des habitudes personnelles.

À moyen terme, il pourra évoluer vers une **Progressive Web App**, offrant une expérience proche d'une application native sur smartphone.

À long terme, son architecture permettra le développement d'une application **Flutter** utilisant le même backend PHP via une API REST, garantissant la réutilisation de la logique métier et la pérennité du projet.

L'objectif est donc de construire un logiciel robuste, modulaire et évolutif, capable d'accompagner de futures fonctionnalités sans remise en cause de son architecture fondamentale.
