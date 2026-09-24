# Voltaris — Schéma d'infrastructure et plan d'adressage

**Projet :** mise en place d'une infrastructure informatique locale, sans cloud
**Entreprise :** Voltaris — PME de 14 salariés, conception et vente d'objets connectés pour la gestion de l'énergie
**Domaine Active Directory :** `voltaris.local`
**Nom de domaine public :** `voltaris.fr`

---

## 1. Vue d'ensemble

L'infrastructure est découpée en **trois zones étanches** :

| Zone | Contenu | Accessible depuis Internet |
|---|---|---|
| **LAN interne** | Les 14 postes, les 5 serveurs internes, le Wi-Fi | Non |
| **DMZ** | Le site vitrine, l'application SAV et sa base de données | Oui (via NAT statique) |
| **Externe** | Clients, service externe (API transporteur), télétravail | — |

La règle structurante : **la DMZ ne peut jamais remonter vers le LAN**. Un serveur web compromis n'atteint donc pas les plans R&D, ce qui répond directement à l'exigence du cahier des charges de conserver les données sensibles en interne.

---

## 2. Schéma d'infrastructure

```
┌──────────────────────── ZONE EXTERNE · INTERNET · 198.51.100.0/24 ────────────────────────┐
│                                                                                           │
│   PC-INTERNET          SRV-API-EXT           SRV-DNS-PUB          R-HOME + PC-TELETRAVAIL │
│   198.51.100.10        198.51.100.20         198.51.100.53        198.51.100.100          │
│   client / visiteur    API transporteur      zone voltaris.fr     tunnel VPN IPsec        │
│                        (service externe)                          LAN 192.168.200.0/24    │
│                                                                                           │
└─────────────────────────────────────────┬─────────────────────────────────────────────────┘
                                          │
                                     ┌────┴─────┐
                                     │  R-FAI   │   simule le fournisseur d'accès
                                     └────┬─────┘
                                          │  203.0.113.0/24
                                          │
                                   ┌──────┴───────┐
                                   │   R-EDGE     │   ISR 2911
                                   │  NAT / PAT   │   Gi0/0  203.0.113.2    (nat outside)
                                   │  Pare-feu    │   Gi0/1  10.0.0.1/30    (nat inside)
                                   │  VPN IPsec   │   Gi0/2  192.168.50.254 (nat inside)
                                   └──┬────────┬──┘
                                      │        │
                       10.0.0.0/30    │        │  Gi0/2
                                      │        └──────────────┐
                                      │                       │
                             ┌────────┴────────┐    ┌─────────┴──────────────────────────┐
                             │    SW-CORE      │    │  DMZ · VLAN 50 · 192.168.50.0/24   │
                             │ Catalyst 3650   │    │                                    │
                             │ ip routing      │    │  SW-DMZ (2960)                     │
                             │ SVI 10/20/30/   │    │   ├── SRV-WEB01   192.168.50.10    │
                             │     40/99       │    │   │   site vitrine voltaris.fr     │
                             │ relais DHCP     │    │   ├── SRV-APP01   192.168.50.20    │
                             └───┬─────────┬───┘    │   │   appli web suivi SAV          │
                                 │         │        │   └── SRV-SQL01   192.168.50.30    │
                    trunk 802.1Q │         │        │       base de données MariaDB      │
                                 │         │        └────────────────────────────────────┘
                                 │         │                     ╳
                  ┌──────────────┘         └──────────────┐   la DMZ n'atteint
                  │                                       │   jamais le LAN
        ┌─────────┴─────────┐                   ┌─────────┴─────────┐
        │    SW-ACC-01      │                   │      SW-SRV       │
        │      (2960)       │                   │      (2960)       │
        └─────────┬─────────┘                   └─────────┬─────────┘
                  │                                       │
   ┌──────────────┼──────────────┬─────────────┐          │
   │              │              │             │          │
┌──┴───────┐ ┌────┴─────┐ ┌──────┴────┐ ┌──────┴────┐  ┌──┴──────────────────────────────┐
│ VLAN 10  │ │ VLAN 20  │ │  VLAN 40  │ │  VLAN 99  │  │ VLAN 30 · SERVEURS              │
│ BUREAUX  │ │ ÉTUDES   │ │  WI-FI    │ │   MGMT    │  │ 192.168.30.0/24                 │
│ .10.0/24 │ │ .20.0/24 │ │ .40.0/24  │ │ .99.0/24  │  │                                 │
│          │ │          │ │           │ │           │  │ SRV-AD01    .30.10              │
│ Direction│ │ R&D   5p │ │ AP-WIFI   │ │ PC-SI-01  │  │  AD DS · DNS · DHCP · GPO       │
│      2p  │ │ Atelier  │ │ portables │ │ resp. SI  │  │ SRV-FILE01  .30.20              │
│ Admin 2p │ │       2p │ │ salariés  │ │           │  │  partages SMB/DFS · WSUS        │
│ Comm/SAV │ │ 2 bancs  │ │           │ │           │  │ SRV-MAIL01  .30.30              │
│      3p  │ │ IoT      │ │           │ │           │  │  Exchange · messagerie/agendas  │
│          │ │          │ │           │ │           │  │ SRV-ADM01   .30.40              │
│  7 POSTES│ │ 7 POSTES │ │           │ │           │  │  GLPI · Zabbix · Syslog · NTP   │
└──────────┘ └──────────┘ └───────────┘ └───────────┘  │ SRV-BKP01   .30.50              │
                                                       │  sauvegarde + restauration      │
   └──────────────── LAN INTERNE · voltaris.local ─────┴─────────────────────────────────┘
```

### Utilisateurs par service

| Service | Effectif | VLAN | Groupe Active Directory |
|---|---|---|---|
| Direction | 2 | 10 | `G_Direction` |
| Administratif et comptabilité | 2 | 10 | `G_Admin` |
| Commercial et SAV | 3 | 10 | `G_Commercial` |
| Bureau d'études R&D | 5 | 20 | `G_Etudes` |
| Atelier / assemblage | 2 | 20 | `G_Atelier` |
| **Total** | **14** | | |

Le responsable informatique fait partie de l'un de ces services et appartient en plus au groupe `G_ServiceInfo`.

### Services par serveur

| Serveur | Services rendus | Exigence du cahier des charges |
|---|---|---|
| SRV-AD01 | Active Directory, DNS, DHCP, stratégies de groupe | Annuaire et comptes centralisés |
| SRV-FILE01 | Partages SMB/DFS, WSUS | Serveur de fichiers, mises à jour |
| SRV-MAIL01 | Exchange Server, agendas partagés | Messagerie et agendas |
| SRV-ADM01 | GLPI (tickets ITIL), Zabbix, Syslog, NTP | Tickets, supervision |
| SRV-BKP01 | Sauvegarde nocturne, test de restauration | Sauvegarde locale testée |
| SRV-WEB01 | Site vitrine | Site accessible depuis Internet |
| SRV-APP01 | Application de suivi des retours SAV | Application web SAV |
| SRV-SQL01 | Base de données de l'application | Application reliée à une base |

---

## 3. Plan d'adressage — les VLAN

Convention : la passerelle est **toujours en `.254`**, le pool DHCP commence **toujours en `.100`**.

| VLAN | Nom | Réseau | Masque | Passerelle | Adressage | Plage DHCP |
|---|---|---|---|---|---|---|
| 10 | BUREAUX | 192.168.10.0 | 255.255.255.0 | 192.168.10.254 | DHCP | .100 → .150 |
| 20 | ETUDES | 192.168.20.0 | 255.255.255.0 | 192.168.20.254 | DHCP | .100 → .150 |
| 30 | SERVEURS | 192.168.30.0 | 255.255.255.0 | 192.168.30.254 | Statique | — |
| 40 | WIFI | 192.168.40.0 | 255.255.255.0 | 192.168.40.254 | DHCP | .100 → .200 |
| 50 | DMZ | 192.168.50.0 | 255.255.255.0 | 192.168.50.254 | Statique | — |
| 99 | MANAGEMENT | 192.168.99.0 | 255.255.255.0 | 192.168.99.254 | Statique | — |

### Liens d'interconnexion et adresses publiques

| Liaison | Réseau | Détail |
|---|---|---|
| Segment Internet | 198.51.100.0/24 | Passerelle R-FAI : 198.51.100.254 |
| R-FAI ↔ R-EDGE | 203.0.113.0/24 | R-FAI `.1` · R-EDGE `.2` |
| R-EDGE ↔ SW-CORE | 10.0.0.0/30 | R-EDGE `.1` · SW-CORE `.2` |
| LAN du télétravailleur | 192.168.200.0/24 | Passerelle R-HOME : 192.168.200.254 |
| Publication du site | 203.0.113.10 | NAT statique → 192.168.50.10 |
| Publication du SAV | 203.0.113.20 | NAT statique → 192.168.50.20 |

> Le réseau public est en `/24` et non en `/30` : les adresses publiées `.10` et `.20` doivent appartenir au même réseau que l'interface WAN de R-EDGE, sinon le NAT statique ne répond pas dans Packet Tracer.

---

## 4. Adressage des équipements réseau

| Équipement | Modèle | Rôle | VLAN admin | IP d'administration |
|---|---|---|---|---|
| R-FAI | ISR 2911 | Simule le fournisseur d'accès | — | 203.0.113.1 / 198.51.100.254 |
| R-EDGE | ISR 2911 | NAT/PAT, ACL, DMZ, VPN | 99 | 192.168.99.1 |
| SW-CORE | Catalyst 3650-24PS | Routage inter-VLAN, relais DHCP | 99 | 192.168.99.2 |
| SW-ACC-01 | Catalyst 2960 | Accès des postes | 99 | 192.168.99.3 |
| SW-SRV | Catalyst 2960 | Baie serveurs | 99 | 192.168.99.4 |
| SW-DMZ | Catalyst 2960 | Serveurs exposés | — | — |
| AP-WIFI | Access Point-PT | Wi-Fi des salariés | 40 | — |
| R-HOME | ISR 1941 | Site distant (télétravail) | — | 198.51.100.100 |

**Identifiants communs :** `enable secret V0ltaris2026` · compte local `adminsi` / `V0ltaris2026` · accès **SSH uniquement**.

---

## 5. Adressage des serveurs

| Hostname | VLAN | Adresse IP | Masque | Passerelle | DNS | Services à activer dans Packet Tracer |
|---|---|---|---|---|---|---|
| SRV-AD01 | 30 | 192.168.30.10 | 255.255.255.0 | 192.168.30.254 | 127.0.0.1 | DNS + DHCP (3 pools) |
| SRV-FILE01 | 30 | 192.168.30.20 | 255.255.255.0 | 192.168.30.254 | 192.168.30.10 | FTP |
| SRV-MAIL01 | 30 | 192.168.30.30 | 255.255.255.0 | 192.168.30.254 | 192.168.30.10 | EMAIL (domaine `voltaris.local`) |
| SRV-ADM01 | 30 | 192.168.30.40 | 255.255.255.0 | 192.168.30.254 | 192.168.30.10 | HTTP + SYSLOG + NTP |
| SRV-BKP01 | 30 | 192.168.30.50 | 255.255.255.0 | 192.168.30.254 | 192.168.30.10 | FTP |
| SRV-WEB01 | 50 | 192.168.50.10 | 255.255.255.0 | 192.168.50.254 | 198.51.100.53 | HTTP / HTTPS |
| SRV-APP01 | 50 | 192.168.50.20 | 255.255.255.0 | 192.168.50.254 | 198.51.100.53 | HTTP |
| SRV-SQL01 | 50 | 192.168.50.30 | 255.255.255.0 | 192.168.50.254 | 198.51.100.53 | *aucun — serveur simplement nommé* |
| SRV-API-EXT | Internet | 198.51.100.20 | 255.255.255.0 | 198.51.100.254 | 198.51.100.53 | HTTP |
| SRV-DNS-PUB | Internet | 198.51.100.53 | 255.255.255.0 | 198.51.100.254 | 127.0.0.1 | DNS (zone `voltaris.fr`) |

> Packet Tracer n'a pas de rôle « base de données ». SRV-SQL01 est un serveur générique, nommé et adressé, qui matérialise la séparation application / données exigée par le cahier des charges.

---

## 6. Adressage de chaque poste

### VLAN 10 — BUREAUX (7 postes)

| Poste | Service | Adresse IP | Mode | Switch | Port |
|---|---|---|---|---|---|
| PC-DIR-01 | Direction | 192.168.10.101 | DHCP | SW-ACC-01 | Fa0/1 |
| PC-DIR-02 | Direction | 192.168.10.102 | DHCP | SW-ACC-01 | Fa0/2 |
| PC-ADM-01 | Administratif et comptabilité | 192.168.10.103 | DHCP | SW-ACC-01 | Fa0/3 |
| PC-ADM-02 | Administratif et comptabilité | 192.168.10.104 | DHCP | SW-ACC-01 | Fa0/4 |
| PC-SAV-01 | Commercial et SAV | 192.168.10.105 | DHCP | SW-ACC-01 | Fa0/5 |
| PC-SAV-02 | Commercial et SAV | 192.168.10.106 | DHCP | SW-ACC-01 | Fa0/6 |
| PC-SAV-03 | Commercial et SAV | 192.168.10.107 | DHCP | SW-ACC-01 | Fa0/7 |

### VLAN 20 — ÉTUDES (7 postes + 2 objets connectés)

| Poste | Service | Adresse IP | Mode | Switch | Port |
|---|---|---|---|---|---|
| PC-ETU-01 | Bureau d'études R&D | 192.168.20.101 | DHCP | SW-ACC-01 | Fa0/9 |
| PC-ETU-02 | Bureau d'études R&D | 192.168.20.102 | DHCP | SW-ACC-01 | Fa0/10 |
| PC-ETU-03 | Bureau d'études R&D | 192.168.20.103 | DHCP | SW-ACC-01 | Fa0/11 |
| PC-ETU-04 | Bureau d'études R&D | 192.168.20.104 | DHCP | SW-ACC-01 | Fa0/12 |
| PC-ETU-05 | Bureau d'études R&D | 192.168.20.105 | DHCP | SW-ACC-01 | Fa0/13 |
| PC-ATE-01 | Atelier / assemblage | 192.168.20.106 | DHCP | SW-ACC-01 | Fa0/14 |
| PC-ATE-02 | Atelier / assemblage | 192.168.20.107 | DHCP | SW-ACC-01 | Fa0/15 |
| IOT-BANC-01 | Banc de test objets connectés | 192.168.20.120 | DHCP | SW-ACC-01 | Fa0/16 |
| IOT-BANC-02 | Banc de test objets connectés | 192.168.20.121 | DHCP | SW-ACC-01 | Fa0/17 |

### VLAN 40 — WI-FI

| Poste | Service | Adresse IP | Mode | Raccordement |
|---|---|---|---|---|
| LAP-DIR-01 | Portable direction | 192.168.40.101 | DHCP | AP-WIFI (sans fil) |
| LAP-SAV-01 | Portable commercial | 192.168.40.102 | DHCP | AP-WIFI (sans fil) |

### VLAN 99 — MANAGEMENT

| Poste | Service | Adresse IP | Mode | Switch | Port |
|---|---|---|---|---|---|
| PC-SI-01 | Responsable informatique | 192.168.99.100 | **Statique** | SW-ACC-01 | Fa0/24 |

### Postes hors du LAN

| Poste | Rôle | Adresse IP | Mode | Raccordement |
|---|---|---|---|---|
| PC-INTERNET | Client / visiteur externe | 198.51.100.10 | Statique | Segment public de R-FAI |
| PC-TELETRAVAIL | Salarié en télétravail | 192.168.200.10 | Statique | R-HOME |

> Les adresses des postes en DHCP sont celles que le serveur **doit** distribuer, dans l'ordre de démarrage. C'est ce qu'on vérifie au test T01.

---

## 7. Interfaces et SVI à configurer

| Équipement | Interface | Adresse IP | Masque | Options |
|---|---|---|---|---|
| R-FAI | Gi0/0 | 203.0.113.1 | 255.255.255.0 | — |
| R-FAI | Gi0/1 | 198.51.100.254 | 255.255.255.0 | `ip route 192.168.200.0 255.255.255.0 198.51.100.100` |
| R-EDGE | Gi0/0 | 203.0.113.2 | 255.255.255.0 | `ip nat outside` · `ip route 0.0.0.0 0.0.0.0 203.0.113.1` |
| R-EDGE | Gi0/1 | 10.0.0.1 | 255.255.255.252 | `ip nat inside` · `ip route 192.168.0.0 255.255.0.0 10.0.0.2` |
| R-EDGE | Gi0/2 | 192.168.50.254 | 255.255.255.0 | `ip nat inside` · `ip access-group ACL-DMZ in` |
| R-HOME | Gi0/0 | 198.51.100.100 | 255.255.255.0 | `ip route 0.0.0.0 0.0.0.0 198.51.100.254` |
| R-HOME | Gi0/1 | 192.168.200.254 | 255.255.255.0 | — |
| SW-CORE | Gi1/0/24 | 10.0.0.2 | 255.255.255.252 | `no switchport` · `ip route 0.0.0.0 0.0.0.0 10.0.0.1` |
| SW-CORE | Vlan10 | 192.168.10.254 | 255.255.255.0 | `ip helper-address 192.168.30.10` |
| SW-CORE | Vlan20 | 192.168.20.254 | 255.255.255.0 | `ip helper-address 192.168.30.10` |
| SW-CORE | Vlan30 | 192.168.30.254 | 255.255.255.0 | — |
| SW-CORE | Vlan40 | 192.168.40.254 | 255.255.255.0 | `ip helper-address 192.168.30.10` · `ip access-group ACL-WIFI in` |
| SW-CORE | Vlan99 | 192.168.99.254 | 255.255.255.0 | — |

> Sans la commande `ip routing` sur SW-CORE, aucun SVI ne route et rien ne communique entre VLAN.

---

## 8. Ports des switches

| Switch | Port(s) | Mode | VLAN | Raccordé à |
|---|---|---|---|---|
| SW-CORE | Gi1/0/1 | Trunk 802.1Q | 10,20,30,40,99 | SW-ACC-01 Gi0/1 |
| SW-CORE | Gi1/0/2 | Trunk 802.1Q | 10,20,30,40,99 | SW-SRV Gi0/1 |
| SW-CORE | Gi1/0/24 | Port routé | — | R-EDGE Gi0/1 |
| SW-ACC-01 | Gi0/1 | Trunk 802.1Q | 10,20,30,40,99 | SW-CORE Gi1/0/1 |
| SW-ACC-01 | Fa0/1 → Fa0/8 | Accès | 10 | Postes bureaux |
| SW-ACC-01 | Fa0/9 → Fa0/17 | Accès | 20 | Postes études, atelier, bancs IoT |
| SW-ACC-01 | Fa0/20 | Accès | 40 | AP-WIFI |
| SW-ACC-01 | Fa0/24 | Accès | 99 | PC-SI-01 |
| SW-SRV | Gi0/1 | Trunk 802.1Q | 10,20,30,40,99 | SW-CORE Gi1/0/2 |
| SW-SRV | Fa0/1 → Fa0/5 | Accès | 30 | Les 5 serveurs internes |
| SW-DMZ | Fa0/1 | Accès | 1 | R-EDGE Gi0/2 |
| SW-DMZ | Fa0/2 → Fa0/4 | Accès | 1 | SRV-WEB01, SRV-APP01, SRV-SQL01 |

Sur tous les ports d'accès utilisateurs : `port-security maximum 2`, `mac-address sticky`, `violation restrict`, `spanning-tree portfast`. Les ports inutilisés sont placés dans un VLAN mort et éteints.

---

## 9. Pools DHCP (à créer sur SRV-AD01)

| Nom du pool | Default Gateway | DNS Server | Start IP | Subnet Mask | Max Users |
|---|---|---|---|---|---|
| POOL-VLAN10 | 192.168.10.254 | 192.168.30.10 | 192.168.10.100 | 255.255.255.0 | 50 |
| POOL-VLAN20 | 192.168.20.254 | 192.168.30.10 | 192.168.20.100 | 255.255.255.0 | 50 |
| POOL-VLAN40 | 192.168.40.254 | 192.168.30.10 | 192.168.40.100 | 255.255.255.0 | 100 |

> Le serveur DHCP est dans le VLAN 30. Les postes des autres VLAN ne l'atteignent que si `ip helper-address 192.168.30.10` est configuré sur leur SVI. C'est l'oubli le plus fréquent sur ce type de maquette.

---

## 10. Enregistrements DNS

### Zone interne `voltaris.local` — sur SRV-AD01

| Nom | Type | Adresse |
|---|---|---|
| srv-ad01.voltaris.local | A | 192.168.30.10 |
| srv-file01.voltaris.local | A | 192.168.30.20 |
| srv-mail01.voltaris.local | A | 192.168.30.30 |
| srv-adm01.voltaris.local | A | 192.168.30.40 |
| glpi.voltaris.local | A | 192.168.30.40 |
| srv-bkp01.voltaris.local | A | 192.168.30.50 |

### Zone publique `voltaris.fr` — sur SRV-DNS-PUB

| Nom | Type | Adresse |
|---|---|---|
| www.voltaris.fr | A | 203.0.113.10 |
| sav.voltaris.fr | A | 203.0.113.20 |

> Deux zones distinctes : la zone interne n'est jamais publiée, la zone publique ne contient que les deux adresses NAT de la DMZ. Aucune adresse privée ne sort.

---

## 11. NAT et listes de contrôle d'accès

| Type | Règle | Appliqué sur | Effet |
|---|---|---|---|
| PAT | `ip nat inside source list NAT-LAN interface gi0/0 overload` | R-EDGE | Tous les postes sortent derrière 203.0.113.2 |
| ACL standard | `NAT-LAN : permit 192.168.0.0 0.0.255.255` | R-EDGE | Désigne les réseaux à traduire |
| NAT statique | `ip nat inside source static 192.168.50.10 203.0.113.10` | R-EDGE | Publie le site vitrine |
| NAT statique | `ip nat inside source static 192.168.50.20 203.0.113.20` | R-EDGE | Publie l'application SAV |
| ACL étendue | `ACL-DMZ` | R-EDGE Gi0/2, sens `in` | La DMZ envoie ses logs mais ne remonte pas dans le LAN |
| ACL étendue | `ACL-WIFI` | SW-CORE Vlan40, sens `in` | Le Wi-Fi n'atteint pas le VLAN Études |
| ACL standard | `ACL-MGMT : permit 192.168.99.0 0.0.0.255` | Lignes vty, `access-class in` | Seul le VLAN Management ouvre une session SSH |
| VPN IPsec | `crypto map CM`, pair 198.51.100.100 | R-EDGE Gi0/0 et R-HOME Gi0/0 | Tunnel chiffré vers le domicile |

```
ip access-list extended ACL-DMZ
 permit udp 192.168.50.0 0.0.0.255 host 192.168.30.40 eq 514
 deny   ip  192.168.50.0 0.0.0.255 192.168.0.0 0.0.255.255
 permit ip  any any

ip access-list extended ACL-WIFI
 deny   ip 192.168.40.0 0.0.0.255 192.168.20.0 0.0.0.255
 permit ip any any

ip access-list standard ACL-MGMT
 permit 192.168.99.0 0.0.0.255
```

---

## 12. Droits d'accès aux fichiers

Les droits par service sont portés par les **groupes Active Directory et les permissions NTFS**, pas par les VLAN. Les VLAN cloisonnent les flux réseau ; l'AD décide qui ouvre quel dossier.

| Partage | Groupe autorisé | Droit | Justification |
|---|---|---|---|
| `\\SRV-FILE01\Commun` | Tous les utilisateurs du domaine | Lecture / écriture | Échanges entre équipes |
| `\\SRV-FILE01\Etudes` | `G_Etudes` | Lecture / écriture | Plans R&D gardés en interne |
| `\\SRV-FILE01\Direction` | `G_Direction` | Lecture / écriture | Suivi de l'activité |
| `\\SRV-FILE01\Compta` | `G_Admin` | Lecture / écriture | Données financières, RGPD |
| `\\SRV-FILE01\SAV` | `G_Commercial` | Lecture / écriture | Suivi des retours clients |
| Tous les partages | `G_ServiceInfo` | Contrôle total | Responsable informatique |

---

*Les adresses publiques `203.0.113.0/24` et `198.51.100.0/24` sont des plages de documentation (RFC 5737), à remplacer par celles du fournisseur d'accès en production.*
