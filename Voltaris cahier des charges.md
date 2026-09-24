# Cahier des charges
## Système d'information de Voltaris — infrastructure locale et Microsoft 365

---

### 1. Contexte

Voltaris est une PME de **14 salariés** qui conçoit et vend des objets connectés pour la gestion de l'énergie. Créée à partir d'une petite équipe, elle a grandi rapidement, mais son informatique est restée basique.

- **Activité :** conception, assemblage et vente d'objets connectés.
- **Organisation :** direction (2), bureau d'études / R&D (4), production / atelier (3), commercial et marketing (2), service après-vente (2), administration (1). Le rôle de responsable informatique est assuré par un collaborateur du pôle technique.
- **Outils actuels :** vieille messagerie, serveur de fichiers local vieillissant, outils personnels.
- **Difficultés :** outils dispersés, aucune sécurité centralisée, pas de suivi des demandes, données sensibles (plans R&D) mal protégées.

### 2. Besoin

Voltaris a besoin de centraliser ses outils, de sécuriser ses accès et ses données, et de faciliter le travail de ses équipes, tout en gardant ses **données sensibles maîtrisées en interne**.

**Problématique :** Comment doter Voltaris d'un système d'information centralisé, collaboratif et sécurisé, qui protège ses données sensibles en local tout en offrant à ses équipes des outils de travail modernes ?

### 3. Objectif

Mettre en place un **système d'information hybride** :

- une **infrastructure locale** (serveur Active Directory, serveur de fichiers, serveur applicatif, supervision) qui héberge l'identité, les données sensibles et les applications de l'entreprise ;
- la **suite Microsoft 365** ajoutée par-dessus pour la messagerie, la collaboration, la gestion des postes et la sécurité.

Les deux sont reliés par une synchronisation d'annuaire (Active Directory local vers Entra ID). L'objectif est clair, réaliste et lié à la problématique.

### 4. Utilisateurs

| Utilisateur | Attente |
|---|---|
| Direction | Suivre l'activité de l'entreprise |
| Bureau d'études / R&D | Collaborer et protéger les documents techniques sensibles |
| Production / Atelier | Accéder aux procédures et documents de fabrication |
| Commercial / SAV | Gérer les clients et suivre les demandes |
| Administration | Gérer les documents administratifs de façon sécurisée |
| Responsable informatique | Gérer les comptes, les accès et la sécurité |
| Clients | Contacter l'entreprise et le SAV |

### 5. Fonctionnalités

Les fonctionnalités sont priorisées avec la méthode **MoSCoW**.

| Priorité | Signification |
|---|---|
| Must | Indispensable |
| Should | Important |
| Could | Optionnel |
| Won't | Pas prévu maintenant |

**Must (indispensable)**
- Annuaire et comptes centralisés, synchronisés (Active Directory local + Entra ID)
- Authentification sécurisée par MFA via l'application Microsoft Authenticator
- Messagerie et agendas partagés (Exchange Online / Outlook)
- Communication et réunions en ligne (Teams)
- Stockage local des données sensibles (serveur de fichiers) et partage collaboratif (SharePoint / OneDrive)
- Gestion des demandes et des incidents par tickets ITIL (GLPI)
- Sauvegarde des données avec test de restauration
- Site vitrine accessible depuis Internet
- Application web de suivi des retours SAV (base de données + service externe)
- Supervision de l'infrastructure (Zabbix)

**Should (important)**
- Gestion et sécurisation des postes de travail (Intune)
- Protection des données et contre les menaces (Defender, Purview)
- Gestion des droits par service (groupes de sécurité)
- Planification et suivi du projet (Planner)

**Could (optionnel)**
- Accès distant sécurisé aux ressources locales (VPN)
- Intranet d'entreprise (SharePoint)
- Prise de rendez-vous SAV en ligne (Bookings)

**Won't (pas prévu maintenant)**
- Téléphonie Teams Phone
- Assistant Copilot
- Portail client en ligne

*On développe d'abord les Must, puis les Should, et enfin les Could si le temps le permet.*

### 6. Contraintes

Les règles que le projet doit respecter :

- **Données :** les données sensibles restent hébergées en local ; seul le travail collaboratif s'appuie sur le cloud.
- **Budget :** rester dans un budget adapté à une PME (licences Microsoft 365 Business Premium + solutions open source) ; versions d'évaluation / éducation pour le projet.
- **Sécurité :** contrôler les accès, gérer les droits par service, sécuriser les comptes (MFA).
- **Légal :** respecter le RGPD.
- **Continuité de service :** ne pas interrompre l'activité pendant la mise en place ; sauvegarde et restauration testées.
- **Technique :** infrastructure sur plusieurs machines, gestion des tickets selon ITIL, site accessible depuis Internet, application reliée à une base de données et à un service externe, supervision de l'infrastructure.
- **Simplicité :** limiter le nombre de logiciels ; la suite Microsoft 365 couvre à elle seule la messagerie, la collaboration, la gestion des postes et la sécurité.

### 7. Architecture retenue

L'infrastructure repose sur **plusieurs machines** :

- un **serveur Windows** (Active Directory, DNS, DHCP, dossiers partagés) qui gère l'identité et le stockage local ;
- un **serveur applicatif** hébergeant le site vitrine, l'application web SAV et sa base de données, ainsi que l'outil GLPI ;
- un **serveur de supervision** (Zabbix) ;
- une **sauvegarde** des serveurs, complétée par la rétention Microsoft 365.

La suite **Microsoft 365** (Exchange Online, Teams, SharePoint, OneDrive, Entra ID, Intune, Defender) est reliée à l'Active Directory local par synchronisation d'annuaire.
