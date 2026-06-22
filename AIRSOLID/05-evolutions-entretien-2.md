# Évolutions — Entretien 2 (mise à jour des besoins)

## Décision d'hébergement

Suite au deuxième entretien, **le client confirme opter exclusivement pour une architecture on-premise**.

Les options cloud public et hybride ont été écartées. Les raisons principales :
- Maîtrise totale des données sensibles (équipements aérauliques, données clients)
- Coût récurrent (OpEx cloud) jugé moins pertinent qu'un investissement matériel amorti sur 5 ans
- Salle serveur existante au siège + salle serveur entrepôt en cours d'acquisition : infrastructure physique déjà engagée

L'architecture retenue repose sur **2 serveurs physiques en configuration passif/actif** :
- Site principal (siège) : serveur actif
- Entrepôt secondaire (en cours d'acquisition) : serveur passif — bascule automatique en cas de panne (RTO cible : 30 sec – 5 min)

---

## Partenaire logistique — Flux EDI

Le client intègre prochainement un **partenaire logistique externe** nécessitant :

- **Ouverture d'un flux EDI** (Échange de Données Informatisé) entre AirSolid et le partenaire
- **Accès cadré et tracé** : le partenaire ne doit accéder qu'aux ressources strictement nécessaires (principe du moindre privilège)

### Exigences techniques à prévoir

| Point                  | Exigence                                                                 |
| ---------------------- | ------------------------------------------------------------------------ |
| Protocole EDI          | AS2 ou SFTP sécurisé (à confirmer avec le partenaire)                   |
| Accès réseau           | DMZ dédiée ou VLAN isolé, pas d'accès direct au LAN interne             |
| Authentification       | Compte de service dédié, certificat ou clé SSH, pas de compte nominatif |
| Traçabilité            | Journalisation des échanges (logs horodatés, conservés 12 mois minimum) |
| Flux entrants/sortants | Ouverture pare-feu restreinte aux IP du partenaire uniquement           |
| RGPD                   | DPA à signer avec le partenaire ; données échangées à inventorier       |

---

## Sauvegarde & conformité RGPD

*(Inchangé par rapport au premier entretien)*

- Mise en place **urgente** d'une politique de sauvegarde (règle 3-2-1 minimum)
- Chiffrement des sauvegardes, registre des traitements, DPA si sous-traitants
- Durée de rétention à définir selon les typologies de données

---

## Haute disponibilité & continuité

*(Inchangé — renforcé par la décision on-prem)*

- Architecture HA active/passive entre site principal et entrepôt
- Redondance internet : lien de secours 4G/5G ou second opérateur à étudier
- Plan de reprise d'activité (PRA) à formaliser

---

## Mobilité & accès distant

*(Inchangé)*

- VPN actuel à évaluer (enquête de satisfaction commerciaux)
- Solution ZTNA (Zero Trust) recommandée pour les 40 commerciaux nomades
- Accès prestataires externes : traçabilité obligatoire

---

## ERP & infrastructure applicative

*(Inchangé)*

- ERP Odoo sous Docker/Podman, cluster PostgreSQL HA
- 2 instances applicatives ERP (SRV-LNX-ERP-01 / SRV-LNX-ERP-02)
- Base de données répliquée (streaming replication)

---

## Services complémentaires

*(Inchangé)*

- **KMS** : centralisation des licences Windows
- **Téléphonie** : migration VoIP recommandée (Teams, 3CX…)
- **Vidéosurveillance** : accès distant sécurisé à formaliser
- **Site WordPress** : hébergement externalisé recommandé

---

## Budget comparatif — On-Premise vs Hybride

Le client a souhaité disposer d'une comparaison chiffrée pour justifier la décision on-prem.

### Tableau comparatif sur 5 ans (~80 utilisateurs)

| Poste                       | On-Premise (retenu)           | Cloud hybride (écarté)       |
| --------------------------- | ----------------------------- | ---------------------------- |
| Serveurs / infrastructure   | 8 000–20 000 € (achat ×2)     | 4 000–10 000 € (×1 serveur)  |
| Maintenance & MCO           | 600–2 000 €/an                | 300–1 000 €/an               |
| Licences OS & logiciels     | 3 000–6 000 €/an              | 3 000–5 000 €/an             |
| Sauvegarde & stockage local | 1 500–3 000 €/an              | 1 000–2 000 €/an             |
| Services cloud (VM/backup)  | —                             | 7 200–19 800 €/an            |
| Connectivité (HA)           | 2 400–4 800 €/an              | 2 400–4 800 €/an             |
| **Total estimé / an**       | **~20 000–36 000 €/an**       | **~26 000–48 000 €/an**      |
| **Total sur 5 ans**         | **~100 000–180 000 €**        | **~130 000–240 000 €**       |

### Représentation visuelle (coût annuel estimé, fourchette médiane)

```
Coût annuel estimé — comparaison on-premise vs hybride
(en milliers d'euros, fourchette médiane)

50k €  |
       |
40k €  |                              ████████████ Hybride (~37k€/an)
       |
30k €  |  ████████████ On-Premise (~28k€/an)
       |
20k €  |
       |
10k €  |
       |
 0k €  +--------------------------------------------
              On-Premise                Hybride
              (retenu)                 (écarté)

  Économie estimée : ~9 000 €/an — soit ~45 000 € sur 5 ans
```

### Points de vigilance

- Le **on-premise est moins cher à long terme** pour une charge stable de ~80 utilisateurs
- Le cloud hybride implique un **coût récurrent élevé** pour le stockage de 10–50 To de backup
- Le serveur HP (10e gen, 128 Go RAM, 50 To stockage) s'amortit sur 5 ans : avantage CapEx confirmé
- La décision on-prem suppose une **salle serveur correctement dimensionnée** à l'entrepôt (alimentation, climatisation, accès physique sécurisé)

---

## Synthèse des évolutions post-entretien 2

| Sujet                        | Statut                                |
| ---------------------------- | ------------------------------------- |
| Architecture                 | On-premise exclusif — décision finale |
| Flux EDI partenaire logistique | À mettre en œuvre — spécifications à affiner avec le partenaire |
| Budget comparatif            | Documenté — on-prem plus économique sur 5 ans |
| Reste du périmètre           | Inchangé par rapport à l'entretien 1  |
