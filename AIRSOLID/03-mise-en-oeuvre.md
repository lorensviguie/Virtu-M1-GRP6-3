# Infrastructure Active Directory — AirSolid

## Informations générales

| Paramètre              | Valeur                             |
| ---------------------- | ---------------------------------- |
| **Domaine**            | `airsolid.local`                   |
| **NetBIOS**            | `AIRSOLID`                         |
| **Réseau**             | `10.1.248.0/24`                    |
| **Masque**             | `255.255.255.0`                    |
| **Passerelle**         | `10.1.248.1`                       |
| **Niveau fonctionnel** | Windows Server 2016 (WinThreshold) |

---

## Serveurs

| Serveur             | IP            | Rôle                                             |
| ------------------- | ------------- | ------------------------------------------------ |
| `DC1-AIRSOLID`      | `10.1.248.3`  | Contrôleur de domaine principal, DNS primaire    |
| `DC2-AIRSOLID`      | `10.1.248.5`  | Contrôleur de domaine secondaire, DNS secondaire |
| `SRV-TEST-AIRSOLID` | `10.1.248.11` | Serveur membre du domaine                        |
| `SRV-LNX-BDD-01`    | `10.1.248.9`  | Serveur de la base de donnée pour l'erp          |
| `SRV-LNX-ERP-01`    | `10.1.248.8`  | Serveur 1 pour l'erp                             |
| `SRV-LNX-ERP-02`    | `10.1.248.10` | Serveur 2 pour l'erp                             |
| `SRV-LNX-VPN-01`    | `10.1.248.12` | Serveur Peer pour la liaison cloudflare VPN      |

---

## DC1-AIRSOLID — Contrôleur de domaine principal

### Configuration réseau

```
IP        : 10.1.248.3
Masque    : 255.255.255.0
Passerelle: 10.1.248.1
DNS       : 10.1.248.3 (lui-même)
IPv6      : Désactivé
```

### Installation des rôles

```powershell
Install-WindowsFeature -Name AD-Domain-Services, DNS -IncludeManagementTools
```

### Création de la forêt

```powershell
$DSRMPassword = ConvertTo-SecureString "AirSolid@DSRM2024!" -AsPlainText -Force

Install-ADDSForest `
    -DomainName "airsolid.local" `
    -DomainNetbiosName "AIRSOLID" `
    -DomainMode "WinThreshold" `
    -ForestMode "WinThreshold" `
    -SafeModeAdministratorPassword $DSRMPassword `
    -InstallDns:$true `
    -DatabasePath "C:\Windows\NTDS" `
    -LogPath "C:\Windows\NTDS" `
    -SysvolPath "C:\Windows\SYSVOL" `
    -Force:$true
```

### Configuration DNS

```powershell
# Forwarders
Set-DnsServerForwarder -IPAddress "8.8.8.8","1.1.1.1"

# Zone de recherche inversée
Add-DnsServerPrimaryZone `
    -NetworkID "10.1.248.0/24" `
    -ReplicationScope "Forest" `
    -DynamicUpdate "Secure"

# Enregistrement PTR pour DC1
Add-DnsServerResourceRecordPtr `
    -ZoneName "248.1.10.in-addr.arpa" `
    -Name "3" `
    -PtrDomainName "DC1-AIRSOLID.airsolid.local"

# Scavenging
Set-DnsServerScavenging -ScavengingState $true -ScavengingInterval "7.00:00:00" -PassThru

Set-DnsServerZoneAging `
    -Name "airsolid.local" `
    -Aging $true `
    -NoRefreshInterval "7.00:00:00" `
    -RefreshInterval "7.00:00:00"
```

### Zones DNS créées

| Zone                    | Type                   | Rôle                    |
| ----------------------- | ---------------------- | ----------------------- |
| `airsolid.local`        | Principale intégrée AD | Zone directe du domaine |
| `_msdcs.airsolid.local` | Principale intégrée AD | Localisation des DC     |
| `248.1.10.in-addr.arpa` | Principale intégrée AD | Zone inverse            |
|                         |                        |                         |

### Rôles FSMO sur DC1

| Rôle                  | Serveur      |
| --------------------- | ------------ |
| Schema Master         | DC1-AIRSOLID |
| Domain Naming Master  | DC1-AIRSOLID |
| Infrastructure Master | DC1-AIRSOLID |

---

## DC2-AIRSOLID — Contrôleur de domaine secondaire

### Configuration réseau

```
IP        : 10.1.248.5
Masque    : 255.255.255.0
Passerelle: 10.1.248.1
DNS       : 10.1.248.3 (DC1) + 10.1.248.5 (lui-même)
IPv6      : Désactivé
```

### Installation des rôles

```powershell
Install-WindowsFeature -Name AD-Domain-Services, DNS -IncludeManagementTools
```

### Promotion en contrôleur de domaine additionnel

```powershell
$DSRMPassword = ConvertTo-SecureString "AirSolid@DSRM2024!" -AsPlainText -Force
$Cred = Get-Credential "AIRSOLID\Administrateur"

Install-ADDSDomainController `
    -DomainName "airsolid.local" `
    -InstallDns:$true `
    -Credential $Cred `
    -SafeModeAdministratorPassword $DSRMPassword `
    -DatabasePath "C:\Windows\NTDS" `
    -LogPath "C:\Windows\NTDS" `
    -SysvolPath "C:\Windows\SYSVOL" `
    -Force:$true
```

### Configuration DNS

```powershell
# Enregistrement PTR pour DC2
Add-DnsServerResourceRecordPtr `
    -ZoneName "248.1.10.in-addr.arpa" `
    -Name "5" `
    -PtrDomainName "DC2-AIRSOLID.airsolid.local"

# Forwarders
Set-DnsServerForwarder -IPAddress "8.8.8.8","1.1.1.1"
```

### Rôles FSMO sur DC2

| Rôle         | Serveur      |
| ------------ | ------------ |
| PDC Emulator | DC2-AIRSOLID |
| RID Master   | DC2-AIRSOLID |
|              |              |

### Transfert des rôles FSMO

```powershell
Move-ADDirectoryServerOperationMasterRole `
    -Identity "DC2-AIRSOLID" `
    -OperationMasterRole PDCEmulator, RIDMaster
```

---

## Structure Active Directory

### Unités d'organisation (OUs)

```
DC=airsolid,DC=local
└── OU=AirSolid
    ├── OU=Utilisateurs
    │   ├── OU=Direction
    │   ├── OU=Informatique
    │   ├── OU=Comptabilite
    │   ├── OU=Commercial
    │   ├── OU=RH
    │   └── OU=Production
    ├── OU=Ordinateurs
    ├── OU=Serveurs
    ├── OU=Groupes
    └── OU=Comptes de service
```

### Groupes de sécurité

| Groupe                | Type              | Description              |
| --------------------- | ----------------- | ------------------------ |
| `GRP_Direction`       | Global / Sécurité | Département Direction    |
| `GRP_Informatique`    | Global / Sécurité | Département Informatique |
| `GRP_Comptabilite`    | Global / Sécurité | Département Comptabilité |
| `GRP_Commercial`      | Global / Sécurité | Département Commercial   |
| `GRP_RH`              | Global / Sécurité | Département RH           |
| `GRP_Production`      | Global / Sécurité | Département Production   |
| `GRP_Admins_Systeme`  | Global / Sécurité | Administrateurs système  |
| `GRP_Admins_Helpdesk` | Global / Sécurité | Helpdesk                 |
| `GRP_VPN_Users`       | Global / Sécurité | Utilisateurs VPN         |
| `GRP_Impression`      | Global / Sécurité | Accès imprimantes        |

---

## Politiques de mots de passe

### Politique par défaut du domaine

| Paramètre             | Valeur           |
| --------------------- | ---------------- |
| Longueur minimale     | 12 caractères    |
| Complexité            | Activée          |
| Historique            | 10 mots de passe |
| Durée maximale        | 90 jours         |
| Durée minimale        | 1 jour           |
| Seuil de verrouillage | 5 tentatives     |
| Durée de verrouillage | 30 minutes       |
| Fenêtre d'observation | 15 minutes       |

### PSO Admins (Fine-Grained Password Policy)

Appliquée au groupe `GRP_Admins_Systeme`.

| Paramètre             | Valeur           |
| --------------------- | ---------------- |
| Longueur minimale     | 16 caractères    |
| Complexité            | Activée          |
| Historique            | 20 mots de passe |
| Durée maximale        | 60 jours         |
| Durée minimale        | 1 jour           |
| Seuil de verrouillage | 3 tentatives     |
| Durée de verrouillage | 1 heure          |
| Fenêtre d'observation | 30 minutes       |
| Priorité              | 10               |

---

## Compte de service LDAP

| Paramètre               | Valeur                                                                |
| ----------------------- | --------------------------------------------------------------------- |
| **Nom**                 | `ldap_elec`                                                           |
| **UPN**                 | `ldap_elec@airsolid.local`                                            |
| **DN**                  | `CN=ldap_elec,OU=Comptes de service,OU=AirSolid,DC=airsolid,DC=local` |
| **OU**                  | `OU=Comptes de service`                                               |
| **Mot de passe expire** | Non                                                                   |
| **Usage**               | Connexion LDAP pour Odoo ERP                                          |

### Configuration LDAP pour Odoo

| Champ          | Valeur                                                                |
| -------------- | --------------------------------------------------------------------- |
| Serveur LDAP   | `10.1.248.3`                                                          |
| Port           | `389`                                                                 |
| DN de liaison  | `CN=ldap_elec,OU=Comptes de service,OU=AirSolid,DC=airsolid,DC=local` |
| Base DN        | `OU=AirSolid,DC=airsolid,DC=local`                                    |
| Filtre LDAP    | `(sAMAccountName=%s)`                                                 |
| Attribut login | `sAMAccountName`                                                      |

---

## SRV-TEST-AIRSOLID — Serveur membre

### Configuration réseau

```
IP        : 10.1.248.11
Masque    : 255.255.255.0
Passerelle: 10.1.248.1
DNS       : 10.1.248.3 (DC1) + 10.1.248.5 (DC2)
IPv6      : Désactivé
```

### Jonction au domaine

```powershell
Add-Computer `
    -DomainName "airsolid.local" `
    -Credential (Get-Credential "AIRSOLID\Administrateur") `
    -Restart -Force
```

### Autorisation RDP pour les utilisateurs AD

```powershell
# Activer le RDP
Set-ItemProperty `
    -Path "HKLM:\System\CurrentControlSet\Control\Terminal Server" `
    -Name "fDenyTSConnections" `
    -Value 0

# Autoriser dans le pare-feu
Enable-NetFirewallRule -DisplayGroup "Bureau à distance"

# Ajouter un utilisateur au groupe RDP
Add-LocalGroupMember `
    -Group "Utilisateurs du Bureau à distance" `
    -Member "AIRSOLID\jdupont"
```

---

## Commandes de vérification

### Vérifier la réplication AD

```powershell
repadmin /showrepl
repadmin /replsummary
repadmin /syncall /AdeP
```

### Vérifier les rôles FSMO

```powershell
netdom query fsmo
```

### Vérifier les DC du domaine

```powershell
Get-ADDomainController -Filter * | Select-Object Name, IPv4Address, IsGlobalCatalog
```

### Diagnostics complets

```powershell
dcdiag /v
```

### Vérifier le DNS

```powershell
Resolve-DnsName "DC1-AIRSOLID.airsolid.local"
Resolve-DnsName "DC2-AIRSOLID.airsolid.local"
nslookup -type=SRV _ldap._tcp.airsolid.local
```
---



## Architecture finale

```
┌─────────────────────────┐         ┌─────────────────────────┐
│     DC1-AIRSOLID        │◄───────►│     DC2-AIRSOLID        │
│     10.1.248.3          │  Répl.  │     10.1.248.5          │
│                         │   HA    │                         │
│  Rôles FSMO :           │         │  Rôles FSMO :           │
│   - Schema Master       │         │   - PDC Emulator        │
│   - Domain Naming       │         │   - RID Master          │
│   - Infra Master        │         │                         │
│  DNS Primaire           │         │  DNS Secondaire         │
└─────────────────────────┘         └─────────────────────────┘
            │                                   │
            └──────────────┬────────────────────┘
                           │ airsolid.local
                           │ 10.1.248.0/24
            ┌──────────────┴────────────────────┐
            │       SRV-TEST-AIRSOLID            │
            │       10.1.248.11                  │
            │       Membre du domaine            │
            │       RDP : AIRSOLID\jdupont       │
            └───────────────────────────────────┘
```
---

## Infrastructure Cloudflare Zero Trust / VPN Gateway.  

Le serveur SRV-LNX-VPN-01 agit en tant que relais réseau exclusif. Il établit un tunnel chiffré sortant vers Cloudflare, permettant l'accès distant via le client WARP sans aucune ouverture de port sur le pare-feu périphérique (NAT/Box).  

Déploiement et Routage Réseau (Rocky Linux 9)  
```sh
# 1. Ajout du dépôt officiel Cloudflare et installation de cloudflared
curl -fsSL [https://pkg.cloudflare.com/cloudflared-ascii.repo](https://pkg.cloudflare.com/cloudflared-ascii.repo) | sudo tee /etc/yum.repos.d/cloudflared.repo
sudo dnf update -y && sudo dnf install cloudflared -y

# 2. Enregistrement du démon système (Remplacer par le jeton généré sur la console Zero Trust)
sudo cloudflared service install EYJuIjoiYmY2...[VOTRE_TOKEN]...

# 3. Activation du routage des paquets IP (IP Forwarding)
sudo sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.d/99-networking.conf

# 4. Configuration stricte de Firewalld pour autoriser le transit réseau
sudo firewall-cmd --zone=trusted --add-interface=tun+ --permanent
sudo firewall-cmd --zone=public --add-masquerade --permanent
sudo firewall-cmd --reload
```

## Infrastructure ERP Odoo & Cluster Base de Données.  

Afin de garantir une isolation applicative et de respecter les politiques de sécurité de l'infrastructure, tous les services Linux s'exécutent au sein du moteur de conteneurs rootless Podman via podman-compose.  

```
┌───────────────────────────────┐
                  │      Clients Distants (WARP)  │
                  └───────────────┬───────────────┘
                                  │ (Cloudflare Tunnel)
                                  ▼
                    ┌───────────────────────────┐
                    │      SRV-LNX-VPN-01       │
                    │       10.1.248.12         │
                    └─────────────┬─────────────┘
                                  │ (Routage LAN)
            ┌─────────────────────┴─────────────────────┐
            ▼                                           ▼
┌───────────────────────┐                   ┌───────────────────────┐
│    SRV-LNX-ERP-01     │                   │    SRV-LNX-ERP-02     │
│      10.1.248.8       │                   │      10.1.248.10      │
│  [Odoo App 1 (8069)]  │                   │ [Odoo App 2 (8080)]   │
└───────────┬───────────┘                   └───────────┬───────────┘
            │                                           │
            └─────────────────────┬─────────────────────┘
                                  ▼
                    ┌───────────────────────────┐
                    │      SRV-LNX-BDD-01       │
                    │        10.1.248.9         │
                    │    [PostgreSQL Master]    │
                    └─────────────┬─────────────┘
                                  │ (Sauvegardes chiffrées)
                                  ▼
                    ┌───────────────────────────┐
                    │     SRV-WIN-BACKUP-01     │
                    │       10.1.248.20         │
                    │   [Stockage de Secours]   │
                    └───────────────────────────┘
```

### 1. Stockage Persistant & Haute Disponibilité de la Base de Données

Le serveur SRV-LNX-BDD-01 héberge l'instance principale de production.  
Architecture Haute Disponibilité (PostgreSQL HA)  

Pour assurer la résilience des données applicatives ERP, la base de données utilise une réplication asynchrone par flux (Streaming Replication). Une instance réplique secondaire (Slave) peut être provisionnée instantanément sur une seconde VM réseau en clonant le volume PostgreSQL.  

Voici la configuration active sur l'instance principale :  

```sh
# [odoo@SRV-LNX-BDD-01 odoo-db]$ cat docker-compose.yml
version: '3.8'

services:
  db:
    image: postgres:15
    container_name: odoo_postgres_ha
    environment:
      - POSTGRES_DB=postgres
      - POSTGRES_USER=odoo
      - POSTGRES_PASSWORD=ChangementRequisSecretAD2026!
      - PGDATA=/var/lib/postgresql/data/pg_data
    volumes:
      - odoo-db-data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    command: >
      postgres 
      -c wal_level=replica 
      -c max_wal_senders=5 
      -c max_replication_slots=5 
      -c hot_standby=on
    restart: always

volumes:
  odoo-db-data:
```

### 2. Serveur Applicatif ERP - Instance 01 (SRV-LNX-ERP-01)  

```sh
# [odoo@SRV-LNX-ERP-01 odoo-app]$ cat docker-compose.yml
version: '3.8'

services:
  web:
    image: odoo:17.0
    container_name: odoo_app_ha
    ports:
      - "8069:8069"
    volumes:
      - odoo-web-data:/var/lib/odoo
    environment:
      - HOST=10.1.248.9
      - USER=odoo
      - PASSWORD=ChangementRequisSecretAD2026!
    restart: always

volumes:
  odoo-web-data:
```

### 3. Serveur Applicatif ERP - Instance 02 (SRV-LNX-ERP-02)

```sh
# [odoo@SRV-LNX-ERP-02 odoo-app2]$ cat docker-compose.yaml
version: '3.8'

services:
  web2:
    image: odoo:17.0
    container_name: odoo_app_ha_2
    ports:
      - "8080:8069"
    volumes:
      - odoo-web2-data:/var/lib/odoo
    environment:
      - HOST=10.1.248.9
      - USER=odoo
      - PASSWORD=ChangementRequisSecretAD2026!
    restart: always

volumes:
  odoo-web2-data:
```

## Stratégie de Sauvegarde Chiffrée Extérieure (Disaster Recovery). 

Les volumes de données gérés par Podman font l'objet d'un instantané (snapshot) quotidien à chaud. La base de données PostgreSQL est exportée de manière consistante via un script automatisé par tâche cron, puis transférée vers le serveur de stockage centralisé localisé sur SRV-WIN-BACKUP-01.  
Script de Restauration / Sauvegarde Chiffrée Automatisée  

Le script effectue une sauvegarde logique complète, la compresse, chiffre l'archive à l'aide d'une clé AES-256 symétrique, puis pousse le livrable vers le partage Windows sécurisé via le protocole SMB3.  

```sh
#!/bin/bash
# Script de sauvegarde odoo_backup.sh
BACKUP_DIR="/home/odoo/backups"
PASSPHRASE="CléUltraSecrèteDeChiffrementAirSolid2026"
SHARE_TARGET="//10.1.248.20/Backups/Odoo"
DATE=$(date +%Y%m%d_%H%M%S)

# 1. Extraction à chaud de la base PostgreSQL
podman exec odoo_postgres_ha pg_dumpall -U odoo > $BACKUP_DIR/dump_$DATE.sql

# 2. Compression et Chiffrement AES-256 symétrique via OpenSSL
openssl enc -aes-256-cbc -salt -in $BACKUP_DIR/dump_$DATE.sql -out $BACKUP_DIR/db_encrypted_$DATE.enc -k $PASSPHRASE

# 3. Copie sécurisée vers le serveur de stockage Windows 
smbclient $SHARE_TARGET -U "AIRSOLID\ldap_elec%MotDePasse" -c "cd Database; put $BACKUP_DIR/db_encrypted_$DATE.enc"

# 4. Nettoyage local
rm -f $BACKUP_DIR/dump_$DATE.sql $BACKUP_DIR/db_encrypted_$DATE.enc
```

---

*Document généré le 17/06/2026 — Infrastructure AirSolid*
