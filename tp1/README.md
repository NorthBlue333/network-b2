# TP1 : Back to basics

# Sommaire

- [TP1 : Back to basics](#tp1--back-to-basics)
- [Sommaire](#sommaire)
- [I. Gather informations](#i-gather-informations)
  - [II. Edit configuration](#ii-edit-configuration)
    - [1. Configuration cartes réseau](#1-configuration-cartes-r%c3%a9seau)
    - [2. Serveur SSH](#2-serveur-ssh)
- [III. Routage simple](#iii-routage-simple)
- [IV. Autres applications et métrologie](#iv-autres-applications-et-m%c3%a9trologie)
  - [1. Commandes](#1-commandes)
  - [2. Cockpit](#2-cockpit)
  - [3. Netdata](#3-netdata)

# I. Gather informations

**Première étape : récupération d'infos sur le système.**  

* 🌞 récupérer une **liste des cartes réseau** avec leur nom, leur IP et leur adresse MAC :

**Commande :** `ip a`

**Résultat :**
```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000 #nom
    link/ether 08:00:27:12:e0:e7 brd ff:ff:ff:ff:ff:ff #adresse MAC
    inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic noprefixroute enp0s3 #adresse IP
       valid_lft 85685sec preferred_lft 85685sec
    inet6 fe80::fd60:7592:daf0:6335/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000 #nom
    link/ether 08:00:27:21:aa:29 brd ff:ff:ff:ff:ff:ff #adresse MAC
    inet 192.168.56.102/24 brd 192.168.56.255 scope global dynamic noprefixroute enp0s8 #adresse IP
       valid_lft 622sec preferred_lft 622sec
    inet6 fe80::37c1:9e38:1360:9b30/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
```
* 🌞 déterminer si les cartes réseaux ont récupéré une **IP en DHCP** ou non

Il y a plusieurs méthodes pour savoir si l'IP est attribuée par un DHCP, comme par exemple regarder le BOOTPROTO des fichiers correspondants aux interfaces réseaux dans `/etc/sysconfig/network-scripts/`, qui sera `dhcp` dans ce cas, ou encore regarder la liste des baux dans `/var/lib/NetworkManager`. Les IPs sont attribuées dynamiquement (mot clé `dynamic` dans le résultat de la commande `ip a`).

  * si oui, affichez le bail DHCP utilisé par la machine
**NAT :**
```
cat /var/lib/NetworkManager/internal-9ca56c00-391a-4117-9d80-0aaf6a4ae647-enp0s3.lease
# This is private data. Do not parse.
ADDRESS=10.0.2.15
NETMASK=255.255.255.0
ROUTER=10.0.2.2
SERVER_ADDRESS=10.0.2.2
NEXT_SERVER=10.0.2.4
T1=43200
T2=75600
LIFETIME=86400
DNS=10.33.10.20 10.33.10.2 8.8.8.8 8.8.4.4
DOMAINNAME=auvence.co
CLIENTID=0108002712e0e7
```
**VHOST :**
```
cat /var/lib/NetworkManager/internal-8ca712aa-07de-4659-9174-67d4b1e4823d-enp0s8.lease
# This is private data. Do not parse.
ADDRESS=192.168.56.102
NETMASK=255.255.255.0
SERVER_ADDRESS=192.168.56.100
T1=600
T2=1050
LIFETIME=1200
CLIENTID=0108002721aa29
```
* 🌞 afficher la **table de routage** de la machine et sa **table ARP**
**[Routage] Commande :** `ip route`
**[Routage] Résultat :**
**[ARP] Commande :** `ip neigh`
**[ARP] Résultat :**
  * expliquez chacune des lignes des deux tables 
  * *"cette route est celle de la carte XXX, elle est utilisée pour une connexion (locale|externe), la passerelle de cette route est à l'IP XXX et cette IP est portée par XXX"* par exemple
* 🌞 récupérer **la liste des ports en écoute** (*listening*) sur la machine (TCP et UDP)
  * trouver/déduire la liste des applications qui écoutent sur un port TCP ou UDP sur la machine (au moins un serveur SSH)
* 🌞 récupérer **la liste des DNS utilisés par la machine**
  * effectuez une requête DNS afin de récupérer l'adresse IP associée au domaine `www.reddit.com` ~~(parce que c'est important d'avoir les bonnes adresses)~~
  * dans le retour de cette requête DNS, vérifier que vous utilisez bien les bons DNS renseignés sur votre machine
* 🌞 afficher **l'état actuel du firewall**
  * quelles interfaces sont filtrées ?
  * quel port TCP/UDP sont autorisés/filtrés ?
  * 🐙 sous CentOS8, ce n'est plus `iptables` qui est utilisé pour manipuler le filtrage réseau mais `nftables`. Jouez un peu avec `nft` et affichez les "vraies" règles firewall (`firewalld`, manipulé avec `firewall-cmd` n'est qu'une surcouche à `nft`)

## II. Edit configuration

**Deuxièmement : Modifier la configuration existante**

---

### 1. Configuration cartes réseau

**NB** : sur CentOS8, la gestion des cartes réseau a légèrement changé. Il existe un démon qui gère désormais tout ce qui est relatif au réseau : NetworkManager.  

Marche à suivre pour modifier la configuration d'une carte réseau :
* édition du fichier de configuration
  * `sudo vim /etc/sysconfig/network-scripts/ifcfg-enp0s8`
* refresh de NetworkManager ("Hey prend mes modifications en compte stp !")
  * `sudo nmcli connection reload` 
  * `sudo nmcli con reload` même chose, on peut abréger les commandes `nmcli`
  * `sudo nmcli c reload` même chose aussi
* restart de l'interface
  * `sudo ifdown enp0s8` puis `sudo ifup enp0s8`
  * **OU** `sudo nmcli con up enp0s8`

> Pour les hipsters, y'a moyen de ne plus passer du tout par les fichiers dans `/etc/sysconfig` et tout gérer directement avec NetworkManager, cf [la doc officielle](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html-single/configuring_and_managing_networking/index#Selecting-Network-Configuration-methods_overview-of-Network-configuration-methods). 

---

* 🌞 modifier la configuration de la carte réseau privée
  * modifier la configuration de la carte réseau privée pour avoir une nouvelle IP statique définie par vos soins

* ajouter une nouvelle carte réseau dans un DEUXIEME réseau privé UNIQUEMENT privé
  * il faudra par exemple créer un nouveau host-only dans VirtualBox
  * 🌞 dans la VM définir une IP statique pour cette nouvelle carte

* vérifier vos changements
  * afficher les nouvelles cartes/IP
  * vérifier les nouvelles tables ARP/de routage

* 🐙 mettre en place un NIC *teaming* (ou *bonding*)
  * le *teaming* ou *bonding* consiste à agréger deux cartes réseau pour augmenter les performances/la bande passante
  * je vous laisse free sur la configuration (active/passive, loadbalancing, round-robin, autres)
  * prouver que le NIC *teaming* est en place

---

### 2. Serveur SSH

* 🌞 modifier la configuration du système pour que le serveur SSH tourne sur le port 2222
  * adapter la configuration du firewall (fermer l'ancien port, ouvrir le nouveau)

* pour l'étape suivante, il faudra un hôte qui ne s'est jamais connecté à la VM afin d'observer les échanges ARP (vous pouvez aussi juste vider la table ARP du client). Je vous conseille de faire une deuxième VM dans le même réseau, mais vous pouvez utiliser votre PC hôte.

* 🌞 analyser les trames de connexion au serveur SSH
  * intercepter avec Wireshark et/ou `tcpdump` le trafic entre le client SSH et le serveur SSH
  * détailler l'établissement de la connexion
    * doivent figurer au moins : échanges ARP, 3-way handshake TCP
    * 🐙 configurer une connexion par échange de clés, analyser les échanges réseau réalisés par le protocole SSH au moment de la connexion
  * une fois la connexion établie, choisir une trame du trafic SSH et détailler son contenu

# III. Routage simple

Dans cette partie, vous allez remettre en place un routage statique simple. Vous êtes libres du choix de la techno (CentOS8, Cisco, autres. Vous pouvez utiliser GNS3). 

Vous devez reproduire la mini-archi suivante : 
```
                   +-------+
                   |Outside|
                   | world |
                   +---+---+
                       |
                       |
+-------+         +----+---+         +-------+
|       |   net1  |        |   net2  |       |
|  VM1  +---------+ Router +---------+  VM2  |
|       |         |        |         |       |
+-------+         +--------+         +-------+
```

* **Description**
  * Le routeur a trois interfaces, dont une qui permet de joindre l'extérieur (internet)
  * La `VM1` a une interface dans le réseau `net1`
  * La `VM2` a une interface dans le réseau `net2`
  * Les deux VMs peuvent joindre Internet en passant par le `Router`

* 🌞 **To Do** 
  * Tableau récapitulatif des IPs
  * Configuration (bref) de VM1 et VM2
  * Configuration routeur
  * Preuve que VM1 passe par le routeur pour joindre internet
  * Une (ou deux ? ;) ) capture(s) réseau ainsi que des explications qui mettent en évidence le routage effectué par le routeur

# IV. Autres applications et métrologie

Dans cette partie, on va jouer un peu avec de nouvelles commandes qui peuvent être utiles pour diagnostiquer un peu ce qu'il se passe niveau réseau.

---

## 1. Commandes

* jouer avec `iftop`
  * expliquer son utilisation et imaginer un cas où `iftop` peut être utile

---

## 2. Cockpit

* 🌞 mettre en place cockpit sur la VM1
  * c'est quoi ? C'est un service web. Pour quoi faire ? Vous allez vite comprendre en le voyant.
  * `sudo dnf install -y cockpit`
  * `sudo systemctl start cockpit`
  * trouver (à l'aide d'une commande shell) sur quel port (TCP ou UDP) écoute Cockpit 
  * vérifier que le port est ouvert dans le firewall
* 🌞 explorer Cockpit, plus spécifiquement ce qui est en rapport avec le réseau

---

## 3. Netdata

Netdata est un outil utilisé pour récolter des métriques et envoyer des alertes. Il peut aussi être utilisé afin de visionner ces métriques, à court terme. Nous allons ici l'utiliser pour observer les métriques réseau et mettre en place un service web supplémentaire.

* 🌞 mettre en place Netdata sur la VM1 et la VM2
  * se référer à la documentation officielle
  * repérer et ouvrir le port dédié à l'interface web de Netdata
* 🌞 explorer les métriques liées au réseau que récolte Netdata
