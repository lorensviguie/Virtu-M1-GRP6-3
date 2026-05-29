# Architecture Proposée.*

voici l'architecture de l'infra actuel de airsolid.

![alt text](./assets/archi_actuel.png)

# Archi on prem proposée :

![alt text](./assets/archi_onprem.png)

archi on prem :

2 serveur physiques : 

1 salle serveur déjà existantes 
1 salle serveur dans l'entrepot en cours d'acquisition. 

pour le budjet on a pris : 

serveur hp 10eme gen avec 128go de ram et 50to de stockage. 
~ 6 000 à 20 000 €/ 5 ans avec maintenance et coût de l'elec.

On serait sur une proposition de passif/actif avec celui de l'entrepot 2 qui servirait de passif si l'actif tombe. 
temps de reprise en HA attendu a 30sec / 5 mins

# archi hybride physique/cloud ha

![alt text](./assets/archi_hybride.png)

archi hybride : 

1 serveur physique dans la salle server existante. 

même server que pour la solution onprem

cloud en mode filet de sécu. sauvegarde quotidiennes des données grâce au object storage.
pannes courte, électricité / reboot serveur pas de reprise d'activité sur le cloud. 

panne disque / serveur HS, bascule partielle :

- AD cloud prend relais immédiatement
- VPN bascule cloud
- ERP redémarre sur VM cloud
- fichiers restaurés depuis sync récente

catastrophe totale (serveur mort),  cloud devient production :

-VM cloud “cold standby” activée
-restauration des dernières données
-DNS bascule

### Coût

Serveur physique : 

~ 6 000 à 20 000 €/ 5 ans avec maintenance et coût de l'elec.


Cloud minimal HA/hybride :

VM standby : 50–300 €/mois
AD + VPN cloud : 50–150 €/mois
stockage 50 To backup : 500–1 200 €/mois

total = 600-1600€/mois

( si panne totale = plus cher )
