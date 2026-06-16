# Architecture.

voici l'architecture de l'infra actuel de airsolid.

![alt text](./assets/archi_actuel.png)

## Archi on prem proposée :

![alt text](./assets/archi_onprem.png)

archi on prem :

2 serveur physiques : 

1 salle serveur déjà existantes 
1 salle serveur dans l'entrepot en cours d'acquisition. 

pour le budjet on a pris : 

serveur hp 10eme gen avec 128go de ram et 50to de stockage. 

4 000 à 10 000 € achat d'un serveur 
Maintenance / pannes / disques : ~300–1 000 €/an



On serait sur une proposition de passif/actif avec celui de l'entrepot 2 qui servirait de passif si l'actif tombe. 
temps de reprise en HA attendu a 30sec / 5 mins

## archi hybride physique/cloud ha

![alt text](./assets/archi_hybride.png)

archi hybride : 

1 serveur physique dans la salle server existante. 

même server que pour la solution onprem

Côté cloud : 

A. Site miroir (HA simple) :

copie du site interne
hébergée sur 1–2 VM cloud
 - si ton serveur tombe :
site continue en cloud
 - 1 à 10 minutes de bascule

B. Active Directory secondaire
1 contrôleur AD dans le cloud
réplication automatique
si serveur HS :
authentification continue

C. VPN de secours
instance VPN cloud (WireGuard / OpenVPN / Azure VPN)
activation automatique ou manuelle rapide

D. Sauvegardes des données

backup complet + incrémental
stockage object storage (OVH / S3 / équivalent)
versioning + historique
10–50 To en backup
pas en usage actif

E. ERP ( access, microsoft 365)

réplication quasi temps réel de l’ERP
serveur principal = actif
cloud = miroir chaud (warm standby)

### Coût

Serveur physique : 

~ 6 000 à 20 000 €/ 5 ans avec maintenance et coût de l'elec.


Cloud minimal HA/hybride :

VM standby : 50–300 €/mois
AD + VPN cloud : 50–150 €/mois
stockage 50 To backup : 500–1 200 €/mois

total = 600-1600€/mois et 7200-19200€/5ans 

( si panne totale = plus cher )


### Conclusion 

👉 Le cloud est :

✔ flexible
✔ scalable
✔ sans maintenance
❌ très cher pour 50 To constants

👉 Le serveur HP est :

✔ énormément moins cher à long terme
✔ parfait pour charge stable
❌ moins flexible
❌ maintenance à gérer