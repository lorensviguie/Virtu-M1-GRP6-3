# AIRSOLID — Objectifs pédagogiques

> Ces 8 objectifs couvrent les compétences attendues par le module Virtualisation, appliquées au cas AIRSOLID.

---

## Objectif 1 — Proxmox VE : hyperviseur de type 1

**Contexte AIRSOLID :** Le serveur physique de 2012 est en fin de vie et constitue un SPOF. Il est remplacé par un cluster Proxmox VE à deux nœuds (site principal + entrepôt).

**Ce que nous devons expliquer :**
- Proxmox VE est un hyperviseur de **type 1** (bare-metal) basé sur KVM + LXC, installé directement sur le matériel sans OS hôte intermédiaire.
- Il permet de mutualiser les ressources physiques en plusieurs VMs isolées : AD, ERP Docker, fichiers, PBS, VPN, supervision, KMS.
- La fonctionnalité **Proxmox HA Cluster** permet de faire migrer ou redémarrer automatiquement une VM sur un nœud sain en cas de panne d'un nœud — directement applicable à la contrainte *"Plus jamais 48h sans ERP"*.
- **Pourquoi Proxmox :** open source (pas de coût de licence hyperviseur), cluster HA natif, intégration PBS, interface web complète, adapté à une infrastructure sans IT interne.

---

## Objectif 2 — Ressources & sécurité : dimensionnement, isolation, accès

**Contexte AIRSOLID :** 80 utilisateurs répartis en 4 profils (nomades, admin, bureau d'études, production), données commerciales potentiellement sensibles, accès prestataires externes à sécuriser.

**Ce que nous devons expliquer :**
- **Dimensionnement :** `vm-erp` reçoit 4 vCPU / 8 Go RAM pour absorber la charge des 80 utilisateurs. `vm-file` dispose de 500 Go pour les partages. Les ressources sont allouées dynamiquement — Proxmox permet d'ajuster sans redémarrage de l'hôte.
- **Isolation :** segmentation en VLANs distincts — BUREAU, SERVEURS, VPN/ZTNA, CAM, PRESTATAIRES. Les commerciaux nomades n'ont pas accès aux serveurs internes directement, seulement aux applications autorisées.
- **Accès :** AD centralise l'authentification. Profils par rôle via GPO. Comptes prestataires sur VLAN dédié (VLAN 60), avec MFA, durée limitée et journalisation complète.
- **Sécurité données :** chiffrement des sauvegardes (AES-256), registre des traitements RGPD, DPA avec les sous-traitants cloud.

---

## Objectif 3 — Architecture hybride on-premise / cloud

**Contexte AIRSOLID :** Client ouvert cloud et on-premise, souveraineté européenne souhaitée (non bloquante). Pas de budget en déficit — des graphiques comparatifs sont attendus.

**Ce que nous devons expliquer :**
- **On-premise (site principal + entrepôt) :** héberge les services critiques — AD, ERP Docker, partages fichiers, PBS. Ces services fonctionnent sans dépendre d'une connexion internet.
- **Cloud (OVHcloud ou équivalent EU) :** héberge les sauvegardes offsite (Object Storage S3), le quorum Proxmox (witness VPS), et potentiellement le site WordPress externalisé.
- **Hybride :** les deux couches sont complémentaires. L'ERP Docker est conçu pour être porté sur cloud (Kubernetes) si la décision est prise — l'architecture ne verrouille pas le client.
- **Comparatif présenté au client :**

| Scénario | Avantages | Inconvénients | Coût estimé / an |
|----------|-----------|---------------|------------------|
| Full on-premise | Maîtrise totale, CapEx | Risque panne matérielle, MCO | 20 000–39 000 € |
| Cloud hybride | Résilience + maîtrise données | Coût récurrent OpEx | 19 000–38 000 € |
| Full cloud | Scalabilité, 0 matériel | Dépendance internet, OpEx élevé | 17 000–30 000 € |

- **Recommandation :** cloud hybride — services critiques on-premise (site principal + entrepôt), sauvegarde offsite et services nomades sur cloud EU.

---

## Objectif 4 — Supervision : suivi des VMs et services critiques

**Contexte AIRSOLID :** Pannes logicielles et coupures internet fréquentes. Aucune IT interne — la supervision doit détecter les incidents avant qu'ils bloquent 80 utilisateurs.

**Ce que nous devons expliquer :**
- **Outil retenu : Zabbix** (open source, déployé sur `vm-supervision`).
- Éléments surveillés : état du cluster Proxmox (2 nœuds), VMs critiques (AD, ERP, PBS, VPN), liens réseau (fibre + 4G failover), lien inter-sites site principal ↔ entrepôt, services applicatifs (port ERP, AD LDAP, etc.).
- **Alertes :** notification email / SMS au prestataire externe dès qu'un seuil est franchi (service down, CPU > 90 %, disque > 80 %, lien réseau coupé).
- **Historique :** Zabbix conserve les métriques sur 6 mois minimum — permet d'analyser les causes des pannes passées (le LAN défaillant identifié peut être corrélé aux métriques réseau).
- Sans supervision active, la panne de 48h récente aurait pu être détectée en quelques minutes plutôt que découverte par les utilisateurs.

---

## Objectif 5 — Sauvegardes & PRA : stratégie et tests de restauration

**Contexte AIRSOLID :** Aucune sauvegarde existante. Obligation RGPD non respectée. Panne de 48h récente. Priorité absolue.

**Ce que nous devons expliquer :**
- **Règle 3-2-1 appliquée :**
  - Copie 1 : sauvegarde nightly des VMs via Proxmox Backup Server (site principal)
  - Copie 2 : réplication horaire Proxmox vers le nœud entrepôt (inter-sites)
  - Copie 3 : export hebdomadaire chiffré (AES-256) vers OVH Object Storage (offsite)
- **Conformité RGPD :** registre des traitements, chiffrement obligatoire, DPA avec les sous-traitants, durées de rétention définies par type de donnée.
- **RTO cible ERP : < 2h** — en cas de panne du nœud principal, Proxmox HA redémarre `vm-erp` sur le nœud entrepôt automatiquement (RTO quasi nul si HA actif).
- **RPO cible : < 1h** — grâce à la réplication inter-sites toutes les heures.
- **Tests de restauration :** mensuel sur `vm-erp` depuis PBS, trimestriel depuis OVH S3. Documentés et signés par le prestataire externe.
- **Comparaison avec la situation actuelle :** zéro sauvegarde = RPO infini (toutes les données depuis la dernière copie USB sont perdues en cas de panne).

---

## Objectif 6 — VDI & profils : accès distant et postes centralisés

**Contexte AIRSOLID :** 40 commerciaux nomades sans poste fixe au bureau. VPN existant dont la satisfaction est inconnue.

**Ce que nous devons expliquer :**
- **Pas de VDI complet retenu** : le coût et la complexité d'un VDI (Citrix, VMware Horizon…) ne sont pas justifiés pour des commerciaux qui utilisent principalement l'ERP et Microsoft 365 — deux services accessibles via navigateur.
- **Solution retenue :** accès applicatif via VPN ou ZTNA selon résultats de l'enquête de satisfaction.
  - VPN WireGuard : accès réseau complet au VLAN autorisé, client léger, performant
  - ZTNA : accès par application uniquement (ERP, partages), sans accès réseau global — plus sécurisé, plus adapté à des nomades sur réseaux non maîtrisés (Wi-Fi public, 4G)
- **Profils AD itinérants :** les commerciaux disposent d'un profil AD centralisé, leurs données sont stockées sur `vm-file` — aucune donnée critique sur les postes locaux.
- **Pertinence VDI :** documenté comme évolution si AIRSOLID recrute massivement ou ouvre d'autres sites où des postes légers (thin clients) seraient déployés.

---

## Objectif 7 — Hyper-V & résilience : lien avec l'atelier résilience Windows

**Contexte AIRSOLID :** Architecture Proxmox retenue, mais l'atelier « résilience Windows » porte sur Hyper-V — il faut établir le lien entre les deux.

**Ce que nous devons expliquer :**
- **Hyper-V** est l'hyperviseur type 1 Microsoft intégré à Windows Server. Il aurait pu être retenu pour AIRSOLID (environnement AD + M365), mais Proxmox a été préféré pour la flexibilité et l'absence de licence supplémentaire.
- **Résilience Windows appliquée à AIRSOLID :**
  - Le **Windows Server Failover Cluster (WSFC)** d'Hyper-V permet la migration automatique de VMs entre hôtes physiques — c'est le pendant Microsoft de Proxmox HA Cluster.
  - Le **contrôleur de domaine secondaire** (à ajouter sur le nœud entrepôt) est une bonne pratique issue de l'atelier résilience : un seul DC = SPOF sur l'annuaire.
  - La notion de **quorum** (nœud témoin pour éviter le split-brain dans un cluster à 2 nœuds) est identique entre Hyper-V et Proxmox — le VPS witness cloud joue ce rôle ici.
- **Lien atelier :** les mécanismes de bascule automatique étudiés en atelier Hyper-V (live migration, VM failover) se retrouvent dans Proxmox HA — même objectif de résilience, même concept de cluster, deux implémentations différentes.

---

## Objectif 8 — PRA / PCA : plan de continuité documenté

**Contexte AIRSOLID :** Panne de 48h récente. Promesse client : *"Plus jamais 48h sans ERP"*. Aucun PRA formalisé.

**Ce que nous devons expliquer :**
- **PRA (Plan de Reprise d'Activité) :** procédure de redémarrage des services après incident. Déclenché quand le service est interrompu.
- **PCA (Plan de Continuité d'Activité) :** maintien d'une activité minimale *pendant* l'incident — plus ambitieux.
- **Pour AIRSOLID, on cible un PCA via l'architecture HA** : le cluster Proxmox assure la continuité sans intervention humaine en cas de panne d'un nœud. Le PRA couvre les scénarios de panne totale.

**Scénarios documentés :**

| Scénario | Impact | Action | RTO |
|----------|--------|--------|-----|
| Coupure internet fibre | Perte accès nomades + cloud | Bascule automatique 4G failover | < 5 min |
| Panne nœud Proxmox site principal | VMs down sur site principal | Proxmox HA redémarre VMs sur nœud entrepôt | < 15 min |
| Panne ERP (container Docker) | ERP inaccessible | Redémarrage container ou restauration PBS | < 1h |
| Panne lien inter-sites | Réplication suspendue | Alerte Zabbix, intervention prestataire | / |
| Ransomware | Données chiffrées | Restauration depuis snapshot PBS (avant infection) | < 4h |
| Panne totale (site principal + entrepôt) | Toute infra down | Restauration depuis OVH S3 sur nouveau matériel | < 8h |

- **Le PRA est testé trimestriellement** — les tests sont documentés et comparés aux RTO cibles.
- **Déclenchement :** le prestataire externe reçoit une alerte Zabbix, consulte le runbook (procédure documentée step-by-step), intervient selon le scénario identifié.
- **Point clé :** avec l'ancienne architecture (serveur unique 2012), tous ces scénarios aboutissaient à la même issue — 48h d'arrêt. Avec le cluster HA, seul le scénario de panne totale nécessite une restauration longue.