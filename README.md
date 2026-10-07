# Project SIOSISR

#PFSENSE
BTS SIO option SISR
TP pfSense / Addon
Package de monitoring, bureau à distance via NAT et serveur web en DMZ
1. Contexte et objectifs
La société LSL vient de faire installer pfSense, avec un accès au serveur web situé dans la DMZ. Le DSI demande deux évolutions :
•	Monitoring : pouvoir surveiller depuis un navigateur le maximum d’informations sur le réseau LAN (choix et installation d’un package pfSense).
•	Bureau à distance : installer un serveur Windows dans la DMZ, accessible en RDP depuis Internet. Pour des raisons de sécurité, un port différent de 3389 est utilisé côté WAN, redirigé vers le port par défaut du serveur (@WAN_pfSense:port choisi vers @WS:3389).
L’ensemble a été reproduit dans un laboratoire virtuel sous Proxmox. Ce compte rendu présente les choix effectués, la démarche détaillée, les commandes utilisées et les validations. Une partie complémentaire décrit l’ajout du serveur web LAMP dans la DMZ, présent dans la mise en situation.
2. Infrastructure mise en place
2.1 Schéma réseau
 
Figure 1 — Schéma de l’infrastructure (WAN, LAN, DMZ) et redirections NAT
2.2 Plan d’adressage
Segment	Bridge Proxmox	Interface pfSense	Réseau
WAN	vmbr0	vtnet0	DHCP, 192.168.20.0/24 (pfSense : 192.168.20.77)
LAN	lanvmbr1	vtnet1	192.168.10.0/24 (pfSense : .1, client Debian : .10)
DMZ	dmzvmbr2	vtnet2 (OPT1)	172.16.10.0/24 (pfSense : .1, LAMP : .10, serveur Windows : .20)

2.3 Machines utilisées
Machine	Emplacement	Description
pfSense (VM 108)	WAN / LAN / DMZ	pfSense CE 2.9.0, 4 vCPU, disque 32 Go, 3 cartes réseau VirtIO
Client Debian	LAN	192.168.10.10/24, navigateur Firefox, sert à administrer pfSense et à consulter ntopng
Serveur Windows	DMZ	172.16.10.20/24, bureau à distance activé
LAMP-DMZ	DMZ	Conteneur Proxmox TurnKey LAMP, 172.16.10.10/24, Apache
PC physique	Côté WAN	192.168.20.108, utilisé pour simuler un accès depuis l’extérieur

La VM pfSense possède trois cartes réseau VirtIO, chacune reliée à un bridge dédié :
 
Figure 2 — Cartes réseau de la VM pfSense : net0 sur vmbr0 (WAN), net1 sur lanvmbr1 (LAN), net2 sur dmzvmbr2 (DMZ)
 
Les bridges lanvmbr1 et dmzvmbr2
3. Installation de pfSense
3.1 Démarrage de l’installeur
La VM démarre sur l’ISO Netgate. Le menu de démarrage s’affiche, puis l’installeur demande d’accepter la licence.
 
Figure 3 — Menu de démarrage de l’ISO pfSense
 
Figure 4 — Acceptation de la licence (Accept)
3.2 Configuration réseau pendant l’installation
L’installeur demande de configurer les interfaces WAN et LAN. Le WAN est laissé en DHCP afin de disposer d’un accès à Internet pendant l’installation. Le LAN est configuré en adresse statique, sans serveur DHCP.
 
Figure 5 — WAN (vtnet0) : mode DHCP client
 
Figure 6 — LAN (vtnet1) : statique 192.168.10.1/24, DHCP désactivé
 
Figure 7 — Confirmation de l’assignation des interfaces (WAN = vtnet0, LAN = vtnet1)
3.3 Choix de l’édition et de la version
L’installeur propose d’abord pfSense Plus, qui nécessite un abonnement actif. Aucun abonnement n’étant disponible, c’est l’édition gratuite pfSense CE (Community Edition) qui est installée, dans sa version stable la plus récente, la 2.9.0.
 
Figure 8 — Validation de l’abonnement : choix de « Install CE »
 
Figure 9 — Choix de la version : 2.9.0 (Current Stable Version)
3.4 Premier démarrage
À la fin de l’installation, le lecteur CD/DVD de la VM est éjecté dans Proxmox (Hardware > CD/DVD Drive > Do not use any media) puis la VM redémarre. La console affiche le menu de pfSense avec le WAN en DHCP (192.168.20.77) et le LAN en 192.168.10.1.
 
Figure 10 — Menu de la console pfSense après le premier démarrage
3.5 Ajout de l’interface DMZ
L’installeur ne gère que le WAN et le LAN. La troisième interface (vtnet2) est donc assignée depuis la console, avec l’option 1 puis l’option 2.
1.	Option 1 (Assign Interfaces) : VLAN : non ; WAN = vtnet0 ; LAN = vtnet1 ; Optional 1 = vtnet2 ; confirmation par « y ».
2.	Option 2 (Set interface(s) IP address) : interface OPT1, IPv4 par DHCP : non, adresse 172.16.10.1, masque 24, pas de passerelle, IPv6 : non, serveur DHCP : non, retour en HTTP : non.
 
Figure 11 — Assignation de OPT1 sur vtnet2
 
Figure 12 — Configuration de l’adresse IPv4 de OPT1 : 172.16.10.1/24
 
Figure 13 — Console avec les trois interfaces : WAN, LAN et DMZ (OPT1)
3.6 Accès à l’interface web
Le client Debian du LAN (192.168.10.10) accède à l’interface web par https://192.168.10.1 (compte admin, mot de passe modifié lors de l’assistant de configuration). Le nom d’hôte configuré est pfSense et le domaine lsl.local. L’interface OPT1 a ensuite été activée et renommée DMZ (Interfaces > OPT1).
 
Figure 14 — Page de connexion de pfSense depuis le client LAN
 
Figure 15 — Tableau de bord : pfSense CE 2.9.0, interfaces WAN 192.168.20.77, LAN 192.168.10.1 et DMZ 172.16.10.1
4. Configuration du pare-feu
4.1 Interface WAN
Le WAN est situé sur un réseau privé (192.168.20.0/24). Par défaut, pfSense bloque les réseaux privés sur le WAN, ce qui empêcherait l’accès depuis le PC physique, lui aussi en 192.168.20.x. Dans Interfaces > WAN, l’option « Block private networks » est donc décochée, puis Save et Apply Changes.
L’option « Block bogon networks » est laissée activée : les bogons sont des plages non attribuées par l’IANA, ce qui ne concerne pas 192.168.20.0/24. Cette règle apparaît toujours dans les règles du WAN (voir la figure de la section 6.3) et elle ne gêne pas les tests.  
4.2 Règles du LAN
Les règles par défaut sont conservées : la règle anti-verrouillage et l’autorisation de tout le trafic du LAN (IPv4 et IPv6).
 
Figure 16 — Règles de l’interface LAN
4.3 Règle de la DMZ
Par défaut une interface optionnelle ne laisse rien passer. Une règle est ajoutée pour que les serveurs de la DMZ puissent sortir vers Internet (mises à jour, DNS), tout en leur interdisant l’accès au réseau interne. C’est le principe d’une DMZ : si un serveur exposé est compromis, il ne doit pas permettre d’atteindre le LAN.
Paramètre	Valeur
Action / Interface	Pass / DMZ
Famille / Protocole	IPv4 / Any
Source	DMZ subnets
Destination	LAN subnets avec « Invert match » (tout sauf le LAN)
Description	DMZ vers Internet, LAN interdit

 
Figure 17 — Formulaire de la règle DMZ
 
Figure 18 — Règle appliquée sur l’interface DMZ (Apply Changes)
4.4 Validation de la connectivité du LAN
Depuis le client Debian, un ping vers 8.8.8.8 aboutit : le LAN accède à Internet en traversant pfSense (NAT sortant et règle LAN).
ping 8.8.8.8
 
Figure 19 — Ping du client LAN vers 8.8.8.8 : 5 paquets reçus sur 5
5. Partie 1 : mise en place d’une solution de monitoring
5.1 Étape 1 : choix de la solution
Le DSI veut consulter via un navigateur le maximum d’informations sur le LAN. Quatre solutions disponibles sous forme de package pfSense ont été comparées.
Solution	Interface web	Avantages	Limites
ntopng	Oui	Trafic en temps réel, hôtes, protocoles applicatifs, flux, classement des plus gros consommateurs, alertes	Version communautaire limitée
bandwidthd	Oui	Très léger, bande passante par adresse IP	Peu d’informations
darkstat	Oui	Léger, statistiques par hôte et par port	Moins détaillé
Zabbix agent / Telegraf	Non (serveur nécessaire)	Supervision complète	Nécessite une infrastructure supplémentaire

Choix retenu : ntopng. Il répond directement à la demande : il s’installe comme package pfSense, s’utilise depuis un navigateur sans serveur supplémentaire et fournit le plus d’informations sur le LAN (hôtes, protocoles détectés, flux actifs, débit, top talkers).
5.2 Étape 2 : installation et configuration
Installation du package
System > Package Manager > Available Packages, recherche de « ntopng », puis Install et Confirm.
 
Figure 20 — ntopng 6.2.0_7 installé et ses dépendances (Installed Packages)
Configuration
La configuration se fait dans Diagnostics > ntopng Settings. Enable ntopng est coché et un mot de passe administrateur est défini.
Paramètre	Valeur
Server Interface	LAN (écoute IPv4)
Monitored Interfaces	LAN (le WAN n’est pas surveillé, comme recommandé)
Promiscuous Mode	Désactivé
DNS Mode	Decode DNS responses and resolve all numeric IPs
Additional Local Networks	192.168.10.0/24 et 172.16.10.0/24

 
Figure 21 — Page Diagnostics > ntopng Settings
Accès à l’interface
Depuis le client LAN, ntopng est accessible sur https://192.168.10.1:3000 (compte admin). Le navigateur affiche un avertissement car le certificat de pfSense est auto-signé. Il est attendu dans un environnement de test et il suffit de choisir « Continuer ». En production, un certificat émis par une autorité de confiance serait installé.
 
Figure 22 — Avertissement de certificat auto-signé (SEC_ERROR_SELF_SIGNED_CERT)
5.3 Validation
Du trafic est généré depuis le client LAN (ping, navigation, consultation de ntopng), puis vérifié dans ntopng.
ping 8.8.8.8
 
Figure 23 — Trafic généré depuis le client LAN
 
Figure 24 — Tableau de bord ntopng : flux principaux, hôtes, applications et classification du trafic
 
Figure 25 — Onglet Hosts : le client 192.168.10.10, pfSense (pfsense.lsl.local) et 8.8.8.8 (dns.google)
 
Figure 26 — Onglet Flows : connexions actives sur l’interface vtnet1 (LAN)
ntopng détecte le client, les serveurs contactés, les protocoles (TLS, DNS, ICMP) et affiche les flux en temps réel avec leur débit. Les flux TLS vers pfSense sur le port 3000 correspondent à la consultation de ntopng elle-même. La demande du DSI est satisfaite : le LAN est supervisé depuis un simple navigateur.
6. Partie 2 : bureau à distance via NAT
6.1 Serveur Windows dans la DMZ
Le serveur Windows est connecté au bridge dmzvmbr2, avec une adresse fixe dans la DMZ : 172.16.10.20, passerelle 172.16.10.1 (pfSense), DNS 172.16.10.1.
 
Figure 27 — Propriétés IPv4 du serveur Windows
Le bureau à distance est activé dans les propriétés système (Utilisation à distance), avec l’option d’authentification NLA laissée cochée. Le compte Administrateur est utilisé pour la connexion.
 
Figure 28 — Activation du bureau à distance sur le serveur
6.2 Choix du port
Le port 3389 (RDP par défaut) est très scanné sur Internet. Le port WAN choisi est le 33890. Ce choix réduit le bruit des scans automatiques mais ne constitue pas une protection suffisante en lui-même : c’est de la sécurité par l’obscurité, à compléter par d’autres mesures (voir la conclusion). Le serveur continue d’écouter sur le port par défaut 3389.
6.3 Règle de redirection NAT
Firewall > NAT > Port Forward > Add, avec les paramètres suivants, puis Save et Apply Changes.
Paramètre	Valeur
Interface	WAN
Protocole	TCP
Destination	WAN address
Destination port range	33890
Redirect target IP	172.16.10.20
Redirect target port	3389
Description	RDP vers WS DMZ
Filter rule association	Add associated filter rule

 
Figure 29 — Formulaire de la redirection de port RDP
 
Figure 30 — Règle de redirection créée (Port Forward)
La règle de pare-feu associée est générée automatiquement sur le WAN : elle autorise le TCP vers 172.16.10.20 sur le port 3389. La règle « Block bogon networks » reste active, mais elle ne gêne pas le test puisque 192.168.20.0/24 est un réseau privé et non un bogon.
 
Figure 31 — Règles de l’interface WAN, dont la règle associée à la redirection RDP
6.4 Validation
La validation est faite depuis le PC physique (192.168.20.108), qui se trouve côté WAN. Le trafic traverse donc bien pfSense. Un test de port est d’abord réalisé en PowerShell :
Test-NetConnection 192.168.20.77 -Port 33890
 
Figure 32 — Test du port 33890 : TcpTestSucceeded = True
La connexion RDP est ensuite lancée avec mstsc, en saisissant 192.168.20.77:33890. Le serveur présente un certificat auto-signé (nom WIN-LDGBL85QQVS), ce qui provoque un avertissement attendu.
 
Figure 33 — Avertissement de certificat lors de la connexion RDP
La session s’ouvre sur le serveur. La commande ipconfig exécutée dans la session confirme qu’il s’agit bien du serveur de la DMZ (172.16.10.20, passerelle 172.16.10.1). Le ping vers le LAN (192.168.10.10) depuis ce serveur échoue, ce qui confirme l’isolation de la DMZ.
 
Figure 34 — Session RDP ouverte depuis 192.168.20.77:33890 : ipconfig affiche 172.16.10.20
7. Serveur web LAMP dans la DMZ
La mise en situation indique que le serveur web est situé dans la DMZ. Un serveur LAMP (Apache, MariaDB, PHP) est donc ajouté pour compléter la maquette, avec un accès depuis le LAN et depuis le WAN.
7.1 Création et configuration réseau
Le serveur est un conteneur Proxmox basé sur le modèle TurnKey LAMP, nommé LAMP-DMZ. Sa carte réseau eth0 est reliée au bridge dmzvmbr2, avec une adresse statique. Le pare-feu Proxmox de la carte est désactivé afin que seul pfSense filtre le trafic.
Paramètre	Valeur
Bridge	dmzvmbr2
IPv4	Statique 172.16.10.10/24
Passerelle	172.16.10.1 (pfSense)
Firewall Proxmox	Désactivé

 
Figure 35 — Configuration de la carte réseau du conteneur LAMP-DMZ
Le conteneur est redémarré pour appliquer le changement de bridge.
7.2 Tests de connectivité
Les tests suivants sont réalisés depuis la console du conteneur :
ip a
ping 172.16.10.1
ping 8.8.8.8
ping 192.168.10.10
Test	Résultat	Interprétation
ip a	172.16.10.10/24 sur eth0	Adresse correctement appliquée
ping 172.16.10.1	Réponse (0 % de perte)	Le serveur joint pfSense
ping 8.8.8.8	Réponse (0 % de perte)	La DMZ accède à Internet (règle DMZ et NAT sortant)
ping 192.168.10.10	100 % de perte	La DMZ ne peut pas joindre le LAN (règle d’isolation)

 
Figure 36 — Tests réseau depuis LAMP-DMZ
7.3 Service Apache
systemctl status apache2
Apache est actif (running) et activé au démarrage (enabled).
 
Figure 37 — Statut du service apache2
7.4 Accès depuis le LAN
La règle par défaut du LAN autorise le trafic vers la DMZ. Depuis le client Debian, la page d’accueil TurnKey LAMP s’affiche sur http://172.16.10.10.
 
Figure 38 — Page TurnKey LAMP vue depuis le client LAN
7.5 Publication du serveur web sur le WAN
Pour rendre le serveur web accessible depuis l’extérieur, une seconde redirection est créée dans Firewall > NAT > Port Forward, avec une règle de pare-feu associée.
Paramètre	Valeur
Interface / Protocole	WAN / TCP
Destination	WAN address, port 80 (HTTP)
Redirect target IP	172.16.10.10
Redirect target port	80 (HTTP)
Filter rule association	Add associated filter rule

 
Figure 39 — Redirection du port 80 vers le serveur LAMP
Depuis le PC physique, l’adresse http://192.168.20.77 affiche la page du serveur LAMP de la DMZ, ce qui valide la redirection.
 
Figure 40 — Page TurnKey LAMP vue depuis le PC physique via l’adresse WAN de pfSense
7.6 Remarque de sécurité
La page d’accueil TurnKey donne accès à Webmin (administration système) et à Adminer (administration de base de données). Publiée sur le WAN, elle les expose aussi. En production, seul le site serait publié et ces interfaces seraient désactivées ou restreintes à des adresses IP d’administration.
8. Problèmes rencontrés et solutions
Problème	Cause	Solution
L’installeur demande un abonnement	Le démarrage se fait en pfSense Plus	Choix de « Install CE » (Community Edition)
Plage DHCP du LAN en 192.168.1.x dans l’installeur	Valeurs par défaut	LAN fixé à 192.168.10.1/24 et DHCP désactivé
LAN et DMZ inversés sur le réseau	Les cartes net1 et net2 de la VM étaient sur les mauvais bridges	net1 remis sur lanvmbr1 et net2 sur dmzvmbr2
Pas d’interface DMZ après l’installation	L’installeur ne gère que WAN et LAN	Assignation de vtnet2 en OPT1 via la console (options 1 et 2)
Règle DMZ trop restrictive	Protocole TCP au lieu de Any : ni ping ni DNS (UDP)	Protocole passé à Any
Mot de passe admin oublié	—	Option 3 de la console (voir annexe)
Avertissements de certificat (ntopng, RDP)	Certificats auto-signés	Acceptés en environnement de test
Enregistrement du NAT web refusé	Le port (:80) avait été saisi dans le champ « Redirect target IP »	IP seule (172.16.10.10) dans ce champ, port 80 dans « Redirect target port »
Aucune ligne RDP dans les journaux du pare-feu	Le trafic autorisé n’est journalisé que si « Log » est coché sur la règle ; le WAN affiche surtout des blocages (multicast, IPv6 local)	Validation par le test de port, la session RDP et la page web
Message « could not identify which pfSense kernel is installed » au premier démarrage	Message du premier boot	Disparu après le redémarrage, sans effet
Masque 255.255.0.0 sur le serveur Windows	Valeur saisie différente du /24 prévu	Fonctionnel, car le réseau 172.16.10.0/24 est inclus ; à corriger en 255.255.255.0 pour rester cohérent avec le plan d’adressage

9. Conclusion et améliorations possibles
Les deux demandes du DSI sont réalisées : le LAN est supervisé depuis un navigateur avec ntopng, et le serveur Windows de la DMZ est joignable en RDP depuis le WAN par le port 33890 redirigé vers le port 3389. Le serveur web LAMP complète la DMZ, accessible depuis le LAN et depuis le WAN, tout en étant isolé du réseau interne.
Pour renforcer la sécurité d’une mise en production :
•	VPN : utiliser OpenVPN ou WireGuard plutôt que d’exposer le RDP sur Internet.
•	Filtrage par source : limiter la redirection RDP à des adresses IP autorisées.
•	Comptes : créer un utilisateur dédié au bureau à distance plutôt que d’utiliser le compte Administrateur, avec des mots de passe robustes.
•	Détection et blocage : pfBlockerNG ou Snort pour bloquer les tentatives de connexion suspectes.
•	Certificats : installer des certificats valides pour pfSense, ntopng et le RDP.
•	Serveur web : désactiver ou restreindre Webmin et Adminer côté WAN.
•	Supervision : ajouter l’interface DMZ aux interfaces surveillées par ntopng et activer la journalisation des règles sensibles.
Annexe : réinitialisation du compte admin de pfSense
En cas de perte du mot de passe, le compte admin peut être réinitialisé depuis la console de la VM (Proxmox > VM 108 > Console) :
1.	Taper 3 (Reset admin account and password) puis Entrée.
2.	Confirmer par « y ».
3.	Saisir puis confirmer un nouveau mot de passe, puis se reconnecter sur https://192.168.10.1.
 
Figure 41 — Console pfSense : option 3, réinitialisation du compte admin
