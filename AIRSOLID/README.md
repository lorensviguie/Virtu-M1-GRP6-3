# AirSolid — Compte rendu d'entretien & Analyse des besoins SI

---

## 1. Contexte

AirSolid est une entreprise avec des profils utilisateurs variés et une forte composante terrain (commerciaux nomades). L'infrastructure actuelle est peu documentée et génératrice de pannes récurrentes. Un nouveau site (entrepôt) est en cours d'ouverture, offrant une opportunité de repenser l'architecture globale. La direction est ouverte à une migration cloud tout ou partielle, sous contrainte budgétaire forte.

---

## 2. Cartographie des utilisateurs

| Profil          | Effectif | Postes            | Mobilité                  |
| --------------- | -------- | ----------------- | ------------------------- |
| Commerciaux     | 40       | Sans bureau fixe  | Nomades (100% distanciel) |
| Administratif   | 10       | 10 postes         | Sédentaires               |
| Bureau d'études | 20       | 20 postes         | Sédentaires               |
| Production      | 10       | 2 postes partagés | Sur site                  |
| **Total**       | **80**   | **32 postes**     |                           |

---

## 3. État de l'existant

### Infrastructure & réseau

| Élément          | État                                                             |
| ---------------- | ---------------------------------------------------------------- |
| Serveur physique | Présent mais **caractéristiques inconnues** — capacités limitées |
| Réseau           | Monophasé — alimentation **12 kVA** (non triphasé)               |
| Connectivité     | 1 fibre pro 1 Gbps                                               |
| VPN              | En place pour les commerciaux nomades                            |
| Sauvegarde       | **Inexistante**                                                  |
| Conformité RGPD  | Non assurée actuellement                                         |

### Applications & services

| Élément           | Détail                                                                              |
| ----------------- | ----------------------------------------------------------------------------------- |
| ERP               | Logiciel interne, **tourne sous Docker** — pas de contrainte technique particulière |
| Site web          | WordPress, développé en interne                                                     |
| Téléphonie        | Sans fil (à préciser : DECT, VoIP ?)                                                |
| Vidéosurveillance | Système de caméras en place                                                         |
| Licences Windows  | KMS envisagé mais non déployé                                                       |
| Messagerie / AD   | Non précisés — **à investiguer**                                                    |

### Points de douleur exprimés

- Pannes d'infrastructure récurrentes et pénalisantes
- Coupures internet impactant l'activité
- Pannes logicielles (ERP ?)
- VPN existant : **satisfaction des commerciaux à évaluer** (enquête à mener)
- 1 seul NAS en production — **risque de perte de données**
- Absence totale de sauvegarde

---

## 4. Analyse des risques

| Domaine        | Risque identifié                                                     | Niveau       |
| -------------- | -------------------------------------------------------------------- | ------------ |
| Données        | Aucune sauvegarde — perte totale possible en cas de panne            | 🔴 Critique |
| Conformité     | Données non sauvegardées et non conformes RGPD                       | 🔴 Critique |
| Disponibilité  | Serveur physique de capacité inconnue/limitée, SPOF                  | 🔴 Critique |
| Continuité     | Coupures internet = arrêt d'activité (nomades + ERP cloud potentiel) | 🟠 Élevé    |
| Sécurité accès | Accès prestataires externes non formalisé                            | 🟠 Élevé    |
| Stockage       | NAS unique sans redondance                                           | 🟠 Élevé    |
| Licences       | Gestion des licences Windows non structurée (pas de KMS)             | 🟡 Modéré   |
| Satisfaction   | VPN commercial : qualité de service non mesurée                      | 🟡 Modéré   |

---

## 5. Nouveau site (entrepôt)

Le nouvel entrepôt représente une opportunité stratégique :

- Possibilité d'y installer une **salle serveur secondaire**
- Architecture **HA (Haute Disponibilité)** entre le site primaire et l'entrepôt
- À étudier : réplication de données en temps réel, basculement automatique (failover)

---

## 8. Feuille de route

| Priorité | Action                                                 | Bénéfice attendu                      | Horizon   |
| -------- | ------------------------------------------------------ | ------------------------------------- | --------- |
| 🔴 P1   | Mise en place sauvegarde + conformité RGPD             | Protection données, conformité légale | 0–1 mois  |
| 🔴 P1   | Audit serveur physique & inventaire complet            | Base pour décision cloud/on-premise   | 0–1 mois  |
| 🔴 P1   | Enquête satisfaction VPN commerciaux                   | Décision de maintien ou remplacement  | 0–2 mois  |
| 🟠 P2   | Choix stratégie hébergement (cloud/hybride/on-premise) | Feuille de route infrastructure       | 1–3 mois  |
| 🟠 P2   | Redondance internet (lien de secours)                  | Continuité d'activité                 | 2–4 mois  |
| 🟠 P2   | Architecture HA site principal ↔ entrepôt              | Haute disponibilité                   | 3–6 mois  |
| 🟠 P2   | Déploiement KMS + formalisation accès prestataires     | Sécurité & gouvernance licences       | 3–6 mois  |
| 🟡 P3   | Migration / amélioration accès distant nomades (ZTNA)  | Expérience commerciaux + sécurité     | 4–8 mois  |
| 🟡 P3   | Audit & externalisation site WordPress                 | Réduction surface d'attaque           | 4–6 mois  |
| 🟡 P3   | Unification téléphonie (VoIP)                          | Réduction coûts, mobilité             | 6–12 mois |

---

## 9. Points ouverts à clarifier

- [ ] Inventaire précis du serveur physique (RAM, CPU, stockage, OS)
- [ ] Contenu et volumétrie des données sur le NAS
- [ ] Nature des coupures internet : opérateur, équipement réseau ?
- [ ] Connectivité fibre disponible sur le site entrepôt
- [ ] Alimentation électrique de l'entrepôt (capacité salle serveur)
- [ ] Logiciel de téléphonie actuel (DECT, VoIP, opérateur ?)
- [ ] Résultats de l'enquête de satisfaction VPN commerciaux
- [ ] Données personnelles traitées (périmètre RGPD à définir)

---
