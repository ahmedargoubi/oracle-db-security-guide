# Chapitre 4 — Sécurité et audit

> Authentifier les utilisateurs et leur accorder des privilèges ne suffit pas.
> Un système véritablement sécurisé doit aussi surveiller ce que font les utilisateurs
> une fois connectés — même ceux en qui on a confiance. C'est le rôle de l'audit.

---

## Table des matières

- [4.1 Principe du moindre privilège](#41-principe-du-moindre-privilège)
  - [4.1.1 Protéger le dictionnaire de données](#411-protéger-le-dictionnaire-de-données)
  - [4.1.2 Révoquer les privilèges inutiles du rôle PUBLIC](#412-révoquer-les-privilèges-inutiles-du-rôle-public)
  - [4.1.3 Limiter les utilisateurs administrateurs](#413-limiter-les-utilisateurs-administrateurs)
  - [4.1.4 Désactiver l'authentification à distance par l'OS](#414-désactiver-lauthentification-à-distance-par-los)
- [4.2 Audit de la base de données](#42-audit-de-la-base-de-données)
  - [4.2.1 Vue d'ensemble des types d'audit](#421-vue-densemble-des-types-daudit)
- [4.3 Audit standard](#43-audit-standard)
  - [4.3.1 Activer l'audit standard](#431-activer-laudit-standard)
  - [4.3.2 Niveaux d'audit](#432-niveaux-daudit)
  - [4.3.3 Arrêter et annuler un audit](#433-arrêter-et-annuler-un-audit)
  - [4.3.4 Vues d'audit standard](#434-vues-daudit-standard)
- [4.4 Audit basé sur les données (Triggers)](#44-audit-basé-sur-les-données-triggers)
- [4.5 Audit détaillé — Fine Grained Auditing (FGA)](#45-audit-détaillé--fine-grained-auditing-fga)
  - [4.5.1 Le package DBMS_FGA](#451-le-package-dbms_fga)
  - [4.5.2 Créer une stratégie FGA](#452-créer-une-stratégie-fga)
  - [4.5.3 Gérer les stratégies FGA](#453-gérer-les-stratégies-fga)
  - [4.5.4 Vues d'audit FGA](#454-vues-daudit-fga)
- [Résumé](#résumé)

---

## 4.1 Principe du moindre privilège

Le **principe du moindre privilège** est la règle fondamentale de la sécurité Oracle :
n'accorder à chaque utilisateur que les privilèges **strictement nécessaires** pour
accomplir ses tâches, et rien de plus.

Ce principe se traduit concrètement par quatre actions préventives.

---

### 4.1.1 Protéger le dictionnaire de données

Le dictionnaire de données contient toutes les métadonnées de la base. Pour empêcher
les utilisateurs ordinaires d'y accéder directement, le paramètre d'initialisation
`O7_DICTIONARY_ACCESSIBILITY` doit être positionné à `FALSE` (valeur par défaut recommandée).

```sql
-- Vérifier la valeur actuelle
SHOW PARAMETER O7_DICTIONARY_ACCESSIBILITY;

-- Effet de FALSE :
-- - Les utilisateurs avec des privilèges ANY TABLE ne peuvent pas accéder
--   aux tables du dictionnaire de données
-- - L'utilisateur SYS ne peut se connecter que AS SYSDBA
```

---

### 4.1.2 Révoquer les privilèges inutiles du rôle PUBLIC

Le rôle `PUBLIC` est automatiquement accordé à tous les utilisateurs. Tout privilège
accordé à PUBLIC est donc accessible par n'importe qui. Certains packages Oracle
accordent par défaut le privilège `EXECUTE` à PUBLIC, ce qui représente un risque.

Les packages suivants ne doivent **jamais** avoir leurs droits d'exécution accordés à PUBLIC :

| Package | Risque potentiel |
|---------|-----------------|
| `UTL_SMTP` | Envoi de messages électroniques arbitraires depuis la base |
| `UTL_HTTP` | Connexion vers des sites web externes — risque d'exfiltration de données |
| `UTL_FILE` | Lecture et écriture sur le système de fichiers du serveur |

```sql
-- Révoquer les exécutions dangereuses de PUBLIC
REVOKE EXECUTE ON utl_file   FROM PUBLIC;
REVOKE EXECUTE ON utl_http   FROM PUBLIC;
REVOKE EXECUTE ON utl_smtp   FROM PUBLIC;
```

---

### 4.1.3 Limiter les utilisateurs administrateurs

Les comptes avec des privilèges élevés doivent être strictement contrôlés. Il faut
régulièrement vérifier qui dispose du rôle `DBA` ou des connexions `SYSDBA`/`SYSOPER`.

```sql
-- Lister tous les utilisateurs ayant le rôle DBA
SELECT grantee
FROM dba_role_privs
WHERE granted_role = 'DBA';

-- Lister les utilisateurs auxquels SYSDBA ou SYSOPER a été accordé
SELECT * FROM v$pwfile_users;
```

Les types de privilèges à surveiller en priorité :
- Attribution des rôles DBA et SYSDBA
- Privilèges de type `DROP ANY TABLE`, `ALTER ANY TABLE`
- Connexions directes avec les comptes SYS et SYSTEM

---

### 4.1.4 Désactiver l'authentification à distance par l'OS

L'authentification à distance par le système d'exploitation permet à un utilisateur
de se connecter sans mot de passe Oracle depuis une machine distante, en s'appuyant
uniquement sur son identité OS. Cette pratique est risquée.

```sql
-- Vérifier que le paramètre est bien désactivé (valeur FALSE par défaut)
SHOW PARAMETER REMOTE_OS_AUTHENT;

-- Si nécessaire, le désactiver
ALTER SYSTEM SET REMOTE_OS_AUTHENT = FALSE SCOPE = SPFILE;
-- Un redémarrage est nécessaire pour prendre effet
```

---

## 4.2 Audit de la base de données

L'audit consiste à **enregistrer et surveiller les activités** qui ont lieu dans la base
de données. Même des utilisateurs légitimes et authentifiés peuvent, intentionnellement
ou non, compromettre la sécurité du système.

### 4.2.1 Vue d'ensemble des types d'audit

Oracle propose trois mécanismes d'audit complémentaires :

| Type d'audit | Événements audités | Contenu de la trace |
|--------------|--------------------|---------------------|
| **Audit standard** | Utilisation des privilèges, accès aux objets | Ensemble fixe de données |
| **Audit basé sur les données** (Triggers) | Données modifiées par les instructions LMD | Défini librement par l'administrateur |
| **Audit détaillé (FGA)** | Instructions SQL (INSERT, UPDATE, DELETE, SELECT) en fonction du contenu | Ensemble fixe + instruction SQL complète |

---

## 4.3 Audit standard

L'audit standard est le mécanisme intégré d'Oracle. Il est activé via le paramètre
`AUDIT_TRAIL` et enregistre les événements dans une table système ou sur le système de fichiers.

### 4.3.1 Activer l'audit standard

Le paramètre `AUDIT_TRAIL` contrôle où et comment les enregistrements sont stockés :

| Valeur | Comportement |
|--------|-------------|
| `NONE` | Audit désactivé |
| `OS` | Enregistrements stockés dans la trace d'audit du système d'exploitation |
| `DB` | Enregistrements stockés dans la table `SYS.AUD$` (recommandé) |
| `DB, EXTENDED` | Comme `DB`, plus les colonnes `SQLBIND` et `SQLTEXT` de `SYS.AUD$` |
| `XML` | Enregistrements dans des fichiers XML sur le système d'exploitation |
| `XML, EXTENDED` | Comme `XML`, avec en plus `SQLBIND` et `SQLTEXT` |

```sql
-- Afficher la valeur actuelle
SHOW PARAMETER AUDIT_TRAIL;

-- Activer l'audit en base (nécessite un redémarrage)
ALTER SYSTEM SET AUDIT_TRAIL = DB SCOPE = SPFILE;

-- Activer en mode étendu
ALTER SYSTEM SET AUDIT_TRAIL = DB, EXTENDED SCOPE = SPFILE;
```

> **Audit via l'OS vs via la BD :**
> L'audit via l'OS permet de centraliser les traces de plusieurs bases en un seul endroit.
> L'audit via la BD est plus pratique pour l'interrogation SQL, mais la table `SYS.AUD$`
> doit être archivée et purgée régulièrement pour éviter de saturer le tablespace SYSTEM.

> **Bonne pratique de protection :** Auditer toutes les actions sur la table `SYS.AUD$` elle-même,
> pour qu'un utilisateur malveillant ne puisse pas effacer ses traces :
> ```sql
> AUDIT ALL ON SYS.AUD$ BY ACCESS;
> ```

---

### 4.3.2 Niveaux d'audit

L'audit standard opère sur quatre niveaux :

**1. Audit de commandes (LDD)**

Audite les instructions LDD (CREATE, ALTER, DROP…), ciblées ou non :

```sql
-- Non ciblé : toute instruction CREATE TABLE par quiconque
AUDIT TABLE;

-- Ciblé : uniquement pour l'utilisateur HR, en cas d'échec
AUDIT TABLE BY hr WHENEVER NOT SUCCESSFUL;
```

**2. Audit de privilèges système**

Audite l'utilisation de privilèges système :

```sql
-- Non ciblé : toute utilisation de ces privilèges
AUDIT SELECT ANY TABLE, CREATE ANY TRIGGER;

-- Ciblé : pour HR, un seul enregistrement par session
AUDIT SELECT ANY TABLE BY hr BY SESSION;
```

**3. Audit d'objets de schéma**

Audite les accès à des objets précis :

```sql
-- Toutes les opérations sur hr.employees
AUDIT ALL ON hr.employees;

-- UPDATE et DELETE, un enregistrement par accès
AUDIT UPDATE, DELETE ON hr.employees BY ACCESS;
```

**4. Audit de connexion**

Audite les connexions et déconnexions à la base :

```sql
-- Toutes les connexions échouées
AUDIT SESSION WHENEVER NOT SUCCESSFUL;

-- Toutes les connexions (réussies et échouées)
AUDIT SESSION;
```

**Options de granularité :**

| Option | Comportement |
|--------|-------------|
| `BY SESSION` | Un seul enregistrement par session, quel que soit le nombre d'opérations de même type |
| `BY ACCESS` | Un enregistrement par opération — obligatoire pour les instructions LDD |
| `WHENEVER SUCCESSFUL` | Audite uniquement les opérations réussies |
| `WHENEVER NOT SUCCESSFUL` | Audite uniquement les opérations échouées |

---

### 4.3.3 Arrêter et annuler un audit

Pour désactiver un audit, on utilise la commande `NOAUDIT` avec la même syntaxe que `AUDIT` :

```sql
-- Arrêter l'audit de session
NOAUDIT SESSION;
NOAUDIT SESSION BY scott, lori;

-- Arrêter l'audit d'un privilège
NOAUDIT DELETE ANY TABLE;
NOAUDIT SELECT TABLE, INSERT TABLE, DELETE TABLE, EXECUTE PROCEDURE;

-- Arrêter tout audit de commandes et de privilèges
NOAUDIT ALL;
NOAUDIT ALL PRIVILEGES;

-- Arrêter l'audit d'un objet
NOAUDIT DELETE ON emp;
NOAUDIT SELECT, INSERT, DELETE ON jward.dept;
NOAUDIT ALL ON emp;
```

> Pour désactiver un audit de commande ou de privilège, le privilège système `AUDIT SYSTEM` est requis.

---

### 4.3.4 Vues d'audit standard

| Vue | Description |
|-----|-------------|
| `DBA_AUDIT_TRAIL` | Toutes les entrées de la trace d'audit |
| `DBA_AUDIT_OBJECT` | Enregistrements concernant les objets de schémas |
| `DBA_AUDIT_SESSION` | Toutes les entrées de connexion et de déconnexion |
| `DBA_AUDIT_STATEMENT` | Enregistrements d'audit des instructions |
| `DBA_OBJ_AUDIT_OPTS` | Options d'audit actives sur les objets |
| `DBA_PRIV_AUDIT_OPTS` | Options d'audit actives sur les privilèges système |

**Exemples de requêtes :**

```sql
-- Lister toutes les opérations effectuées par un utilisateur
SELECT username, timestamp, action_name, obj_name
FROM dba_audit_trail
WHERE username = 'TD4'
ORDER BY timestamp;

-- Compter les connexions et déconnexions d'un utilisateur
SELECT username, COUNT(*) AS nb_sessions
FROM dba_audit_session
WHERE username = 'TD4'
GROUP BY username;

-- Vérifier les options d'audit actives sur les objets
SELECT object_name, sel, ins, upd, del
FROM dba_obj_audit_opts
WHERE owner = 'TD41';
```

---

## 4.4 Audit basé sur les données (Triggers)

L'audit standard identifie *quelles* opérations ont eu lieu, mais ne capture pas
les *valeurs* modifiées. Pour enregistrer les anciennes et nouvelles valeurs des données,
on utilise des **triggers d'audit**.

Le principe est simple : lorsqu'un utilisateur modifie une ligne, un trigger se déclenche
et insère un enregistrement dans une table d'audit dédiée.

**Exemple — Auditer les modifications sur une table `employes` :**

```sql
-- 1. Créer une table d'audit
CREATE TABLE audit_employes (
  id_audit     NUMBER GENERATED ALWAYS AS IDENTITY,
  utilisateur  VARCHAR2(50),
  date_action  DATE,
  action       VARCHAR2(10),
  ancien_salaire NUMBER,
  nouveau_salaire NUMBER
);

-- 2. Créer le trigger d'audit
CREATE OR REPLACE TRIGGER trg_audit_employes
AFTER UPDATE OF salaire ON employes
FOR EACH ROW
BEGIN
  INSERT INTO audit_employes (utilisateur, date_action, action, ancien_salaire, nouveau_salaire)
  VALUES (USER, SYSDATE, 'UPDATE', :OLD.salaire, :NEW.salaire);
END trg_audit_employes;
/
```

**Syntaxe générale d'un trigger d'audit :**

```sql
CREATE [OR REPLACE] TRIGGER nom_trigger
  { BEFORE | AFTER | INSTEAD OF }
  { INSERT | UPDATE [ OF nom_colonne [, ...] ] | DELETE }
  [ OR { INSERT | UPDATE [ OF nom_colonne [, ...] ] | DELETE } ]
  REFERENCING { [ OLD [AS] ancien ] | [ NEW [AS] nouveau ] }
  ON nom_table
  [ FOR EACH ROW ]
  [ WHEN (condition) ]
DECLARE
  /* déclarations */
BEGIN
  /* traitement */
  [ EXCEPTION ]
END;
/
```

| Pseudo-variable | Description |
|-----------------|-------------|
| `:OLD.colonne` | Valeur avant la modification (disponible dans UPDATE et DELETE) |
| `:NEW.colonne` | Valeur après la modification (disponible dans INSERT et UPDATE) |

---

## 4.5 Audit détaillé — Fine Grained Auditing (FGA)

Le **Fine Grained Auditing (FGA)** permet de définir des conditions précises pour
déclencher l'audit. Contrairement à l'audit standard qui audite toutes les occurrences
d'une opération, le FGA n'enregistre que les accès qui correspondent à un **prédicat défini**.

**Caractéristiques du FGA :**
- Surveille l'accès aux données **en fonction de leur contenu**
- Audite les opérations `SELECT`, `INSERT`, `UPDATE` et `DELETE`
- Peut être lié à une table ou à une vue
- Peut déclencher l'exécution d'une procédure PL/SQL lors d'un accès audité
- Géré exclusivement via le package `DBMS_FGA`

---

### 4.5.1 Le package DBMS_FGA

```sql
-- Accorder le droit d'utilisation du package à un administrateur
GRANT EXECUTE ON dbms_fga TO system;
```

| Sous-programme | Description |
|----------------|-------------|
| `ADD_POLICY` | Crée une stratégie d'audit à partir d'un prédicat (condition) |
| `DROP_POLICY` | Supprime une stratégie d'audit |
| `ENABLE_POLICY` | Active une stratégie d'audit existante |
| `DISABLE_POLICY` | Désactive temporairement une stratégie d'audit |

---

### 4.5.2 Créer une stratégie FGA

Une stratégie FGA se compose de deux parties :
- **Les critères d'audit** — la condition qui doit être vraie pour déclencher l'audit
- **L'action d'audit** — une procédure PL/SQL optionnelle exécutée lorsque la condition est vérifiée

**Exemple — Auditer les requêtes sur les salaires du département 10 :**

```sql
-- 1. Créer la table de journalisation
CREATE TABLE audit_messages (
  date_action  DATE,
  objet        VARCHAR2(100)
);

-- 2. Créer la procédure d'action
CREATE OR REPLACE PROCEDURE ins_audit_message(
  p_schema  VARCHAR2,
  p_table   VARCHAR2,
  p_policy  VARCHAR2
) IS
BEGIN
  INSERT INTO audit_messages VALUES (
    SYSDATE,
    p_schema || '.' || p_table || '.' || p_policy
  );
END;
/

-- 3. Créer la stratégie FGA
BEGIN
  DBMS_FGA.ADD_POLICY(
    object_schema    => 'hr',
    object_name      => 'employees',
    policy_name      => 'audit_salary_dept10',
    audit_condition  => 'department_id = 10',
    audit_column     => 'salary',
    handler_schema   => 'system',
    handler_module   => 'ins_audit_message',
    enable           => TRUE,
    statement_types  => 'SELECT'
  );
END;
/
```

**Paramètres de `ADD_POLICY` :**

| Paramètre | Description |
|-----------|-------------|
| `object_schema` | Schéma propriétaire de l'objet audité |
| `object_name` | Nom de la table ou vue auditée |
| `policy_name` | Nom unique de la stratégie |
| `audit_condition` | Prédicat SQL — si NULL, toutes les lignes sont auditées |
| `audit_column` | Colonne(s) sensible(s) — la stratégie ne se déclenche que si la requête accède à cette colonne |
| `handler_schema` | Schéma de la procédure d'action |
| `handler_module` | Nom de la procédure à exécuter lors du déclenchement |
| `enable` | `TRUE` pour activer immédiatement la stratégie |
| `statement_types` | Types d'instructions auditées : `SELECT`, `INSERT`, `UPDATE`, `DELETE` |

> **Règle importante :** La table ou vue auditée doit exister avant la création de la stratégie.
> Si la condition d'audit contient une erreur de syntaxe, une erreur sera générée lors de l'accès à l'objet.

---

### 4.5.3 Gérer les stratégies FGA

```sql
-- Désactiver temporairement une stratégie
BEGIN
  DBMS_FGA.DISABLE_POLICY(
    object_schema => 'hr',
    object_name   => 'employees',
    policy_name   => 'audit_salary_dept10'
  );
END;
/

-- Réactiver une stratégie
BEGIN
  DBMS_FGA.ENABLE_POLICY(
    object_schema => 'hr',
    object_name   => 'employees',
    policy_name   => 'audit_salary_dept10'
  );
END;
/

-- Supprimer définitivement une stratégie
BEGIN
  DBMS_FGA.DROP_POLICY(
    object_schema => 'hr',
    object_name   => 'employees',
    policy_name   => 'audit_salary_dept10'
  );
END;
/
```

---

### 4.5.4 Vues d'audit FGA

| Vue | Description |
|-----|-------------|
| `DBA_FGA_AUDIT_TRAIL` | Tous les événements d'audit détaillé enregistrés |
| `DBA_AUDIT_POLICIES` | Toutes les stratégies d'audit FGA de la base de données |
| `ALL_AUDIT_POLICIES` | Stratégies FGA pour les objets accessibles à l'utilisateur actuel |
| `USER_AUDIT_POLICIES` | Stratégies FGA sur les objets du schéma de l'utilisateur actuel |

**Interroger la trace d'audit FGA :**

```sql
-- Voir tous les accès enregistrés pour une stratégie donnée
SELECT session_id,
       timestamp,
       db_user,
       os_user,
       object_schema,
       object_name,
       policy_name,
       sql_text
FROM dba_fga_audit_trail
WHERE policy_name = 'AUDIT_SALARY_DEPT10'
ORDER BY timestamp DESC;
```

Exemple de résultat — l'instruction SQL complète est enregistrée :

```
SESSION_ID  TIMESTAMP   DB_USER  OBJECT_NAM  POLICY_NAME             SQL_TEXT
----------  ----------  -------  ----------  ----------------------  ------------------------------------------
186         13-AUG-01   HR       EMPLOYEES   AUDIT_SALARY_DEPT10     SELECT name, salary FROM employees WHERE department_id = 10
```

> Seules les requêtes qui accèdent à la colonne `salary` **et** dont la condition
> `department_id = 10` est vraie déclenchent un enregistrement.
> Une requête `SELECT name FROM employees WHERE department_id = 10` ne serait pas auditée
> car elle n'accède pas à la colonne sensible.

---

## Résumé

| Concept | Point clé |
|---------|-----------|
| Moindre privilège | N'accorder que les droits strictement nécessaires — règle de base de la sécurité |
| `O7_DICTIONARY_ACCESSIBILITY` | Positionner à `FALSE` pour protéger le dictionnaire de données |
| Rôle PUBLIC | Révoquer les packages dangereux (`UTL_FILE`, `UTL_HTTP`, `UTL_SMTP`) |
| `REMOTE_OS_AUTHENT` | Positionner à `FALSE` pour désactiver l'authentification OS à distance |
| Audit standard | Activé via `AUDIT_TRAIL` ; enregistre dans `SYS.AUD$` ou sur le système de fichiers |
| Commande `AUDIT` | Démarre un audit de commande, de privilège, d'objet ou de session |
| Commande `NOAUDIT` | Arrête un audit actif |
| `BY SESSION` / `BY ACCESS` | Granularité de l'enregistrement — une entrée par session ou par opération |
| Audit par trigger | Capture les valeurs avant/après modification — plus de flexibilité que l'audit standard |
| FGA (Fine Grained Auditing) | Audit conditionnel basé sur le contenu des données — géré via `DBMS_FGA` |
| `DBMS_FGA.ADD_POLICY` | Crée une stratégie d'audit FGA avec prédicat et action optionnelle |
| `DBA_AUDIT_TRAIL` | Vue principale de l'audit standard |
| `DBA_FGA_AUDIT_TRAIL` | Vue principale de l'audit FGA |

---

*Chapitre précédent : [Sécurité utilisateur ←](./chapter3_user_security.md)*
*Chapitre suivant : [Flashback →](./chapter5_flashback.md)*
