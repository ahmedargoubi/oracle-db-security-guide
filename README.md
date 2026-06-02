# Sécurité des Bases de Données Oracle — Documentation de cours

> Un guide structuré et progressif sur l'administration et la sécurité des bases de données Oracle,
> conçu pour les étudiants et les praticiens qui souhaitent acquérir des compétences solides et applicables.

---

## À propos de ce cours

Cette documentation couvre l'**Administration et la Sécurité des Bases de Données Oracle**,
un cours qui articule les connaissances théoriques sur l'architecture interne d'Oracle avec
une pratique concrète en SQL et PL/SQL.

Que vous configuriez votre première instance Oracle ou que vous exploriez des mécanismes
avancés d'audit et de récupération, le cours est organisé pour vous accompagner
progressivement à travers les concepts fondamentaux que tout administrateur de base de données
doit maîtriser.

Le contenu s'appuie sur les supports de cours, les travaux pratiques (TP) et des scénarios
réels d'administration. Chaque chapitre s'appuie sur le précédent — il est donc recommandé
de les lire dans l'ordre — mais chacun peut également servir de référence autonome.

---

## Objectifs du cours

À la fin de ce cours, vous serez capable de :

- Comprendre l'architecture interne d'un serveur Oracle et les interactions entre ses composants
- Gérer les structures de stockage physiques et logiques (tablespaces, fichiers de données, extents)
- Créer et administrer des utilisateurs Oracle avec des politiques de sécurité précises
- Mettre en place des stratégies d'audit pour surveiller et tracer l'activité de la base
- Utiliser les technologies Oracle Flashback pour récupérer des données après une erreur humaine ou une modification non souhaitée

---

## Table des matières

| # | Chapitre | Description |
|---|----------|-------------|
| 1 | [Architecture du serveur Oracle](./chapter1_oracle_architecture.md) | Instance, SGA, processus d'arrière-plan, démarrage et arrêt |
| 2 | [Structures de stockage](./chapter2_storage_structure.md) | Tablespaces, fichiers de données, segments, extents et blocs |
| 3 | [Sécurité utilisateur](./chapter3_user_security.md) | Création d'utilisateurs, privilèges, rôles et profils |
| 4 | [Sécurité et audit](./chapter4_security_audit.md) | Stratégies d'audit, Fine-Grained Auditing (FGA) et politiques de sécurité |
| 5 | [Flashback](./chapter5_flashback.md) | Flashback Query, Table, Database et technologies associées |

---

## Comment utiliser cette documentation

Chaque chapitre suit la même structure :

- **Introduction** — Une présentation rapide du sujet et de son importance
- **Concepts clés** — Des explications claires, accompagnées de tableaux et de schémas textuels
- **Exemples SQL / PL/SQL** — Des blocs de code pratiques, prêts à l'emploi
- **Résumé** — Les points essentiels à retenir à la fin de chaque section importante

> **Note :** Les chapitres 4 et 5 seront ajoutés progressivement au fil de l'avancement du cours.

---

## Prérequis

- Connaissances de base en SQL (SELECT, INSERT, UPDATE, DELETE)
- Familiarité avec les concepts des bases de données relationnelles (tables, clés, contraintes)
- Un environnement Oracle Database (11g, 12c ou ultérieur) pour la pratique

---

*Documentation réalisée dans le cadre du cours Administration des Bases de Données — Esprit School of Engineering.*
