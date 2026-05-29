## Contextes et besoins.

### Stratégie d'hébergement (cloud vs on-premise)

Le client est ouvert aux deux options.

- **Cloud public** (Azure, AWS, OVHcloud…) : réduit les pannes matérielles, scalable, OpEx
- **On-premise / salle serveur entrepôt** : maîtrise des données, CapEx, adapté à l'ERP Docker
- **Hybride** : ERP et données sensibles on-premise, services bureautiques et nomades sur cloud

### Sauvegarde & conformité RGPD
- Mise en place **urgente** d'une politique de sauvegarde (règle 3-2-1 minimum)
- Chiffrement des sauvegardes, registre des traitements, DPA si sous-traitants cloud
- Durée de rétention à définir selon les typologies de données

### Haute disponibilité & continuité
- Redondance internet : ajout d'un **lien de secours** (4G/5G ou second opérateur) ( à voir )
- Architecture HA entre site principal et entrepôt
- Plan de reprise d'activité (PRA) à formaliser

### Mobilité & accès distant
- Remplacement ou amélioration du VPN actuel selon résultats de l'enquête de satisfaction
- Envisager une solution **ZTNA** (Zero Trust Network Access) pour les 40 commerciaux
- Accès sécurisé et tracé pour les prestataires externes (SI, maintenance)

### ERP & infrastructure applicative
- ERP Docker : portage facilité vers cloud (Kubernetes, VM dédiée)
- À documenter : dépendances réseau, volumes de données, SLA requis

### Services complémentaires
- **KMS** : déploiement pour centraliser la gestion des licences Windows
- **Téléphonie** : migration vers solution VoIP unifiée (Teams, 3CX…) recommandée
- **Vidéosurveillance** : vérifier compatibilité et accès distant sécurisé (entrepôt)
- **Site WordPress** :c hébergement externalisé recommandé

### Coûts estimatifs annuels (base ~80 utilisateurs)

| Poste                      | On-Premise                    | Cloud hybride        | Cloud full           |
| -------------------------- | ----------------------------- | -------------------- | -------------------- |
| Serveurs / infrastructure  | 8 000–15 000 € (amort. 5 ans) | 4 000–8 000 €        | 0 €                  |
| Licences OS & logiciels    | 3 000–6 000 €                 | 3 000–5 000 €        | 2 000–4 000 €        |
| Sauvegarde & stockage      | 1 500–3 000 €                 | 1 000–2 000 €        | 800–1 500 €          |
| Maintenance & MCO          | 5 000–10 000 €                | 3 000–6 000 €        | Inclus SLA           |
| Connectivité (HA)          | 2 400–4 800 €                 | 2 400–4 800 €        | 2 400–4 800 €        |
| Services cloud (IaaS/SaaS) | —                             | 6 000–12 000 €       | 12 000–20 000 €      |
| **Total estimé / an**      | **~20 000–39 000 €**          | **~19 000–38 000 €** | **~17 000–30 000 €** |

**Points de vigilance budget :**
- Le cloud **réduit les pannes matérielles** mais génère un coût récurrent (OpEx) vs investissement ponctuel (CapEx)
- Le cloud full dépend fortement de la connectivité internet — nécessite lien de secours
- L'entrepôt avec salle serveur peut amortir un cloud hybride sur 5–7 ans