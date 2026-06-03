# Chapitre 3 — Sécurité utilisateur

> Gérer qui peut accéder à la base de données, et ce qu'il est autorisé à faire une fois connecté,
> est au cœur de toute politique de sécurité Oracle. Ce chapitre couvre les comptes utilisateurs,
> les privilèges, les rôles et les profils — les quatre piliers de la sécurité utilisateur.

---

## Table des matières

- [3.1 Comptes utilisateurs](#31-comptes-utilisateurs)
  - [3.1.1 Attributs d'un compte utilisateur](#311-attributs-dun-compte-utilisateur)
  - [3.1.2 Comptes prédéfinis : SYS et SYSTEM](#312-comptes-prédéfinis--sys-et-system)
  - [3.1.3 Méthodes d'authentification](#313-méthodes-dauthentification)
  - [3.1.4 Créer un compte utilisateur](#314-créer-un-compte-utilisateur)
- [3.2 Privilèges](#32-privilèges)
  - [3.2.1 Privilèges système](#321-privilèges-système)
  - [3.2.2 Privilèges objet](#322-privilèges-objet)
  - [3.2.3 Accorder et révoquer des privilèges](#323-accorder-et-révoquer-des-privilèges)
  - [3.2.4 Options d'administration](#324-options-dadministration)
- [3.3 Rôles](#33-rôles)
  - [3.3.1 Avantages des rôles](#331-avantages-des-rôles)
  - [3.3.2 Rôles prédéfinis](#332-rôles-prédéfinis)
  - [3.3.3 Créer et gérer un rôle](#333-créer-et-gérer-un-rôle)
- [3.4 Profils](#34-profils)
  - [3.4.1 Gestion des mots de passe](#341-gestion-des-mots-de-passe)
  - [3.4.2 Gestion des ressources](#342-gestion-des-ressources)
  - [3.4.3 Créer un profil](#343-créer-un-profil)
- [Résumé](#résumé)

---

## 3.1 Comptes utilisateurs

### 3.1.1 Attributs d'un compte utilisateur

Chaque compte utilisateur dans Oracle est défini par un ensemble d'attributs qui contrôlent
son identité, ses droits d'accès et ses limites de ressources.

| Attribut | Description |
|----------|-------------|
| **Nom d'utilisateur** | Identifiant unique dans la base de données |
| **Méthode d'authentification** | Comment l'utilisateur prouve son identité (mot de passe, OS, etc.) |
| **Tablespace par défaut** | Espace de stockage attribué par défaut pour ses objets |
| **Tablespace temporaire** | Espace utilisé pour les tris et opérations temporaires |
| **Quotas** | Limites d'espace sur un ou plusieurs tablespaces |
| **Profil** | Ensemble de règles de sécurité et de limites de ressources |
| **Statut du compte** | Actif, verrouillé ou expiré |

> **Note :** Seuls le nom d'utilisateur et la méthode d'authentification sont obligatoires
> à la création. Les autres attributs prennent des valeurs par défaut si non spécifiés.

Pour consulter les informations des utilisateurs existants :

```sql
SELECT username, account_status, default_tablespace, profile
FROM dba_users
ORDER BY username;
```

---

### 3.1.2 Comptes prédéfinis : SYS et SYSTEM

Oracle crée deux comptes administrateurs lors de l'installation de la base :

**Compte SYS :**
- Possède le rôle d'administrateur de base de données (DBA) avec tous les privilèges `ADMIN OPTION`
- Requis pour le démarrage, l'arrêt et certaines opérations de maintenance
- Propriétaire du **dictionnaire de données** — les tables système ne doivent jamais être modifiées directement
- Doit se connecter exclusivement avec le privilège `AS SYSDBA`

**Compte SYSTEM :**
- Reçoit également le rôle DBA
- Utilisé pour les tâches d'administration courantes qui ne nécessitent pas les droits de SYS

> **Bonne pratique :** Ne jamais utiliser SYS ou SYSTEM pour les tâches quotidiennes.
> Créez des comptes dédiés avec les privilèges strictement nécessaires.

---

### 3.1.3 Méthodes d'authentification

Oracle supporte plusieurs modes d'authentification :

**Authentification par mot de passe (base de données)**

Le mode le plus courant. Un mot de passe est associé au compte et vérifié à chaque connexion.
Il est possible de forcer l'expiration immédiate du mot de passe à la première connexion.

```sql
CREATE USER alice IDENTIFIED BY monmotdepasse;
```

**Authentification par le système d'exploitation (OS)**

L'OS authentifie l'utilisateur avant qu'il se connecte à Oracle. Requiert le privilège `SYSDBA`.

```sql
CONNECT / AS SYSDBA
```

**Authentification externe**

Oracle délègue l'authentification à un service externe (Kerberos, RADIUS, ou le service natif Windows).
Sous Unix, le préfixe `OPS$` (valeur du paramètre `OS_AUTHENT_PREFIX`) est utilisé :

```sql
CREATE USER ops$alice IDENTIFIED EXTERNALLY;
```

**Authentification globale**

Identifie les utilisateurs via Oracle Internet Directory.

```sql
CREATE USER alice IDENTIFIED GLOBALLY AS 'cn=alice,ou=dept,dc=example,dc=com';
```

---

### 3.1.4 Créer un compte utilisateur

**Syntaxe complète :**

```sql
CREATE USER nom_utilisateur
  IDENTIFIED BY mot_de_passe
  [ DEFAULT TABLESPACE nom_tablespace ]
  [ TEMPORARY TABLESPACE nom_tablespace ]
  [ QUOTA { valeur [K|M] | UNLIMITED } ON nom_tablespace [, ...] ]
  [ PROFILE nom_profil ]
  [ PASSWORD EXPIRE ]
  [ ACCOUNT { LOCK | UNLOCK } ];
```

**Exemple pratique :**

```sql
-- Créer un utilisateur avec tablespace et quota définis
CREATE USER scott
  IDENTIFIED BY pwdscott
  DEFAULT TABLESPACE users
  QUOTA UNLIMITED ON users
  PASSWORD EXPIRE;
```

L'option `PASSWORD EXPIRE` force l'utilisateur à changer son mot de passe dès sa première connexion.

**Modifier un compte existant :**

```sql
-- Changer le mot de passe
ALTER USER scott IDENTIFIED BY nouveaumotdepasse;

-- Verrouiller un compte
ALTER USER scott ACCOUNT LOCK;

-- Déverrouiller un compte
ALTER USER scott ACCOUNT UNLOCK;

-- Attribuer un quota sur un tablespace
ALTER USER scott QUOTA 50M ON users;
```

**Supprimer un compte :**

```sql
-- Suppression simple (échoue si l'utilisateur possède des objets)
DROP USER scott;

-- Suppression avec tous ses objets
DROP USER scott CASCADE;
```

---

## 3.2 Privilèges

Un **privilège** est le droit d'exécuter un type particulier d'instruction SQL ou d'accéder
à l'objet d'un autre utilisateur. Il en existe deux catégories.

### 3.2.1 Privilèges système

Les privilèges système autorisent un utilisateur à effectuer certaines opérations sur la base
de données dans son ensemble — créer des tables, des utilisateurs, des sessions, etc.

Voici les principaux privilèges système regroupés par catégorie :

**Sessions**

| Privilège | Description |
|-----------|-------------|
| `CREATE SESSION` | Connexion à la base de données |
| `ALTER SESSION` | Modification de l'état de connexion |
| `RESTRICTED SESSION` | Connexion en mode démarrage restreint |

**Tables**

| Privilège | Description |
|-----------|-------------|
| `CREATE TABLE` | Créer des tables dans son propre schéma |
| `CREATE ANY TABLE` | Créer des tables dans n'importe quel schéma |
| `DROP ANY TABLE` | Supprimer ou tronquer n'importe quelle table |
| `SELECT ANY TABLE` | Interroger n'importe quelle table |
| `INSERT / UPDATE / DELETE ANY TABLE` | Modifier les données de n'importe quelle table |

**Procédures et fonctions**

| Privilège | Description |
|-----------|-------------|
| `CREATE PROCEDURE` | Créer des procédures et fonctions dans son schéma |
| `CREATE ANY PROCEDURE` | Créer dans n'importe quel schéma |
| `EXECUTE ANY PROCEDURE` | Exécuter n'importe quelle procédure ou fonction |

**Utilisateurs et rôles**

| Privilège | Description |
|-----------|-------------|
| `CREATE ANY USER` | Créer des utilisateurs |
| `ALTER USER` | Modifier le profil des autres utilisateurs |
| `DROP USER` | Supprimer un utilisateur |
| `CREATE ROLE` | Créer un rôle |
| `GRANT ANY PRIVILEGE` | Accorder n'importe quel privilège système |

---

### 3.2.2 Privilèges objet

Les privilèges objet permettent à un utilisateur d'effectuer une action particulière sur
un objet spécifique (table, vue, séquence, procédure, etc.).

| Privilège objet | SQL correspondant |
|-----------------|-------------------|
| `SELECT` | `SELECT ... FROM objet` (table, vue, séquence) |
| `INSERT` | `INSERT INTO objet` (table ou vue) |
| `UPDATE` | `UPDATE objet` (table ou vue) |
| `DELETE` | `DELETE FROM objet` (table ou vue) |
| `ALTER` | `ALTER TABLE` ou `ALTER SEQUENCE` |
| `INDEX` | `CREATE INDEX ON objet` |
| `EXECUTE` | Exécuter une procédure ou une fonction |
| `REFERENCES` | Créer une contrainte de clé étrangère vers cet objet |

> Sans permission explicite, un utilisateur n'a accès qu'à ses propres objets.

---

### 3.2.3 Accorder et révoquer des privilèges

**Accorder un privilège — `GRANT`**

```sql
-- Privilège système
GRANT CREATE SESSION TO hr;
GRANT CREATE TABLE, CREATE VIEW TO alice;

-- Privilège objet
GRANT SELECT, UPDATE(salary) ON emp TO scott;
GRANT SELECT ON hr.employees TO alice;
```

**Révoquer un privilège — `REVOKE`**

```sql
-- Révoquer un privilège système
REVOKE CREATE SESSION FROM hr;

-- Révoquer un privilège objet
REVOKE SELECT, UPDATE ON emp FROM scott;
```

**Consulter les privilèges accordés :**

```sql
-- Privilèges système d'un utilisateur
SELECT privilege FROM dba_sys_privs WHERE grantee = 'HR';

-- Privilèges objet d'un utilisateur
SELECT owner, table_name, privilege FROM dba_tab_privs WHERE grantee = 'HR';
```

---

### 3.2.4 Options d'administration

Ces options permettent de **déléguer** la gestion des privilèges à d'autres utilisateurs.

**`WITH ADMIN OPTION`** (pour les privilèges système)

L'utilisateur qui reçoit le privilège peut à son tour l'accorder à d'autres.

```sql
GRANT CREATE SESSION TO hr WITH ADMIN OPTION;

-- HR peut maintenant accorder ce privilège
CONNECT hr/hr
GRANT CREATE SESSION TO scott;
```

> **Attention :** Si on révoque le privilège à HR, Scott **conserve** le privilège —
> la révocation n'est pas en cascade pour les privilèges système.

**`WITH GRANT OPTION`** (pour les privilèges objet)

```sql
GRANT SELECT ON employees TO stock WITH GRANT OPTION;

-- STOCK peut accorder à son tour
CONNECT stock/stock
GRANT SELECT ON employees TO user1;
```

> **Attention :** Si on révoque le privilège objet à STOCK, user1 **perd aussi** son accès —
> la révocation **est** en cascade pour les privilèges objet.

---

## 3.3 Rôles

Un **rôle** est un regroupement nommé de privilèges. Au lieu d'accorder individuellement
chaque privilège à chaque utilisateur, on regroupe les privilèges dans un rôle,
puis on attribue ce rôle aux utilisateurs concernés.

```
Privilèges  →  Rôle  →  Utilisateurs
```

### 3.3.1 Avantages des rôles

**Gestion simplifiée** — Accorder le même ensemble de privilèges à plusieurs utilisateurs
en une seule opération plutôt qu'utilisateur par utilisateur.

**Gestion dynamique** — Si les privilèges d'un rôle sont modifiés, tous les utilisateurs
qui possèdent ce rôle bénéficient automatiquement et immédiatement des changements.

**Disponibilité sélective** — Les rôles peuvent être activés ou désactivés temporairement,
ce qui permet de restreindre l'accès sans révoquer définitivement les privilèges.

---

### 3.3.2 Rôles prédéfinis

Oracle fournit plusieurs rôles prêts à l'emploi :

| Rôle | Privilèges inclus |
|------|------------------|
| `CONNECT` | `CREATE SESSION`, `CREATE TABLE`, `CREATE VIEW`, `CREATE SYNONYM`, `CREATE SEQUENCE`, `CREATE DATABASE LINK`, `CREATE CLUSTER`, `ALTER SESSION` |
| `RESOURCE` | `CREATE TABLE`, `CREATE PROCEDURE`, `CREATE SEQUENCE`, `CREATE TRIGGER`, `CREATE TYPE`, `CREATE CLUSTER`, `CREATE INDEXTYPE`, `CREATE OPERATOR` |
| `DBA` | La plupart des privilèges système et plusieurs autres rôles — **à ne jamais accorder aux non-administrateurs** |
| `SCHEDULER_ADMIN` | `CREATE ANY JOB`, `CREATE JOB`, `EXECUTE ANY CLASS`, `EXECUTE ANY PROGRAM`, `MANAGE SCHEDULER` |
| `SELECT_CATALOG_ROLE` | Plus de 1 600 privilèges objet sur le dictionnaire de données (aucun privilège système) |

---

### 3.3.3 Créer et gérer un rôle

**Créer un rôle simple :**

```sql
CREATE ROLE role_developpeur;

-- Ajouter des privilèges au rôle
GRANT CREATE SESSION, CREATE TABLE, CREATE PROCEDURE TO role_developpeur;

-- Attribuer le rôle à un utilisateur
GRANT role_developpeur TO alice;
```

**Créer un rôle protégé par mot de passe :**

```sql
CREATE ROLE superdba IDENTIFIED BY superpasse;

-- Accorder des privilèges au rôle
GRANT ALTER ANY TABLE TO superdba;
GRANT ALTER ANY PROCEDURE TO superdba;
GRANT dba TO superdba WITH ADMIN OPTION;

-- Affecter le rôle à un utilisateur
GRANT superdba TO stock WITH ADMIN OPTION;
```

L'utilisateur devra activer le rôle avec son mot de passe :

```sql
SET ROLE superdba IDENTIFIED BY superpasse;
```

**Révoquer un rôle :**

```sql
REVOKE role_developpeur FROM alice;
```

**Supprimer un rôle :**

```sql
DROP ROLE role_developpeur;
```

**Consulter les rôles :**

```sql
-- Tous les rôles de la base
SELECT role FROM dba_roles;

-- Rôles accordés à un utilisateur
SELECT granted_role FROM dba_role_privs WHERE grantee = 'ALICE';

-- Privilèges système d'un rôle
SELECT privilege FROM dba_sys_privs WHERE grantee = 'ROLE_DEVELOPPEUR';
```

---

## 3.4 Profils

Un **profil** est un ensemble de règles nommé qui s'applique à un compte utilisateur.
Il contrôle deux aspects complémentaires :

- **La politique de mots de passe** — durée de vie, historique, complexité, verrouillage
- **Les limites de ressources** — sessions simultanées, temps CPU, temps d'inactivité

> Un seul profil est affecté à un utilisateur à un instant donné.
> Si aucun profil n'est spécifié, le profil `DEFAULT` s'applique.

Pour activer la gestion des ressources, le paramètre d'initialisation doit être activé :

```sql
ALTER SYSTEM SET resource_limit = TRUE;
```

---

### 3.4.1 Gestion des mots de passe

**Verrouillage du compte**

| Paramètre | Description |
|-----------|-------------|
| `FAILED_LOGIN_ATTEMPTS` | Nombre d'échecs de connexion avant verrouillage du compte |
| `PASSWORD_LOCK_TIME` | Durée du verrouillage (en jours) après le nombre d'échecs atteint |

**Durée de vie et expiration**

| Paramètre | Description |
|-----------|-------------|
| `PASSWORD_LIFE_TIME` | Durée de vie du mot de passe en jours avant expiration |
| `PASSWORD_GRACE_TIME` | Période de grâce (en jours) pour changer le mot de passe après expiration |

**Historique des mots de passe**

| Paramètre | Description |
|-----------|-------------|
| `PASSWORD_REUSE_TIME` | Nombre de jours pendant lesquels un mot de passe ne peut pas être réutilisé |
| `PASSWORD_REUSE_MAX` | Nombre de changements de mot de passe requis avant de pouvoir réutiliser un ancien |

**Vérification de la complexité**

| Paramètre | Description |
|-----------|-------------|
| `PASSWORD_VERIFY_FUNCTION` | Fonction PL/SQL (appartenant à SYS) qui valide la complexité avant d'accepter un mot de passe |

La fonction de vérification doit :
- Appartenir à l'utilisateur `SYS`
- Retourner une valeur booléenne (`TRUE` si le mot de passe est valide, `FALSE` sinon)

```sql
-- Exemple de fonction de vérification
CREATE OR REPLACE FUNCTION verif_password(
  p_username VARCHAR2,
  p_password VARCHAR2
) RETURN BOOLEAN IS
BEGIN
  -- Le mot de passe doit faire plus de 6 caractères
  IF LENGTH(p_password) <= 6 THEN
    RETURN FALSE;
  END IF;

  -- Le mot de passe ne doit pas être identique au nom d'utilisateur
  IF NLS_LOWER(p_password) = NLS_LOWER(p_username) THEN
    RETURN FALSE;
  END IF;

  RETURN TRUE;
END verif_password;
/
```

---

### 3.4.2 Gestion des ressources

Les profils permettent aussi de limiter la consommation des ressources système :

| Paramètre | Description |
|-----------|-------------|
| `SESSIONS_PER_USER` | Nombre maximal de sessions simultanées pour un compte |
| `CPU_PER_SESSION` | Temps CPU total (en centisecondes) autorisé par session |
| `CPU_PER_CALL` | Temps CPU (en centisecondes) autorisé pour exécuter une seule requête SQL |
| `CONNECT_TIME` | Durée maximale d'une connexion (en minutes) |
| `IDLE_TIME` | Temps d'inactivité maximal (en minutes) avant déconnexion automatique |

---

### 3.4.3 Créer un profil

**Syntaxe :**

```sql
CREATE PROFILE nom_profil LIMIT
  [ paramètre_ressource  valeur ]
  [ paramètre_mot_de_passe  valeur ];
```

Les valeurs possibles sont `entier`, `UNLIMITED` ou `DEFAULT`.

**Exemple complet :**

```sql
CREATE PROFILE profil_developpeur LIMIT
  -- Ressources
  SESSIONS_PER_USER       2
  CPU_PER_SESSION         10000
  CONNECT_TIME            480
  IDLE_TIME               60
  -- Mots de passe
  FAILED_LOGIN_ATTEMPTS   3
  PASSWORD_LOCK_TIME      1/24        -- 1 heure
  PASSWORD_LIFE_TIME      90
  PASSWORD_GRACE_TIME     7
  PASSWORD_REUSE_TIME     365
  PASSWORD_VERIFY_FUNCTION verif_password;
```

**Attribuer un profil à un utilisateur :**

```sql
-- À la création
CREATE USER alice IDENTIFIED BY mdp PROFILE profil_developpeur;

-- Sur un utilisateur existant
ALTER USER alice PROFILE profil_developpeur;
```

**Modifier un profil existant :**

```sql
ALTER PROFILE profil_developpeur LIMIT
  FAILED_LOGIN_ATTEMPTS 5
  IDLE_TIME 30;
```

**Supprimer un profil :**

```sql
-- Les utilisateurs du profil basculent automatiquement sur DEFAULT
DROP PROFILE profil_developpeur;

-- Supprimer même si des utilisateurs l'utilisent encore
DROP PROFILE profil_developpeur CASCADE;
```

**Consulter les profils :**

```sql
-- Tous les profils et leurs paramètres
SELECT profile, resource_name, limit
FROM dba_profiles
ORDER BY profile, resource_name;

-- Profil attribué à un utilisateur
SELECT username, profile FROM dba_users WHERE username = 'ALICE';
```

---

## Résumé

| Concept | Point clé |
|---------|-----------|
| Compte utilisateur | Identifié par un nom unique ; possède un tablespace, un profil et un statut |
| SYS / SYSTEM | Comptes administrateurs prédéfinis — utilisation réservée aux tâches d'administration |
| Privilège système | Droit d'effectuer une opération sur la base (CREATE TABLE, CREATE SESSION…) |
| Privilège objet | Droit d'accès à un objet précis (SELECT sur une table, EXECUTE sur une procédure…) |
| `GRANT` / `REVOKE` | Commandes d'attribution et de révocation des privilèges |
| `WITH ADMIN OPTION` | Permet de déléguer un privilège système — révocation non cascadée |
| `WITH GRANT OPTION` | Permet de déléguer un privilège objet — révocation cascadée |
| Rôle | Regroupement de privilèges — simplifie la gestion et permet une activation dynamique |
| Rôles prédéfinis | `CONNECT`, `RESOURCE`, `DBA`, `SELECT_CATALOG_ROLE`… |
| Profil | Règles de sécurité et limites de ressources appliquées à un compte utilisateur |
| `DBA_USERS` | Vue principale pour consulter les comptes utilisateurs |
| `DBA_SYS_PRIVS` | Vue des privilèges système accordés aux utilisateurs et rôles |
| `DBA_TAB_PRIVS` | Vue des privilèges objet |
| `DBA_ROLE_PRIVS` | Vue des rôles accordés aux utilisateurs |
| `DBA_PROFILES` | Vue des paramètres de chaque profil |

---

*Chapitre précédent : [Structures de stockage ←](./chapter2_storage_structure.md)*
*Chapitre suivant : [Sécurité et audit →](./chapter4_security_audit.md)*
