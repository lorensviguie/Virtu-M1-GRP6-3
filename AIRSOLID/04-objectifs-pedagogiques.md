# AIRSOLID — Objectifs pédagogiques

> Ces 8 objectifs couvrent les compétences attendues par le module Virtualisation, appliquées au cas AIRSOLID.

---

## Objectif 1 — Proxmox VE : hyperviseur de type 1

**Contexte AIRSOLID :** Le serveur physique de 2012 est en fin de vie et constitue un SPOF. Il est remplacé par Proxmox VE sur deux nœuds (site principal + entrepôt), avec une continuité de service assurée par **réplication** plutôt que par un cluster HA à quorum.

**Ce que nous devons expliquer :**
- Proxmox VE est un hyperviseur de **type 1** (bare-metal) basé sur KVM + LXC, installé directement sur le matériel.
- Il permet de mutualiser les ressources physiques en plusieurs VMs isolées : AD, ERP Docker, fichiers, PBS, VPN, supervision, KMS.
- **Continuité de service retenue : réplication, pas de cluster HA à quorum.** Plutôt qu'un cluster Proxmox HA "classique" (qui nécessite un quorum — donc au moins 3 nœuds votants, ou un témoin externe, pour basculer en toute sécurité), AIRSOLID retient la **réplication de stockage** (`pvesr`, basée sur ZFS) entre le site principal et l'entrepôt : les VMs critiques (`vm-erp`, `vm-file`) sont répliquées en continu. Une bascule **scriptée, déclenchée et validée via une alerte Zabbix**, démarre la copie répliquée sur l'entrepôt en cas de panne confirmée.
- **Pourquoi ce choix plutôt qu'un HA "100 % automatique" :** un cluster à seulement 2 nœuds ne peut pas garantir une bascule automatique sûre sans un 3ᵉ témoin de quorum — sans quoi une simple coupure du lien inter-sites pourrait être interprétée comme une panne et provoquer un démarrage simultané de la même VM aux deux endroits (split-brain). La réplication + bascule scriptée évite ce risque et simplifie l'exploitation — un point important pour un client **sans IT interne**, dont la supervision et les interventions sont déléguées à un prestataire externe.
- **Pourquoi Proxmox :** open source (pas de coût de licence hyperviseur), réplication de stockage native, intégration PBS, interface web complète, adapté à une infrastructure sans IT interne.

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
- **Cloud (OVHcloud ou équivalent EU) :** héberge les sauvegardes offsite (Object Storage S3) et potentiellement le site WordPress externalisé. Dans le scénario hybride (non retenu), il pourrait aussi héberger une copie répliquée des VMs critiques en cas d'absence de second site physique.
- **Mécanisme de continuité par réplication (cf. Objectif 1) :** que la bascule se fasse vers l'entrepôt (scénario retenu) ou vers le cloud (scénario hybride), le principe reste le même — réplication régulière + bascule scriptée validée par une alerte Zabbix, sans cluster à quorum automatique. Le nœud principal continue de fonctionner en standalone : sa disponibilité locale ne dépend jamais d'une ressource distante joignable.
- **Hybride :** les deux couches restent complémentaires même en full on-premise. L'ERP Docker est conçu pour être porté sur cloud (Kubernetes) si la décision est prise plus tard — l'architecture ne verrouille pas le client.
- **Comparatif présenté au client :**

| Scénario        | Avantages                     | Inconvénients                | Coût estimé / an |
| --------------- | ----------------------------- | ---------------------------- | ---------------- |
| Full on-premise | Maîtrise totale, CapEx        | Risque panne matérielle, MCO | 20 000–39 000 €  |
| Cloud hybride   | Résilience + maîtrise données | Coût récurrent OpEx          | 19 000–38 000 €  |
|                 |                               |                              |                  |

- **Recommandation : Full on-premise** — services critiques sur site principal + entrepôt, continuité assurée par réplication (pas de cluster à quorum, pas de dépendance cloud pour la disponibilité), sauvegarde offsite sur cloud EU. Ce choix évite le coût récurrent OpEx du cloud hybride tout en restant simple à exploiter pour un prestataire externe sans IT interne sur site.

---

## Objectif 4 — Supervision : suivi des VMs et services critiques

**Contexte AIRSOLID :** Pannes logicielles et coupures internet fréquentes. Aucune IT interne — la supervision doit détecter les incidents avant qu'ils bloquent 80 utilisateurs.

**Ce que nous devons expliquer :**
- **Outil retenu : Zabbix** (open source, déployé sur `vm-supervision`).
- Éléments surveillés : disponibilité des nœuds Proxmox (site principal + entrepôt), VMs critiques (AD, ERP, PBS, VPN), liens réseau (fibre + 4G failover), lien inter-sites, état de la réplication (`pvesr`), services applicatifs (port ERP, AD LDAP, etc.).
- **Zabbix est l'élément central de détection de panne.** Comme la continuité de service repose sur la réplication et non sur un cluster à quorum automatique (cf. Objectif 1), c'est l'alerte Zabbix (service injoignable depuis X minutes) qui déclenche la procédure de bascule scriptée vers l'entrepôt — il n'y a pas de bascule automatique basée sur un vote de cluster.
- **Alertes :** notification email / SMS au prestataire externe dès qu'un seuil est franchi (service down, CPU > 90 %, disque > 80 %, lien réseau coupé, retard de réplication).
- **Historique :** Zabbix conserve les métriques sur 6 mois minimum — permet d'analyser les causes des pannes passées (le LAN défaillant identifié peut être corrélé aux métriques réseau).
- Sans supervision active, la panne de 48h récente aurait pu être détectée en quelques minutes plutôt que découverte par les utilisateurs.

---

## Objectif 5 — Sauvegardes & PRA : stratégie et tests de restauration

**Contexte AIRSOLID :** Aucune sauvegarde existante. Obligation RGPD non respectée. Panne de 48h récente. Priorité absolue.

**Ce que nous devons expliquer :**
- **Deux mécanismes distincts, à ne pas confondre :**
  - **Réplication** (`pvesr`, ZFS) : copie quasi continue des VMs critiques du site principal vers l'entrepôt, toutes les 15 minutes. C'est ce mécanisme qui pilote le RTO/RPO de reprise rapide.
  - **Sauvegarde** (règle 3-2-1, conformité RGPD) : copie distincte servant à la rétention longue durée et à la restauration en cas de corruption/ransomware — pas au basculement rapide.
- **Règle 3-2-1 appliquée :**
  - Copie 1 : sauvegarde nightly des VMs via Proxmox Backup Server (site principal)
  - Copie 2 : export hebdomadaire chiffré (AES-256) vers OVH Object Storage (offsite)
- **Conformité RGPD :** registre des traitements, chiffrement obligatoire, DPA avec les sous-traitants, durées de rétention définies par type de donnée.
- **RTO cible ERP : 5 à 15 min** — détection de la panne par Zabbix, puis bascule scriptée (validée par le prestataire) qui démarre `vm-erp` depuis sa dernière réplique sur l'entrepôt. Ce n'est pas un basculement instantané : une étape de confirmation est nécessaire pour éviter de réagir à une simple coupure réseau passagère.
- **RPO cible : 15 min** — aligné sur l'intervalle de réplication `pvesr`. Le backup PBS nightly reste utile pour la conformité RGPD et la récupération longue durée, mais n'est pas ce qui pilote ce RPO.
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
- **Même contrainte de quorum côté Microsoft :** un cluster **Windows Server Failover Cluster (WSFC)** à 2 nœuds a exactement le même besoin qu'un cluster Proxmox — un témoin de quorum (witness de partage de fichiers ou cloud) pour basculer automatiquement sans risque de split-brain. Ce n'est pas une limite propre à Proxmox mais une contrainte générale des clusters à 2 nœuds — qu'AIRSOLID évite en choisissant la réplication + bascule scriptée plutôt qu'un cluster à quorum (cf. Objectif 1).
- **Résilience Windows appliquée à AIRSOLID :**
  - Le **contrôleur de domaine secondaire** (à ajouter sur le nœud entrepôt) est une bonne pratique issue de l'atelier résilience : un seul DC = SPOF sur l'annuaire.
- **Lien atelier :** les mécanismes de bascule étudiés en atelier Hyper-V (live migration, VM failover, témoin de quorum) se retrouvent dans la réflexion qui a guidé le choix d'architecture Proxmox — même problématique de résilience et de quorum à 2 nœuds, et le même choix pragmatique côté AIRSOLID : simplifier via la réplication plutôt que complexifier via un witness.

---

## Objectif 8 — PRA / PCA : plan de continuité documenté

**Contexte AIRSOLID :** Panne de 48h récente. Promesse client : *"Plus jamais 48h sans ERP"*. Aucun PRA formalisé.

**Ce que nous devons expliquer :**
- **PRA (Plan de Reprise d'Activité) :** procédure de redémarrage des services après incident. Déclenché quand le service est interrompu.
- **PCA (Plan de Continuité d'Activité) :** maintien d'une activité minimale *pendant* l'incident — plus ambitieux, suppose une bascule automatique sans intervention humaine.
- **Pour AIRSOLID, on cible un PRA réactif et rapide, pas un PCA pur :** la réplication entre le site principal et l'entrepôt permet une reprise en quelques minutes après détection (alerte Zabbix) et action (script ou prestataire) — sans prétendre à une continuité totalement transparente. C'est un choix délibéré (cf. Objectif 1) : un PCA automatique à 2 nœuds exigerait un témoin de quorum, alors qu'un PRA rapide via réplication suffit largement à tenir la promesse *"plus jamais 48h sans ERP"* (quelques minutes contre 48h).

**Scénarios documentés :**

| Scénario                                 | Impact                      | Action                                                                        | RTO      |
| ---------------------------------------- | --------------------------- | ----------------------------------------------------------------------------- | -------- |
| Coupure internet fibre                   | Perte accès nomades + cloud | Bascule automatique 4G failover                                               | < 5 min  |
| Panne nœud Proxmox site principal        | VMs down sur site principal | Détection Zabbix + bascule scriptée des VMs depuis la réplique sur l'entrepôt | 5–15 min |
| Panne ERP (container Docker)             | ERP inaccessible            | Redémarrage container ou restauration PBS                                     | < 1h     |
| Panne lien inter-sites                   | Réplication suspendue       | Alerte Zabbix, intervention prestataire                                       | /        |
| Ransomware                               | Données chiffrées           | Restauration depuis snapshot PBS (avant infection)                            | < 4h     |
| Panne totale (site principal + entrepôt) | Toute infra down            | Restauration depuis OVH S3 sur nouveau matériel                               | < 8h     |

- **Le PRA est testé trimestriellement** — les tests sont documentés et comparés aux RTO cibles.
- **Déclenchement :** pour tous les scénarios, le même mécanisme s'applique — le prestataire externe reçoit une alerte Zabbix, consulte le runbook (procédure documentée step-by-step), déclenche la bascule scriptée ou intervient selon le scénario identifié. Il n'y a pas de mécanisme automatique distinct à maintenir en parallèle pour le cas "panne de nœud".
- **Point clé :** avec l'ancienne architecture (serveur unique 2012), tous ces scénarios aboutissaient à la même issue — 48h d'arrêt. Avec la réplication et la bascule scriptée, seul le scénario de panne totale nécessite une restauration longue ; tous les autres se résolvent en quelques minutes à quelques heures via un runbook simple, sans complexité de cluster à quorum.
