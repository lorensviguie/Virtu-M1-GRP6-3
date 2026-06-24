# Architecture.

voici l'architecture de l'infra actuel de airsolid.

![alt text](./assets/archi_actuel.png)

### Serveur physique (2012) — pièce maîtresse et point de défaillance unique
Un seul serveur physique héberge l'intégralité des services critiques : l'Active Directory (annuaire de l'entreprise), l'ERP web sous Docker, et les partages de fichiers Windows. Sa configuration exacte est inconnue du management. Une panne de 48h a déjà paralysé l'activité complète. C'est le risque numéro 1.

### Réseau local

Alimentation monophasée de 12 kVA — insuffisante pour héberger une infrastructure redondante ou une salle serveur sérieuse. La segmentation réseau (bureau / production / éventuellement entrepôt futur) n'est pas documentée.
Connectivité internet
Une seule fibre professionnelle à 1 Gbps. Pas de lien de secours : toute coupure opérateur = arrêt total pour les nomades et les services dépendant du cloud (M365 en déploiement).

### Accès distant
Un VPN est en place pour les 40 commerciaux nomades, mais sa qualité de service n'a jamais été évaluée formellement. Aucun accès préstataire externe formalisé.
Applications
ServiceHébergementÉtatActive DirectoryServeur 2012En production, critiqueERP (app web Docker)Serveur 2012En production, critiquePartages fichiersServeur 2012En productionMicrosoft 365Cloud MicrosoftDéploiement en coursSite WordPressInterne (serveur ?)En production, non sécuriséTéléphonie sans filLocalType non préciséVidéosurveillanceLocalAccès distant non formalisé 


## Archi on prem proposée :

![alt text](./assets/archi_onprem.png)

archi on prem :

2 serveur physiques : 

1 salle serveur déjà existantes 
1 salle serveur dans l'entrepot en cours d'acquisition. 

pour le budjet on a pris : 

serveur hp 10eme gen avec 128go de ram et 50to de stockage. 

4 000–10 000 € achat d'un serveur 
Maintenance / pannes / disques : ~300–1 000 €/an

soit, pour les 2 serveurs (site principal + entrepôt) : 8 000–20 000 € d'achat, 600–2 000 €/an de maintenance



On serait sur une proposition de passif/actif avec celui de l'entrepot 2 qui servirait de passif si l'actif tombe. 
temps de reprise en HA attendu a 30sec / 5 mins

## Répartitions et dimentionnement des vms :

### Hyperviseur

Les deux serveurs HP tournent sous **Proxmox VE** (licence open-source, pas de coût par socket), avec réplication de VMs entre les deux nœuds via le module Proxmox HA.

---

### Serveur 1 — Principal (salle serveur existante)

| VM                        | Rôle                                               | vCPU        | RAM       | Stockage   |
| ------------------------- | -------------------------------------------------- | ----------- | --------- | ---------- |
| VM-AD-01                  | Active Directory DC primaire (Windows Server 2022) | 4           | 8 Go      | 120 Go     |
| VM-AD-02                  | AD DC secondaire / réplica (Windows Server 2022)   | 4           | 8 Go      | 120 Go     |
| VM-ERP-01                 | Hôte Docker — ERP web applicatif + base de données | 8           | 32 Go     | 500 Go     |
| VM-FILE-01                | Partages fichiers Windows (SMB)                    | 4           | 16 Go     | 10 To      |
| VM-VPN-01                 | Serveur VPN (OpenVPN / WireGuard — 40 nomades)     | 2           | 4 Go      | 60 Go      |
| VM-WEB-01                 | WordPress + reverse proxy (Nginx)                  | 4           | 8 Go      | 200 Go     |
| VM-MON-01                 | Supervision et alertes (Zabbix / Grafana)          | 4           | 8 Go      | 200 Go     |
| **Total alloué**          |                                                    | **30 vCPU** | **84 Go** | **~11 To** |
| **Réserve hyperviseur**   | OS Proxmox + overcommit sécurisé                   | —           | 8 Go      | 100 Go     |
| **Disponible (headroom)** | Extensions futures                                 | —           | **36 Go** | **~39 To** |

> Les 39 To restants sont utilisés pour les **sauvegardes locales** (snapshots Proxmox Backup Server) et le stockage brut des données métier si les partages grandissent.

---

### Serveur 2 — Passif / Failover (entrepôt en acquisition)

Le nœud passif héberge les réplicas des VMs critiques via la réplication Proxmox (RPO ≈ 15 min, RTO ≈ 30 s – 5 min selon la VM).

| VM               | Rôle                                               | vCPU        | RAM       | Stockage   |
| ---------------- | -------------------------------------------------- | ----------- | --------- | ---------- |
| VM-AD-01-R       | Réplica AD DC (bascule auto en cas de panne S1)    | 4           | 8 Go      | 120 Go     |
| VM-ERP-01-R      | Réplica ERP Docker (bascule manuelle ou auto HA)   | 8           | 32 Go     | 500 Go     |
| VM-FILE-01-R     | Réplica partages fichiers (DFS-R ou Proxmox repl.) | 4           | 16 Go     | 10 To      |
| VM-VPN-01-R      | VPN de secours (actif/passif, même pool IP)        | 2           | 4 Go      | 60 Go      |
| **Total alloué** |                                                    | **18 vCPU** | **60 Go** | **~11 To** |
| **Disponible**   | Monitoring secondaire, tests, extensions           | —           | **60 Go** | **~39 To** |

> VM-WEB-01 et VM-MON-01 ne sont pas répliquées en priorité : le site WordPress et la supervision sont considérés **non critiques** pour la continuité d'activité immédiate. Elles peuvent être ajoutées si le budget le permet.

---

### Récapitulatif global

| Ressource            | Serveur 1 (alloué) | Serveur 2 (alloué) | Capacité totale (×2)                     |
| -------------------- | ------------------ | ------------------ | ---------------------------------------- |
| vCPU                 | 30                 | 18                 | 128 threads dispo (HP 10e gen ~64/serv.) |
| RAM                  | 84 Go / 128 Go     | 60 Go / 128 Go     | 256 Go total                             |
| Stockage VMs         | ~11 To / 50 To     | ~11 To / 50 To     | 100 To total                             |
| Stockage backup/data | ~39 To             | ~39 To             | ~78 To disponibles                       |

---

### Politique de sauvegarde

- **Proxmox Backup Server** installé sur chaque nœud : snapshots journaliers des VMs (rétention 7 jours)
- **Réplication inter-nœuds** toutes les 15 min pour AD, ERP et fichiers
- **Sauvegarde froide mensuelle** sur disque externe ou NAS dédié (règle 3-2-1)

---

### Coût

Serveur physique : 

~6 000–20 000 €/5 ans avec maintenance et coût de l'elec.


Cloud minimal HA/hybride :

VM standby : 50–300 €/mois
AD + VPN cloud : 50–150 €/mois
stockage 50 To backup : 500–1 200 €/mois

total = 600–1 650 €/mois soit 7 200–19 800 €/an (infra cloud seule : VM + AD/VPN + stockage — hors licences/MCO/connectivité déjà comptés dans le tableau comparatif ci-dessous)

( si panne totale = plus cher )


### Conclusion 

👉 Le serveur HP est :

✔ énormément moins cher à long terme
✔ parfait pour charge stable
❌ moins flexible
❌ maintenance à gérer

|                 | Avantages              | Inconvénients                | Coût estimé / an |
| --------------- | ---------------------- | ---------------------------- | ---------------- |
| Full on-premise | Maîtrise totale, CapEx | Risque panne matérielle, MCO | 20 000–39 000 €  |