# Chapitre 2 — Structures de stockage

> Une base de données Oracle ne stocke pas les données de façon désordonnée sur le disque.
> Elle s'appuie sur une hiérarchie de structures logiques et physiques précisément organisées,
> que tout DBA doit comprendre pour gérer efficacement l'espace et les performances.

---

## Table des matières

- [2.1 Vue d'ensemble des structures de stockage](#21-vue-densemble-des-structures-de-stockage)
- [2.2 Structures logiques](#22-structures-logiques)
  - [2.2.1 Les tablespaces](#221-les-tablespaces)
  - [2.2.2 Les segments](#222-les-segments)
  - [2.2.3 Les extensions (Extents)](#223-les-extensions-extents)
  - [2.2.4 Les blocs de données Oracle](#224-les-blocs-de-données-oracle)
- [2.3 Structures physiques](#23-structures-physiques)
- [2.4 Gestion des tablespaces](#24-gestion-des-tablespaces)
  - [2.4.1 Gestion de l'espace dans les tablespaces](#241-gestion-de-lespace-dans-les-tablespaces)
  - [2.4.2 Créer un tablespace](#242-créer-un-tablespace)
  - [2.4.3 Modifier un tablespace](#243-modifier-un-tablespace)
  - [2.4.4 Supprimer un tablespace](#244-supprimer-un-tablespace)
  - [2.4.5 Consulter les informations relatives aux tablespaces](#245-consulter-les-informations-relatives-aux-tablespaces)
- [Résumé](#résumé)

---

## 2.1 Vue d'ensemble des structures de stockage

Oracle organise le stockage des données sur deux niveaux complémentaires :

- **Structure logique** — comment les données sont organisées du point de vue de la base
- **Structure physique** — comment les données sont réellement stockées sur le disque

La correspondance entre les deux est la suivante :

```
Base de données
    └── Tablespace          ←→    Fichier(s) de données (.dbf)
            └── Segment
                    └── Extent
                            └── Bloc de données Oracle    ←→    Bloc du système d'exploitation
```

Cette hiérarchie permet à Oracle de gérer l'espace de façon abstraite et flexible,
indépendamment des détails du système de fichiers sous-jacent.

---

## 2.2 Structures logiques

### 2.2.1 Les tablespaces

Un **tablespace** est l'unité logique de stockage de plus haut niveau.
C'est dans les tablespaces que sont organisés tous les objets d'une base Oracle
(tables, index, séquences, etc.).

**Caractéristiques essentielles :**

- Une base de données doit avoir au minimum un tablespace : le tablespace **SYSTEM**
  (qui contient le dictionnaire de données)
- Un tablespace ne peut appartenir qu'à une seule base de données
- Chaque tablespace est composé d'un ou plusieurs fichiers de données (*datafiles*)
- Un tablespace peut être **actif (online)** — ses données sont accessibles —
  ou **désactivé (offline)** — ses données sont temporairement inaccessibles
- Le tablespace SYSTEM ne peut jamais être désactivé

**Avantages d'utiliser plusieurs tablespaces :**

- Attribuer des quotas d'espace aux utilisateurs
- Contrôler la disponibilité des données (mettre un tablespace en lecture seule)
- Améliorer les performances en répartissant les zones de charge sur plusieurs disques
- Séparer les données des utilisateurs des objets système

---

### 2.2.2 Les segments

Un **segment** est un ensemble d'extensions (extents) dédié au stockage d'un type
particulier d'informations. Chaque objet de base de données occupe un segment.

**Types de segments :**

| Type | Description |
|------|-------------|
| **Segment de table** | Stocke les lignes d'une table |
| **Segment d'index** | Stocke les entrées d'un index |
| **Segment d'annulation (Undo)** | Stocke les anciennes valeurs avant modification (pour rollback et lecture cohérente) |
| **Segment temporaire** | Utilisé pour les tris et opérations temporaires |

Lors de la création d'un segment, une **extension initiale** lui est allouée.
Si l'espace vient à manquer, de nouvelles extensions sont allouées automatiquement.

---

### 2.2.3 Les extensions (Extents)

Une **extension (extent)** est une suite contiguë de blocs de données sur le disque.
C'est l'unité d'allocation de l'espace dans Oracle.

- Chaque extension est affectée à un seul type de données
- Le nombre de blocs dans une extension est défini par le DBA
- Lorsqu'un segment a besoin de plus d'espace, Oracle lui alloue une nouvelle extension

---

### 2.2.4 Les blocs de données Oracle

Le **bloc de données Oracle** est la plus petite unité de stockage dans Oracle.
Il correspond à un ou plusieurs blocs du système d'exploitation.

- Sa taille est définie par le paramètre `DB_BLOCK_SIZE` (fixé à la création de la base)
- Les valeurs courantes sont : 2 Ko, 4 Ko, 8 Ko, 16 Ko ou 32 Ko
- Tous les accès aux données passent par les blocs

---

## 2.3 Structures physiques

Du point de vue physique, les données sont stockées dans des **fichiers de données**.

| Composant physique | Description |
|-------------------|-------------|
| **Fichiers de données (.dbf)** | Contiennent réellement les données des tablespaces |
| **Fichiers temporaires** | Associés aux tablespaces temporaires (opérations de tri) |
| **Blocs du système d'exploitation** | L'unité de stockage la plus basse — plusieurs peuvent former un bloc Oracle |

**Règles importantes :**

- Un fichier de données ne peut appartenir qu'à un seul tablespace et à une seule base
- Un tablespace peut être constitué de plusieurs fichiers de données
- Les fichiers de données servent de référentiel physique pour tous les objets stockés dans le tablespace

---

## 2.4 Gestion des tablespaces

### 2.4.1 Gestion de l'espace dans les tablespaces

Oracle propose deux modes de gestion de l'espace libre au sein d'un tablespace :

**Tablespace géré localement (recommandé) :**

- Les extents libres sont gérés directement dans le tablespace
- Un **bitmap** est utilisé pour enregistrer l'état de chaque bloc (libre ou occupé)
- Plus performant, moins de contention sur le dictionnaire de données

**Tablespace géré par le dictionnaire :**

- Les extents libres sont gérés par le dictionnaire de données
- Les tables du dictionnaire sont mises à jour à chaque allocation ou libération d'espace
- Moins recommandé (génère plus de contention)

> **Bonne pratique :** Toujours utiliser des tablespaces gérés localement.

---

### 2.4.2 Créer un tablespace

**Syntaxe générale :**

```sql
CREATE [BIGFILE | SMALLFILE] TABLESPACE nom_tablespace
  DATAFILE 'chemin/vers/fichier.dbf' SIZE entier {K|M|G}
  [AUTOEXTEND {OFF | ON [NEXT entier {K|M|G}] [MAXSIZE {UNLIMITED | entier {K|M|G}}]}]
  [{LOGGING | NOLOGGING}]
  [{ONLINE | OFFLINE}];
```

**Paramètres importants :**

| Paramètre | Description |
|-----------|-------------|
| `DATAFILE` | Chemin et taille du fichier de données associé |
| `AUTOEXTEND ON` | Extension automatique du fichier si l'espace vient à manquer |
| `NEXT` | Taille de chaque extension automatique |
| `MAXSIZE` | Taille maximale autorisée pour le fichier |
| `LOGGING` | Les modifications sur les objets du tablespace sont journalisées |
| `NOLOGGING` | Les modifications ne sont pas journalisées (plus rapide, mais moins sûr) |

**Exemples :**

```sql
-- Créer un tablespace simple avec un seul fichier de données
CREATE TABLESPACE TBL01
  DATAFILE 'C:\oracle\oradata\orcl\tbl01fd01.dbf' SIZE 10M;

-- Créer un tablespace avec extension automatique
CREATE TABLESPACE TBL01
  DATAFILE 'C:\oracle\oradata\orcl\tbl01fd01.dbf' SIZE 10M
  AUTOEXTEND ON NEXT 2M MAXSIZE 20M;

-- Créer un tablespace réparti sur plusieurs fichiers de données
CREATE TABLESPACE TBL02
  DATAFILE 'C:\oracle\oradata\orcl\fd01tbl02.dbf' SIZE 10M,
           'C:\oracle\oradata\orcl\fd02tbl02.dbf' SIZE 10M,
           'C:\oracle\oradata\orcl\fd03tbl02.dbf' SIZE 5M;

-- Créer un tablespace temporaire
CREATE TEMPORARY TABLESPACE MonTemp
  TEMPFILE 'C:\oracle\oradata\orcl\montemp01.dbf' SIZE 5M;
```

**Définir le tablespace par défaut du serveur :**

```sql
-- Rendre TBL01 le tablespace permanent par défaut
ALTER DATABASE DEFAULT TABLESPACE TBL01;

-- Rendre MonTemp le tablespace temporaire par défaut
ALTER DATABASE DEFAULT TEMPORARY TABLESPACE MonTemp;
```

---

### 2.4.3 Modifier un tablespace

**Ajouter un fichier de données à un tablespace existant :**

```sql
ALTER TABLESPACE TBL01
  ADD DATAFILE 'C:\oracle\oradata\orcl\tbl01fd02.dbf' SIZE 20M
  AUTOEXTEND ON NEXT 5M MAXSIZE 100M;
```

**Modifier la taille d'un fichier de données existant :**

```sql
ALTER DATABASE DATAFILE 'C:\oracle\oradata\orcl\tbl01fd01.dbf'
  RESIZE 50M;
```

**Ajouter un fichier extensible automatiquement :**

```sql
ALTER TABLESPACE TBL01
  ADD DATAFILE 'C:\oracle\oradata\orcl\fd04tbl01.dbf' SIZE 2M
  AUTOEXTEND ON NEXT 1M MAXSIZE 4M;
```

**Renommer un fichier de données :**

```sql
-- 1. Mettre le tablespace hors ligne
ALTER TABLESPACE TBL02 OFFLINE;

-- 2. Renommer physiquement le fichier au niveau du système d'exploitation
-- (opération OS, hors Oracle)

-- 3. Informer Oracle du nouveau nom
ALTER TABLESPACE TBL02
  RENAME DATAFILE 'ancien_nom.dbf' TO 'nouveau_nom.dbf';

-- 4. Remettre le tablespace en ligne
ALTER TABLESPACE TBL02 ONLINE;
```

**Changer le mode d'accès d'un tablespace :**

```sql
-- Mettre en lecture seule (arrête toutes les écritures)
ALTER TABLESPACE TBL01 READ ONLY;

-- Remettre en lecture-écriture
ALTER TABLESPACE TBL01 READ WRITE;

-- Mettre hors ligne (données inaccessibles)
ALTER TABLESPACE TBL01 OFFLINE;

-- Remettre en ligne
ALTER TABLESPACE TBL01 ONLINE;
```

---

### 2.4.4 Supprimer un tablespace

```sql
-- Suppression logique uniquement (les fichiers physiques restent sur disque)
DROP TABLESPACE nom_tablespace;

-- Suppression logique et suppression des fichiers physiques
DROP TABLESPACE nom_tablespace INCLUDING DATAFILES;

-- Supprimer les objets stockés dans le tablespace avant de le supprimer
DROP TABLESPACE nom_tablespace INCLUDING CONTENTS;

-- Supprimer le tablespace, ses objets et ses contraintes d'intégrité
DROP TABLESPACE nom_tablespace INCLUDING CONTENTS AND DATAFILES CASCADE CONSTRAINTS;
```

> **Attention :** La suppression d'un tablespace est irréversible.
> Toujours vérifier qu'aucun utilisateur n'a ce tablespace comme tablespace par défaut
> avant de le supprimer.

---

### 2.4.5 Consulter les informations relatives aux tablespaces

**Vues principales :**

| Vue | Description |
|-----|-------------|
| `DBA_TABLESPACES` | Liste tous les tablespaces et leurs propriétés |
| `V$TABLESPACE` | Vue dynamique sur les tablespaces |
| `DBA_DATA_FILES` | Liste tous les fichiers de données |
| `V$DATAFILE` | Vue dynamique sur les fichiers de données |
| `DBA_TEMP_FILES` | Liste les fichiers temporaires |
| `V$TEMPFILE` | Vue dynamique sur les fichiers temporaires |
| `DBA_EXTENTS` | Liste toutes les extensions |
| `DBA_SEGMENTS` | Liste tous les segments |
| `DBA_FREE_SPACE` | Espace disponible restant dans chaque tablespace |

**Exemples de requêtes :**

```sql
-- Lister tous les tablespaces
SELECT tablespace_name, status, contents
FROM dba_tablespaces
ORDER BY tablespace_name;

-- Afficher les fichiers de données et leur taille
SELECT tablespace_name, file_name, bytes / 1024 / 1024 AS taille_mo
FROM dba_data_files
ORDER BY tablespace_name;

-- Afficher l'espace libre restant dans chaque tablespace
SELECT tablespace_name, SUM(bytes) / 1024 / 1024 AS espace_libre_mo
FROM dba_free_space
GROUP BY tablespace_name;

-- Afficher la taille du bloc par défaut
SHOW PARAMETER db_block_size;
```

**Bloc PL/SQL — afficher le nom de chaque tablespace et le nombre de fichiers qu'il regroupe :**

```sql
DECLARE
  CURSOR c_ts IS
    SELECT tablespace_name FROM dba_tablespaces ORDER BY tablespace_name;
  v_nb_fichiers NUMBER;
BEGIN
  FOR ts IN c_ts LOOP
    SELECT COUNT(*)
    INTO v_nb_fichiers
    FROM dba_data_files
    WHERE tablespace_name = ts.tablespace_name;

    DBMS_OUTPUT.PUT_LINE(
      'Tablespace : ' || ts.tablespace_name ||
      ' — Nombre de fichiers : ' || v_nb_fichiers
    );
  END LOOP;
END;
/
```

**Procédure stockée — afficher le nom, la taille totale et la taille occupée de chaque tablespace :**

```sql
CREATE OR REPLACE PROCEDURE PS_DETAILS_TAB IS
  CURSOR c_ts IS
    SELECT tablespace_name FROM dba_tablespaces;
  v_taille_totale  NUMBER;
  v_espace_libre   NUMBER;
  v_espace_occupe  NUMBER;
BEGIN
  FOR ts IN c_ts LOOP
    -- Taille totale du tablespace
    SELECT NVL(SUM(bytes), 0)
    INTO v_taille_totale
    FROM dba_data_files
    WHERE tablespace_name = ts.tablespace_name;

    -- Espace libre
    SELECT NVL(SUM(bytes), 0)
    INTO v_espace_libre
    FROM dba_free_space
    WHERE tablespace_name = ts.tablespace_name;

    v_espace_occupe := v_taille_totale - v_espace_libre;

    DBMS_OUTPUT.PUT_LINE(
      'Tablespace : ' || ts.tablespace_name ||
      ' | Taille totale : ' || ROUND(v_taille_totale / 1024 / 1024, 2) || ' Mo' ||
      ' | Taille occupée : ' || ROUND(v_espace_occupe / 1024 / 1024, 2) || ' Mo'
    );
  END LOOP;
END PS_DETAILS_TAB;
/

-- Exécuter la procédure
EXEC PS_DETAILS_TAB;
```

**Fonction stockée — retourner le nombre de tablespaces temporaires :**

```sql
CREATE OR REPLACE FUNCTION FN_NBR_TAB_TEMP RETURN NUMBER IS
  v_nb NUMBER;
BEGIN
  SELECT COUNT(*)
  INTO v_nb
  FROM dba_tablespaces
  WHERE contents = 'TEMPORARY';

  RETURN v_nb;
END FN_NBR_TAB_TEMP;
/

-- Appeler la fonction
SELECT FN_NBR_TAB_TEMP AS nb_tablespaces_temp FROM dual;
```

---

## Résumé

| Concept | Point clé |
|---------|-----------|
| Tablespace | Unité logique de stockage ; composé d'un ou plusieurs fichiers de données |
| Segment | Ensemble d'extensions dédié à un objet (table, index, etc.) |
| Extension (Extent) | Suite contiguë de blocs ; unité d'allocation de l'espace |
| Bloc Oracle | Plus petite unité de stockage ; taille définie par `DB_BLOCK_SIZE` |
| Fichier de données | Stockage physique sur disque ; appartient à un seul tablespace |
| `DBA_TABLESPACES` | Vue principale pour consulter les tablespaces |
| `DBA_DATA_FILES` | Vue principale pour consulter les fichiers de données |
| `DBA_FREE_SPACE` | Vue pour connaître l'espace disponible |
| `CREATE TABLESPACE` | Commande de création d'un tablespace |
| `ALTER TABLESPACE` | Commande de modification (ajout de fichiers, mise en ligne/hors ligne, etc.) |
| `DROP TABLESPACE` | Commande de suppression (irréversible) |

---

*Chapitre précédent : [Architecture du serveur Oracle ←](./chapter1_oracle_architecture.md)*
*Chapitre suivant : [Sécurité utilisateur →](./chapter3_user_security.md)*
