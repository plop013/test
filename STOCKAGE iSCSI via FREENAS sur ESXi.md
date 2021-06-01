# STOCKAGE iSCSI via FREENAS sur ESXi

> Mise en place d'un hyperviseur de Type 1 sous forme de VM sur un cluster (nested) avec une banque de données distante accessible via Ethernet avec le protocole iSCSI derrière un PFSENSE.  
> J'utiliserais donc FreeNAS pour cet espace distant avec la création d'une NIC VMkernel dédié (dans ESXi) pour la gestion ainsi qu'une NIC dédiée au transport des données via le NAS (LUN)



## Création et installation de la VM ESXi

Sur le cluster, je crée donc une 1ere VM ESXi avec les caractéristiques suivantes : 
* CPU = 2
* RAM = 4
* DD1 = 8Go (en dynamique)
* 2 adaptateurs réseaux
* Mettre en ip fixe (VIA F2 après installation)
* Je met une règle de NAT sur mon pfsense afin que je puisse me connecter via mon PC sur l'ESXi

Pour l'installation, je choisis le disque (ici il n'y en a q'un seul), clavier en francais, et je choisis un mot de passe respectant les normes demandés. Après avoir fini, on déconnecte le CD et reboot. Au redémarrage, je fais F2 pour rentrer dans les paramètres systèmes puis je vais dans ***configure management network*** et je passe l'ip en statique. Si besoin, j'active le Shell et/ou le SSH dans ***troubleshooting option*** pour y accéder via un autre poste. Je récupère l'adresse via la page principale afin de m'y connecter via HTTPS. 

---

## Création et installation de la VM FreeNAS

Toujours sur le cluster, je crée une 2eme VM pour y installer FreeNAS avec ces caractéristiques : 
* Sytème = Autre / FreeBSD 11
* CPU = 2
* RAM = 8
* Disque 1 = 8 Go (dynamiques)
* 3 disques de 40 Go (dynamiques)
* ISO = FreeNAS-11.3-U4


Au lancement de la VM, je choisi Install/Upgrade, puis je sélectionne le disque de 8Go pour installer le sytème FreeNAS.   
Je met ensuite un password (faire attention si clavier en QWERTY ou non). Je choisis le ***boot via bios*** (pour une meilleur compatibilité des VMs).  
A la fin de l'installation, je déconnecte le CD et je reboot.
Je récupère l'adresse IP fourni au redémarrage et je me connecte au Freenas via HTTPS (ou HTTP).
Comme pour l'ESXi, je NAT l'adresse dans pfSense : 10.30.10.XXX :XXXX> 192.168.10.XXX :XXXX

---

## Paramètres Réseaux basiques
Dans un 1er temps, je vais fixer l'IP de mon FreeNAS. Pour cela je fais de cette manière :
1. Network > Interfaces, expand le vmx0 puis *EDIT*
2. On désactive le DHCP
3. On ajoute notre ip fixe du FreeNAS

Tips : Fixer aussi l'adresse de la gateway en allant dans Network > Global Configuration.

## Paramétrage de l'iSCSI dans FreeNAS
Une fois les paramètres réseaux basiques réglés, je vais activés le protocole iSCSI en me rendant dans le menu Services. Puis j'active iSCSI ainsi que le redémarage automatique de celui au reboot

Pour créer notre espace de stockage, je vais créer un volumes zvol (système de fichier ZFS dans FreeNAS) dans un pool de mes 3 HDDS de 40Go :

1. Storage > pool puis ADD. Créer le pool avec les 3 disques. Je choisis de laisser en raid-z (équivalent au Raid-5)
2. Je créer ensuite un Zvol
    * Sur la ligne du pool créé précédement, je clic sur les trois points à droite pour ***Ajouter un zvol***
    * Je donne un nom + la taille souhaité et je coche l'option ***sparse*** pour le thin provisionning (dynamique)

Je crée ensuite le partage afin d'y accéder via l'ESXi. Je vais dans Sharing > block shares (iSCSI) et je remplis les champs suivants : 

1. Dans Portal = nom + ip créer pour le stockage
2. Dans Initiators = Allow ALL + nom
3. Dans Target = nom + portal + initiator
4. Dans Extents = nom + type (Device) + décocher TPC + cocher Enabled puis SAVE
5. Dans Associated Target = choisir Target et Extent, donner un LUN (Logical Unit Number) ID

Afin de pouvoir accéder a ce nouveau stockage, nous devons lui attribuer une adresse IP. Je retourne donc dans network > interfaces et je rajoute une IP fixe a la suite de notre IP du FreeNAS.
## Dans l'ESXi
Enfin pour finir, je vais sur l'ESXi et je crée un nouveau switch virtuel via ***networking > onglet add virtual switches > add standard*** puis je lui donne un nom (par exemple vSwitch-iSCSI) et lui ajoute notre 2e carte réseau qui servira à l'échange de données entre le FreeNAS et ESXi.  
J'ajoute une NIC VMkernel avec comme configuration suivante :   

1. new port group
2. un nom (par exemple vSwitch-iSCSI-port)
3. je choisis le vSwitch crée juste avant
4. dans la partie iPV4, je met une IP FIXE   

Dans le menu de gauche, je clic sur storage puis onglet ***adapters*** et ***software iSCSI*** que l'on configure de cette manière : 
1. on l'active (enabled)
2. on clic sur add port binding et dans la fenêtre suivante je choisi la VMKernel NIC créée précédement et select. 
3. Pour finir la configuration, je clic sur ***add dynamic target*** et j'ajoute l'ip du stockage ZVOL créé sur le FreeNAS.

Notre stockage apparait maintenant dans  ***storage > onglet device***. Pour l'utiliser, il nous suffit maintenant de créer un nouveau datastore et penser a choisir VMFS 5 afin d'assurer la compatibilité (nous avions configurer le FreeNAS en boot par BIOS, qui a certaines limitations par rapport a l'UEFI)
