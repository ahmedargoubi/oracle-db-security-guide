# Chapitre 1 — Architecture du serveur Oracle

> Comprendre comment Oracle est construit en interne est le fondement de tout le reste.
> Avant de pouvoir gérer, optimiser ou sécuriser une base de données, il faut savoir ce qui
> se passe en coulisses lorsqu'un utilisateur se connecte, exécute une requête ou valide une transaction.

---

## Table des matières

- [1.1 Le serveur Oracle](#11-le-serveur-oracle)
- [1.2 Connexion à la base de données](#12-connexion-à-la-base-de-données)
- [1.3 L'instance Oracle](#13-linstance-oracle)
  - [1.3.1 La mémoire SGA](#131-la-mémoire-sga)
  - [1.3.2 Les processus en arrière-plan](#132-les-processus-en-arrière-plan)
  - [1.3.3 La mémoire PGA](#133-la-mémoire-pga)
- [1.4 La base de données Oracle (fichiers physiques)](#14-la-base-de-données-oracle-fichiers-physiques)
  - [1.4.1 Le dictionnaire de données](#141-le-dictionnaire-de-données)
- [1.5 Démarrage et arrêt de la base](#15-démarrage-et-arrêt-de-la-base)
  - [1.5.1 Démarrer la base de données](#151-démarrer-la-base-de-données)
  - [1.5.2 Arrêter la base de données](#152-arrêter-la-base-de-données)
- [Résumé](#résumé)

---

## 1.1 Le serveur Oracle

Un **serveur Oracle** est un système de gestion de base de données qui fournit une approche
intégrée, complète et ouverte de la gestion des informations. Il se compose de deux éléments principaux :

- **Une instance Oracle** — la composante en mémoire et en processus
- **Une base de données Oracle** — les fichiers physiques stockés sur disque

Ces deux éléments sont distincts mais fonctionnent ensemble. L'instance est ce qui s'exécute
en mémoire ; la base de données est ce qui persiste sur disque.

**Deux modèles de connexion sont supportés :**

- **Client/Serveur** — une application cliente se connecte directement au serveur de base de données Oracle
- **3 niveaux (3-Tier)** — les clients passent par un serveur d'application (ex. Oracle Application Server),
  qui se connecte à son tour à la base de données

---

## 1.2 Connexion à la base de données

Lorsqu'un utilisateur souhaite se connecter à une base Oracle, la connexion suit plusieurs étapes :

1. Le client contacte le **Listener Oracle** en demandant un service par son nom
2. Le Listener démarre un processus dédié appelé **processus serveur**
3. Le Listener envoie l'adresse du processus serveur au client
4. Le client établit une connexion directe avec le processus serveur
5. Le processus serveur se connecte à l'instance Oracle pour le compte de l'utilisateur,
   créant ainsi une **session utilisateur**

> **Point clé :** Le processus utilisateur n'entre jamais directement en interaction avec le serveur Oracle.
> C'est le *processus serveur* qui interagit avec Oracle, exécute les requêtes
> et renvoie les résultats au client.

---

## 1.3 L'instance Oracle

Une instance Oracle est composée de :

- Une **zone de mémoire partagée** appelée SGA (System Global Area)
- Plusieurs **processus en arrière-plan**, chacun ayant un rôle bien déterminé

L'instance est créée au démarrage d'Oracle et cesse d'exister à son arrêt.
Plusieurs utilisateurs peuvent partager la même instance simultanément.

---

### 1.3.1 La mémoire SGA

La **SGA (System Global Area)** est une zone mémoire partagée par tous les processus serveurs
et les processus en arrière-plan. Elle est allouée au démarrage de l'instance.

**Zones mémoire obligatoires :**

| Zone | Description |
|------|-------------|
| **Zone de mémoire partagée (Shared Pool)** | Contient le Library Cache et le Cache du dictionnaire de données |
| **Cache de tampons (Database Buffer Cache)** | Stocke des copies des blocs de données lus depuis les fichiers de données |
| **Tampon de journalisation (Redo Log Buffer)** | Enregistre toutes les modifications apportées à la base avant leur écriture sur disque |

**Zones mémoire non obligatoires :**

| Zone | Description |
|------|-------------|
| **Large Pool** | Utilisée pour les grandes allocations mémoire (sauvegarde, requêtes parallèles, etc.) |
| **Java Pool** | Dédiée à l'exécution de Java au sein d'Oracle |
| **Streams Pool** | Utilisée par Oracle Streams pour la réplication de données |

#### La zone de mémoire partagée — Shared Pool

Le Shared Pool contient deux structures essentielles :

- **Library Cache** — stocke les instructions SQL et le code PL/SQL analysés (parsés),
  afin d'éviter de les analyser de nouveau à chaque exécution. Le chargement et l'analyse
  d'une instruction SQL consomment beaucoup de ressources ; les partager entre utilisateurs
  améliore significativement les performances.

- **Cache du dictionnaire de données (Row Cache)** — stocke les métadonnées sur les objets
  de la base (noms des tables, définitions des colonnes, privilèges, emplacements des fichiers, etc.).
  Lors de l'analyse d'une requête, le processus serveur y recherche les informations nécessaires
  au lieu de lire depuis le disque à chaque fois.

#### Le cache de tampons — Database Buffer Cache

Ce cache conserve des copies des blocs de données lus depuis les fichiers sur disque.
Lorsqu'un utilisateur lit ou modifie des données, Oracle commence par vérifier ce cache.
Si le bloc s'y trouve déjà (*cache hit*), aucune lecture disque n'est nécessaire.

- Géré par un algorithme **LRU (Least Recently Used)**
- La taille des blocs est contrôlée par le paramètre `DB_BLOCK_SIZE`

#### Le tampon de journalisation — Redo Log Buffer

Enregistre chaque modification apportée à la base (INSERT, UPDATE, DELETE, DDL).
Sa fonction principale est la **récupération de données** — en cas de panne, les modifications
enregistrées peuvent être réappliquées. Sa taille est contrôlée par le paramètre `LOG_BUFFER`.

---

### 1.3.2 Les processus en arrière-plan

Oracle s'appuie sur plusieurs processus en arrière-plan qui s'exécutent automatiquement.
Chacun a un rôle bien défini :

| Processus | Nom complet | Rôle |
|-----------|-------------|------|
| **DBWn** | Database Writer | Écrit les blocs modifiés (dirty) du cache de tampons vers les fichiers de données |
| **LGWR** | Log Writer | Écrit les entrées du tampon de journalisation vers les fichiers de journalisation en ligne |
| **SMON** | System Monitor | Assure la récupération après une panne ; fusionne l'espace libre ; libère les segments temporaires |
| **PMON** | Process Monitor | Nettoie après l'échec d'une session utilisateur (annule les transactions, libère les verrous) |
| **CKPT** | Checkpoint | Signale les points de reprise à DBWn ; met à jour les en-têtes des fichiers de données et de contrôle |
| **ARCn** | Archiver | Copie les fichiers de journalisation en ligne vers des destinations d'archivage (mode ARCHIVELOG) |

#### DBWn — Database Writer

DBWn écrit les tampons modifiés du cache vers le disque dans les cas suivants :

- À un **point de reprise (checkpoint)**
- Lorsque le seuil des tampons modifiés (*dirty*) est atteint
- Lorsqu'aucune mémoire tampon libre n'est disponible
- Lorsque le temps imparti est dépassé
- Lorsqu'un tablespace est mis hors ligne ou en lecture seule
- Lors d'un `DROP` ou `TRUNCATE TABLE`, ou d'un `BEGIN BACKUP`

#### LGWR — Log Writer

LGWR écrit les entrées du tampon de journalisation vers les fichiers redo log dans les cas suivants :

- Lors d'un **COMMIT**
- Lorsqu'un tiers du tampon de journalisation est occupé
- Lorsque la journalisation atteint **1 Mo**
- Toutes les **3 secondes**
- Avant que DBWn n'effectue une opération d'écriture

#### SMON — System Monitor

SMON est responsable de la **récupération de l'instance** après un arrêt anormal :

1. Réapplique les modifications enregistrées dans les fichiers de journalisation (*roll forward*)
2. Ouvre la base de données pour permettre aux utilisateurs de se reconnecter
3. Annule les transactions non validées (*roll back*)

Il fusionne également l'espace libre et libère les segments temporaires inutilisés.

#### PMON — Process Monitor

Lorsqu'une session utilisateur échoue (ex. déconnexion réseau), PMON :

- Annule la transaction incomplète
- Libère tous les verrous détenus par la session
- Libère les autres ressources associées à la session
- Redémarre les processus répartiteurs interrompus

#### CKPT — Checkpoint Process

CKPT gère les événements de point de reprise. À chaque checkpoint, il :

- Signale à DBWn d'écrire les tampons modifiés
- Met à jour les en-têtes des fichiers de données avec les informations de point de reprise
- Met à jour les fichiers de contrôle avec les informations de point de reprise

---

### 1.3.3 La mémoire PGA

La **PGA (Program Global Area)** est une zone mémoire privée allouée pour chaque
session utilisateur connectée. Contrairement à la SGA, elle n'est pas partagée.

La PGA contient :

- Les zones de travail SQL privées (tri, jointures par hachage, etc.)
- Les valeurs des variables attachées (*bind variables*)
- Les informations spécifiques à la session
- L'état des curseurs

Chaque processus serveur et chaque processus en arrière-plan dispose de sa propre PGA
qui lui est exclusivement réservée. Lorsqu'un utilisateur se déconnecte, le processus
serveur associé se termine et la mémoire PGA est libérée.

---

## 1.4 La base de données Oracle (fichiers physiques)

La base de données Oracle est l'ensemble des fichiers physiques persistant sur disque.
Ces fichiers comprennent :

| Type de fichier | Rôle |
|-----------------|------|
| **Fichiers de contrôle** | Contiennent les métadonnées sur la base (structure, statut, infos de checkpoint). Nécessaires pour monter la base. |
| **Fichiers de données** | Stockent les données utilisateur, les métadonnées et le dictionnaire de données |
| **Fichiers de journalisation en ligne** | Utilisés pour la récupération de l'instance ; enregistrent toutes les modifications |
| **Fichiers de journalisation archivés** | Copies historiques des fichiers redo log (utilisées lors d'une récupération complète) |
| **Fichier de paramètres (PFILE / SPFILE)** | Définit la configuration de l'instance au démarrage |
| **Fichier de mots de passe** | Permet aux utilisateurs privilégiés de se connecter à distance |
| **Fichier d'alertes (Alert Log)** | Journal chronologique des messages et erreurs de la base |
| **Fichiers trace** | Générés par les processus serveurs et d'arrière-plan lors d'erreurs internes |

---

### 1.4.1 Le dictionnaire de données

Le **dictionnaire de données** est l'un des composants les plus importants d'une base Oracle.
Il s'agit d'un ensemble de tables et de vues internes qui décrivent la base de données elle-même.

**Caractéristiques principales :**

- Contient les noms et attributs de tous les objets (tables, vues, index, utilisateurs, etc.)
- Mis à jour automatiquement à chaque création ou modification d'un objet
- Créé en même temps que la base de données
- Appartient à l'utilisateur `SYS`
- Non accessible directement — consulté via des vues prédéfinies

**Préfixes des vues du dictionnaire :**

| Préfixe | Portée |
|---------|--------|
| `USER_` | Objets appartenant à l'utilisateur courant |
| `ALL_` | Objets accessibles à l'utilisateur courant |
| `DBA_` | Tous les objets de la base (nécessite des privilèges DBA) |
| `V$` | Vues dynamiques de performance (activité en temps réel, mémoire, sessions) |

**Exemples de requêtes sur le dictionnaire :**

```sql
-- Lister toutes les vues du dictionnaire, triées par nom
SELECT table_name, comments FROM dictionary ORDER BY table_name;

-- Lister tous les objets appartenant à l'utilisateur courant
SELECT object_name, object_type, created, status
FROM user_objects
ORDER BY object_type;

-- Lister les tables appartenant à l'utilisateur courant
SELECT table_name FROM user_tables;

-- Décrire les colonnes d'une table
SELECT column_name, data_type, data_length, nullable
FROM user_tab_columns
WHERE table_name = 'EMPLOYEES';

-- Lister les contraintes d'une table
SELECT constraint_name, constraint_type, search_condition, status
FROM user_constraints
WHERE table_name = 'EMPLOYEES';

-- Lister les colonnes associées aux contraintes
SELECT constraint_name, column_name
FROM user_cons_columns
WHERE table_name = 'EMPLOYEES';

-- Lister les séquences et leur prochain numéro disponible
SELECT sequence_name, min_value, max_value, increment_by, last_number
FROM user_sequences;

-- Lister les synonymes de l'utilisateur courant
SELECT synonym_name, table_owner, table_name
FROM user_synonyms;
```

**Vues dynamiques V$ :**

Ces vues sont constamment mises à jour et reflètent l'activité courante de la base.
Elles lisent les données depuis la mémoire et les fichiers de contrôle.
Elles sont accessibles uniquement par un DBA.

```sql
-- Afficher les sessions actives
SELECT username, status, machine FROM v$session;

-- Afficher la liste des fichiers de journalisation
SELECT member FROM v$logfile;

-- Vérifier le statut des groupes de fichiers redo log
SELECT group#, status, members FROM v$log;
```

---

## 1.5 Démarrage et arrêt de la base

### 1.5.1 Démarrer la base de données

Le démarrage d'une base Oracle se fait en trois étapes successives :

```
SHUTDOWN → NOMOUNT → MOUNT → OPEN
```

| Étape | Ce qui se passe |
|-------|----------------|
| **NOMOUNT** | Lecture du fichier de paramètres ; allocation de la SGA ; démarrage des processus d'arrière-plan |
| **MOUNT** | Ouverture et lecture du fichier de contrôle ; association des fichiers de la base à l'instance |
| **OPEN** | Ouverture de tous les fichiers de données et de journalisation ; la base est accessible aux utilisateurs |

**Fichiers de paramètres d'initialisation :**

Avant de démarrer, Oracle lit un fichier de paramètres qui lui indique comment configurer l'instance.
Il en existe deux types :

| Type | Nom par défaut | Caractéristiques |
|------|---------------|-----------------|
| **PFILE** | `init<SID>.ora` | Fichier texte, modifiable manuellement, les changements prennent effet au prochain démarrage |
| **SPFILE** | `spfile<SID>.ora` | Fichier binaire, géré dynamiquement par Oracle, ne doit pas être modifié manuellement |

```sql
-- Créer un PFILE à partir du SPFILE courant
CREATE PFILE = 'chemin/vers/init.ora' FROM SPFILE;

-- Créer un SPFILE à partir d'un PFILE
CREATE SPFILE FROM PFILE = 'chemin/vers/init.ora';
```

**Paramètres d'initialisation importants :**

| Paramètre | Description |
|-----------|-------------|
| `CONTROL_FILES` | Emplacement des fichiers de contrôle |
| `DB_BLOCK_SIZE` | Taille des blocs de données (fixée à la création) |
| `PROCESSES` | Nombre maximum de processus utilisateur |
| `DB_CACHE_SIZE` | Taille du cache de tampons |
| `SGA_TARGET` | Taille totale de la SGA |
| `SHARED_POOL_SIZE` | Taille de la zone de mémoire partagée |
| `UNDO_MANAGEMENT` | Mode de gestion du volume d'annulation |
| `PGA_AGGREGATE_TARGET` | Quantité de mémoire PGA allouée à l'instance |

**Modifier un paramètre dynamiquement :**

```sql
-- Modifier un paramètre pour la session en cours uniquement
ALTER SESSION SET parametre = valeur;

-- Modifier un paramètre au niveau de l'instance (en mémoire uniquement)
ALTER SYSTEM SET parametre = valeur SCOPE = MEMORY;

-- Modifier un paramètre dans le SPFILE uniquement (effectif au prochain démarrage)
ALTER SYSTEM SET parametre = valeur SCOPE = SPFILE;

-- Modifier à la fois en mémoire et dans le SPFILE
ALTER SYSTEM SET parametre = valeur SCOPE = BOTH;
```

**Consulter la valeur d'un paramètre :**

```sql
-- Via la vue V$PARAMETER
SELECT name, value, issys_modifiable
FROM v$parameter
WHERE name = 'db_cache_size';

-- Ou directement dans SQL*Plus
SHOW PARAMETER db_cache_size;
```

**La commande STARTUP :**

```sql
-- Démarrage complet (mode OPEN par défaut)
STARTUP

-- Démarrage en mode NOMOUNT (création ou recréation d'une base)
STARTUP NOMOUNT

-- Démarrage en mode MOUNT (opérations de récupération)
STARTUP MOUNT

-- Ouverture en lecture seule
STARTUP OPEN READ ONLY

-- Forcer un redémarrage après une situation anormale
STARTUP FORCE

-- Restreindre l'accès aux utilisateurs privilégiés uniquement
STARTUP RESTRICT
```

**Passer d'un mode à l'autre :**

```sql
-- Passer de NOMOUNT à MOUNT
ALTER DATABASE MOUNT;

-- Passer de MOUNT à OPEN
ALTER DATABASE OPEN;

-- Ouvrir en mode lecture-écriture explicitement
ALTER DATABASE OPEN READ WRITE;

-- Annuler la restriction de session
ALTER SYSTEM DISABLE RESTRICTED SESSION;
```

---

### 1.5.2 Arrêter la base de données

L'arrêt se fait dans l'ordre inverse du démarrage :

```
OPEN → FERMETURE → DÉMONTAGE → ARRÊT DE L'INSTANCE
```

**Étapes réalisées :**

1. **Fermer la base** — Oracle écrit les tampons modifiés et les entrées redo log sur disque,
   puis ferme tous les fichiers de données et de journalisation (les fichiers de contrôle restent ouverts)
2. **Démonter la base** — fermeture des fichiers de contrôle
3. **Arrêter l'instance** — libération de la SGA, arrêt des processus en arrière-plan,
   fermeture des fichiers trace et du fichier d'alertes

**Modes d'arrêt :**

| Mode | Comportement |
|------|-------------|
| `SHUTDOWN NORMAL` | Attend que tous les utilisateurs se déconnectent avant d'arrêter |
| `SHUTDOWN TRANSACTIONAL` | Attend la fin des transactions en cours, puis déconnecte les utilisateurs |
| `SHUTDOWN IMMEDIATE` | Annule les transactions actives immédiatement ; n'attend pas les utilisateurs |
| `SHUTDOWN ABORT` | Arrêt instantané — aucun rollback, aucune fermeture propre ; récupération nécessaire au prochain démarrage |

```sql
-- Arrêt normal (peut prendre longtemps)
SHUTDOWN NORMAL

-- Arrêt immédiat (le plus courant en pratique)
SHUTDOWN IMMEDIATE

-- Arrêt d'urgence (à utiliser avec prudence)
SHUTDOWN ABORT
```

> **Bonne pratique :** Préférer `SHUTDOWN IMMEDIATE` dans la plupart des situations d'administration.
> Utiliser `SHUTDOWN ABORT` uniquement lorsque la base est totalement non réactive.
> Après un `SHUTDOWN ABORT`, Oracle effectuera automatiquement une récupération de l'instance
> au prochain démarrage.

---

## Résumé

| Concept | Point clé |
|---------|-----------|
| Serveur Oracle | = Instance (mémoire + processus) + Base de données (fichiers physiques) |
| SGA | Mémoire partagée : Shared Pool, Buffer Cache, Redo Log Buffer |
| DBWn | Écrit les blocs modifiés du cache vers les fichiers de données |
| LGWR | Écrit le tampon redo log vers les fichiers de journalisation en ligne |
| SMON | Gère la récupération de l'instance après une panne |
| PMON | Nettoie après l'échec d'une session utilisateur |
| CKPT | Signale les checkpoints ; met à jour les en-têtes des fichiers |
| PGA | Mémoire privée par session utilisateur |
| Dictionnaire de données | Métadonnées internes — consulté via les vues USER\_, ALL\_, DBA\_, V$ |
| Étapes de démarrage | NOMOUNT → MOUNT → OPEN |
| Modes d'arrêt | NORMAL / TRANSACTIONAL / IMMEDIATE / ABORT |

---

*Chapitre suivant : [Structures de stockage →](./chapter2_storage_structure.md)*
