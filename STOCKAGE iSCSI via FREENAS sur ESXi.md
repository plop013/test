# STOCKAGE iSCSI via FREENAS sur ESXi

> (lab effectué dans le cluster WF3)



## Création de la VM ESXi

* CPU = 2
* RAM = 4
* DD1 = 8Go (en dynamique)
* 2 adaptateurs réseaux
* Mettre en ip fixe (VIA F2 après installation)
* Faire une règle NAT sur pfsense


## Création de la VM FreeNAS

1. Créer une nouvelle machine virtuelle
2. Sytème = Autre / FreeBSD 11
3. CPU = 2
4. RAM = 8
5. Disque 1 = 8 Go (dynamiques)
6. 3 disques de 40 Go (dynamiques)
7. ISO = FreeNAS-11.3-U4
8. Install/Upgrade
9. sélection du disque avec touche ESPACE 
    * mettre un password (Attention que le clavier est peut etre en qwerty, prendre un password fonctionnant sur les 2 claviers) = Cisco
    * choisir boot via bios (pour VM souvent moins galère même si moins sécurisé)
    * ne pas se préoccuper de l'erreur 22
10. Eteindre
11. Déconnecter le CD
12. Rejancer
13. lire l'adresse IP et l'entrer dans un navigateur log/pass = root/Cisco
14. Natter l'adresse dans pfSense : 10.30.10.XXX :XXXX> 192.168.10.XXX :XXXX

## Paramètres Réseaux

1. Network > Interfaces, expand le vmx0 puis *EDIT*
2. On désactive le DHCP
3. On ajoute notre ip fixe du FreeNAS et celle du stockage (que l’on va créer)

## Paramétrage de l'iSCSI dans FreeNAS

1. Services > iSCSI activé (redémmarage)
2. Storage > pool puis ADD. Créer le pool avec les 3 disques
3. Créer un Zvol
    * Sur la ligne du pool créé, les trois ponts à droite pour
    * Nom + Taille sparse pour le thin provisionning (dynamique)
4. Partager en iSCSI
5. sharing > block Shares (iSCSI)
6. Portal = nom + ip créer pour le stockage
7. Initiators = Allow ALL + nom
8. Target = nom + portal + initiator
9. Extents = nom + type (Device) + décocher TPC + cocher Enabled puis SAVE
10. Associated Target = choisir Target et Extent, donner un LUN ID


## Dans l'ESXi

1. Créer un nouveau vSwitch
2. Ajouter une VMkernel NIC = New groupe port + IP FIXE
3. Stockage > adaptateur > iSCSI 
    * Enabled
    * Ajouter liaison de port (VMkernel)
    * Ajouter cible dynamique (stockage FreeNAS)
    * Le stockage Freenas apparait dans ESXi
