# 3 - AIRSOLID — Distribution & legacy

## Identité

- **Secteur :** distribution d'équipements aérauliques et climatisation.
- **Effectif :** environ 80 personnes.
- **IT :** ancien responsable parti ; **aucune équipe IT interne**.

## Pourquoi ça change

- **Un seul serveur physique (2012)** : Active Directory, ERP web, partages fichiers.
- **Panne de 48 heures** récente : activité commerciale et expéditions bloquées.
- Déploiement **Microsoft 365** en cours.
- **Entrepôt secondaire** prévu dans les 12 mois (non encore ouvert).
- Aucune politique de sauvegarde connue du management.

## Organisation & sites

- Site principal (bureaux + logistique légère).
- Commerciaux **nomades** ; quelques postes atelier pour SAV léger.

## Logiciels & usages

| **Domaine**   | **Outil**                                       |
| ------------- | ----------------------------------------------- |
| Annuaire      | Active Directory                                |
| ERP           | Application web interne (sur le serveur unique) |
| Fichiers      | Partages Windows                                |
| Collaboration | Microsoft 365 (déploiement en cours)            |
| Web           | Site vitrine institutionnel                     |

## Contraintes business

- Direction veut **sortir de la dépendance** au serveur unique.
- Charte implicite : *« Plus jamais 48 h sans ERP »*.
- Partenaire logistique à venir : besoin futur d'un **flux EDI** sécurisé.

scalabilité ? ( pour les bureaux ) 
question sur l'entrepot, ou ? quel équipement nécessaire ? 
avis sur le cloud côté client ? français ou pas ?
et en barre metal qu'est ce que vous avez comme infrastructure ( Stockage ??,)
dimensions des baies ? watage dispo ? alimentation triphasé ? 
process sécurité RGPD ? 
réseau ? fibre ? 
budjet déjà défini pour l'infra,   annuel pour du cloud ? 
VPN ?