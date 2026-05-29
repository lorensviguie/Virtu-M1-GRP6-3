# METALIS — Mise en œuvre

> **Environnement :** Salle serveur du site principal (VMs uniquement)  
> **Hyperviseur retenu :** Proxmox VE (type 1, open source, adapté à un budget limité)  
> **Stratégie :** Cloud hybride — on-premise site principal + OVHcloud pour les services exposés

---

## 1. Principes directeurs

METALIS est une PME sans administrateur interne, avec un serveur physique vieillissant (~20 ans), un NAS saturé, deux lignes ADSL et des postes de production sous Windows 7 hors réseau. La priorité absolue est la **continuité de production** et la **protection des données sensibles** (brevets, procédés de fabrication).

L'architecture retenue repose sur trois piliers :

1. **Virtualisation on-premise** (site principal) — héberger les services critiques internes sur Proxmox
2. **Cloud OVHcloud** — externaliser les services exposés (site, e-commerce, sauvegardes distantes)
3. **Sécurisation du périmètre** — VPN, IAM, segmentation réseau, remplacement de l'ADSL

---

## 2. Infrastructure virtuelle (site principal — Proxmox VE)

### 2.1 Serveur hôte Proxmox

Le serveur physique de 20 ans est remplacé par un nouveau serveur dédié installé en salle serveur du site principal (alimentation triphasée disponible). Proxmox VE est installé en bare-metal.

**VMs provisionnées :**

| VM | Rôle | OS | vCPU | RAM | Stockage |
|----|------|----|------|-----|----------|
| `vm-dc01` | Active Directory / DNS / DHCP | Windows Server 2022 | 2 | 4 Go | 60 Go |
| `vm-odoo` | ERP Odoo (production) | Ubuntu 22.04 | 4 | 8 Go | 100 Go |
| `vm-nas` | Serveur de fichiers CAO (migration NAS) | TrueNAS Scale | 2 | 8 Go | 2 To (datastore dédié) |
| `vm-backup` | Proxmox Backup Server (PBS) | Debian 12 | 2 | 4 Go | 1 To |
| `vm-vpn` | Passerelle VPN (WireGuard) | Debian 12 | 1 | 2 Go | 20 Go |
| `vm-supervision` | Supervision (Zabbix ou Grafana/Prometheus) | Ubuntu 22.04 | 2 | 4 Go | 50 Go |

### 2.2 Réseau virtuel (bridges Proxmox)

Segmentation en VLANs pour isoler les flux :

| VLAN | Nom | Usage |
|------|-----|-------|
| VLAN 10 | PROD | Postes atelier, tablettes, CNC (isolé) |
| VLAN 20 | BUREAU | Postes bureau, AD, Odoo |
| VLAN 30 | MGMT | Administration Proxmox, PBS, supervision |
| VLAN 40 | VPN | Tunnel commerciaux en télétravail |
| VLAN 50 | CAM | Caméras IP atelier (isolé) |

> Les postes Windows 7 CNC restent **hors réseau général** mais accessibles depuis le VLAN VPN via jump host sécurisé, permettant l'accès télétravail des commerciaux.

---

## 3. Services cloud OVHcloud

Le client utilise déjà OVH pour son VPS WordPress — on étend cette relation de confiance.

| Service | Solution OVH | Justification |
|---------|-------------|---------------|
| Site WordPress + WooCommerce | VPS OVH renforcé (ou Web Cloud) | Déjà en place, pic de trafic promo absorbé |
| Sauvegardes distantes (offsite) | OVH Object Storage (S3) | Règle 3-2-1 : copie hors site chiffrée |
| Failover DNS | OVH DNS secondaire | Résilience si ligne principale coupée |

---

## 4. Connectivité & redondance réseau

La double ligne ADSL est remplacée par :

- **Lien principal :** Fibre professionnelle (SLA garanti, débit symétrique ≥ 500 Mb)
- **Lien de secours :** Routeur 4G/5G en failover automatique

Le routeur/firewall (pfSense ou OPNsense sur VM dédiée, ou boîtier physique) gère :
- Le load balancing / failover entre les deux liens
- Les règles NAT et pare-feu par VLAN
- Le tunnel VPN WireGuard vers les commerciaux

---

## 5. Accès distant — Commerciaux en télétravail

**Solution retenue : WireGuard VPN** (léger, performant, open source)

- Chaque commercial dispose d'un profil WireGuard (clé publique/privée)
- Accès restreint au VLAN 40 → rebond vers VLAN 10 uniquement via jump host
- Les postes CNC Windows 7 sont accessibles en **RDP** depuis le jump host (pas directement exposés)
- Accès fournisseur CNC : compte dédié à durée limitée, tracé dans les logs

---

## 6. Sauvegarde & PRA

### Politique de sauvegarde — règle 3-2-1

| Copie | Où | Outil | Fréquence |
|-------|----|-------|-----------|
| 1 — Locale VM | `vm-backup` (PBS) | Proxmox Backup Server | Nightly |
| 2 — NAS dédié site principal | Datastore secondaire Proxmox | Snapshot + rsync | Hebdo |
| 3 — Offsite OVH | OVH Object Storage S3 | Rclone (chiffré AES-256) | Hebdo |

### PRA NAS (prioritaire)

Le NAS actuel est le point critique identifié (panne l'an dernier). Migration des fichiers CAO vers `vm-nas` (TrueNAS Scale) avec :
- ZFS RAIDZ1 pour la tolérance aux pannes disque
- Snapshots ZFS quotidiens (rétention 30 jours)
- Réplication vers PBS pour la sauvegarde complète

### Tests de restauration

- Test mensuel de restauration d'une VM depuis PBS
- Test trimestriel de restauration complète depuis OVH S3
- Documentation du RTO/RPO cible :
  - **RTO :** < 4h (reprise en moins d'une demi-journée de travail)
  - **RPO :** < 24h (perte de données maximale acceptable)

---

## 7. Sécurité & IAM

### Gestion des accès

- **Active Directory** (`vm-dc01`) centralise l'authentification de tous les postes bureau
- GPO par profil : commerciaux, atelier, direction, prestataires
- Comptes prestataires (CNC, MSP) : accès temporaires à durée limitée, révocables

### Données sensibles (brevets & procédés)

- Partage dédié sur `vm-nas` avec droits restreints (direction + bureau d'études uniquement)
- Chiffrement au repos (BitLocker sur VMs Windows, ZFS encryption sur TrueNAS)
- Journalisation des accès fichiers activée (audit AD)

### Postes Windows 7

- Isolés sur VLAN 10 (pas d'accès internet direct)
- Migration progressive recommandée vers Windows 10/11 (hors périmètre immédiat du projet)
- Audit de sécurité à planifier avec le prestataire MSP

---

## 8. Supervision

**Solution : Zabbix** (ou Grafana + Prometheus selon préférence)

Éléments supervisés :
- État des VMs Proxmox (CPU, RAM, disque, réseau)
- Services critiques : AD, Odoo, NAS, VPN, PBS
- Liens réseau (fibre + 4G failover)
- Alertes par email / SMS en cas d'incident

---

## 9. Contractualisation & MSP

METALIS n'a pas d'IT interne — la solution ne sera pas auto-administrée. Il est recommandé de :

- Contractualiser avec un **prestataire MSP** (Managed Service Provider) avec SLA défini
- Inclure dans le contrat : supervision 24/7, intervention sous 4h, mises à jour de sécurité mensuelles
- Maintenir le prestataire actuel pour les CNC (accès encadré via compte dédié)

---

## 10. Récapitulatif des choix techniques

| Besoin | Solution retenue | Justification |
|--------|-----------------|---------------|
| Hyperviseur | Proxmox VE | Open source, type 1, adapté budget PME |
| AD / DNS | Windows Server 2022 (VM) | Remplacement serveur 20 ans |
| Stockage CAO | TrueNAS Scale (VM) + ZFS | Fiabilité, snapshots, migration NAS |
| Sauvegarde | Proxmox Backup Server + OVH S3 | Règle 3-2-1, offsite chiffré |
| VPN | WireGuard | Léger, sécurisé, open source |
| Connectivité | Fibre pro + 4G failover | SLA garanti, redondance |
| Cloud | OVHcloud (déjà utilisé) | Continuité, souveraineté française |
| Supervision | Zabbix | Open source, alertes temps réel |
| Firewall | pfSense / OPNsense | Segmentation VLAN, VPN, failover |