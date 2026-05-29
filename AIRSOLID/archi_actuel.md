### Serveur physique (2012) — pièce maîtresse et point de défaillance unique
Un seul serveur physique héberge l'intégralité des services critiques : l'Active Directory (annuaire de l'entreprise), l'ERP web sous Docker, et les partages de fichiers Windows. Sa configuration exacte est inconnue du management. Une panne de 48h a déjà paralysé l'activité complète. C'est le risque numéro 1.

### Stockage

Un NAS est présent sur le réseau local, mais sans aucune politique de sauvegarde connue. Les données qu'il contient (volumétrie inconnue) ne sont ni dupliquées ni protégées. Aucune conformité RGPD n'est assurée.
Réseau local
Alimentation monophasée de 12 kVA — insuffisante pour héberger une infrastructure redondante ou une salle serveur sérieuse. La segmentation réseau (bureau / production / éventuellement entrepôt futur) n'est pas documentée.
Connectivité internet
Une seule fibre professionnelle à 1 Gbps. Pas de lien de secours : toute coupure opérateur = arrêt total pour les nomades et les services dépendant du cloud (M365 en déploiement).
Accès distant
Un VPN est en place pour les 40 commerciaux nomades, mais sa qualité de service n'a jamais été évaluée formellement. Aucun accès préstataire externe formalisé.
Applications
ServiceHébergementÉtatActive DirectoryServeur 2012En production, critiqueERP (app web Docker)Serveur 2012En production, critiquePartages fichiersServeur 2012En productionMicrosoft 365Cloud MicrosoftDéploiement en coursSite WordPressInterne (serveur ?)En production, non sécuriséTéléphonie sans filLocalType non préciséVidéosurveillanceLocalAccès distant non formalisé