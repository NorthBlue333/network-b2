# TP2 : Network low-level, Switching

# Sommaire

- [TP2 : Network low-level, Switching](#tp2--network-low-level-switching)
- [Sommaire](#sommaire)
- [Intro](#intro)
- [0. Etapes préliminaires](#0-etapes-pr%c3%a9liminaires)
- [I. Simplest setup](#i-simplest-setup)
      - [Topologie](#topologie)
      - [Plan d'adressage](#plan-dadressage)
      - [ToDo](#todo)
- [II. More switches](#ii-more-switches)
      - [Topologie](#topologie-1)
      - [Plan d'adressage](#plan-dadressage-1)
      - [ToDo](#todo-1)
      - [Mise en évidence du Spanning Tree Protocol](#mise-en-%c3%a9vidence-du-spanning-tree-protocol)
      - [Reconfigurer STP](#reconfigurer-stp)
      - [🐙 STP & Perfs](#%f0%9f%90%99-stp--perfs)
- [III. Isolation](#iii-isolation)
  - [1. Simple](#1-simple)
      - [Topologie](#topologie-2)
      - [Plan d'adressage](#plan-dadressage-2)
      - [ToDo](#todo-2)
  - [2. Avec trunk](#2-avec-trunk)
      - [Topologie](#topologie-3)
      - [Plan d'adressage](#plan-dadressage-3)
      - [ToDo](#todo-3)
- [IV. Need perfs](#iv-need-perfs)
      - [Topologie](#topologie-4)
      - [Plan d'adressage](#plan-dadressage-4)
      - [ToDo](#todo-4)

# Intro

Dans ce TP on va se pencher un peu plus sur les échanges réseau en eux-mêmes, en **analysant les trames réseau avec Wireshark**. 

On va aussi jouer de façon un peu plus avancée avec des **switches**.

On va commencer à rentrer plus dans le détails des différents éléments.  
**Allez à votre rythme, prenez le temps de comprendre.**  
**Posez des questions.**  
**Prenez des notes au fur et à mesure.**  
**Lisez les parties en entier avant de commencer à travailler dessus.**

Pour ce qui est de la mise en place, on va manipuler des switches (IOS Cisco) et aborder les notions/protocoles suivants : 
* ARP
* `ping`
* Spanning-Tree : STP
* Utilisation de VLAN : Trunking
* Agrégation de ports : LACP 

> **Référez-vous [au README des TPs](/tp/README.md) pour des infos sur le déroulement et le rendu des TPs.**

# 0. Etapes préliminaires

* avoir lu [le README des TPs](/tp/README.md)
* [**GNS3** installé et configuré](/memo/setup-gns3.md) (avec la **GNS3VM**, dans la même version)
* **Wireshark** installé
* Lecture du [mémo/setup GNS3](/memo/setup-gns3.md)
* Lecture du [mémo CLI Cisco](/memo/cli-cisco.md) section [Général](/memo/cli-cisco.md#general) et [Switches](/memo/cli-cisco.md#switches)

**Dans ce TP, vous pouvez considérez que :**
* les `PC` sont [**des VPCS de GNS3**](/memo/setup-gns3.md#utilisation-dun-vpcs) (sauf indication contraire)
* les `SW` sont des Switches Cisco, virtualisé avec l'IOU L2

# I. Simplest setup

#### Topologie

```
+-----+        +-------+        +-----+
| PC1 +--------+  SW1  +--------+ PC2 |
+-----+        +-------+        +-----+
```

#### Plan d'adressage

Machine | `net1`
--- | ---
`PC1` | `10.2.1.1/24`
`PC2` | `10.2.1.2/24`

#### ToDo

Depuis PC1 : `ping 10.2.1.2`

* 🌞 mettre en place la topologie ci-dessus
* 🌞 faire communiquer les deux PCs
  * avec un `ping` qui fonctionne
    * déterminer le protocole utilisé par `ping` à l'aide de Wireshark
  * analyser les échanges ARP
    * utiliser Wireshark et mettre en évidence l'échange ARP entre les deux machines (`ARP Request` et `ARP Reply`)
    * corréler avec les tables ARP des différentes machines
* 🌞 récapituler toutes les étapes (dans le compte-rendu, à l'écrit) quand `PC1` exécute `ping PC2` pour la première fois
  * échanges ARP
  * échange `ping`
* 🌞 expliquer...
  * pourquoi le switch n'a pas besoin d'IP
  * pourquoi les machines ont besoin d'une IP pour pouvoir se `ping`

# II. More switches

#### Topologie

```
                        +-----+
                        | PC2 |
                        +--+--+
                           |
                           |
                       +---+---+
                   +---+  SW2  +----+
                   |   +-------+    |
                   |                |
                   |                |
+-----+        +---+---+        +---+---+        +-----+
| PC1 +--------+  SW1  +--------+  SW3  +--------+ PC3 |
+-----+        +-------+        +-------+        +-----+
```

#### Plan d'adressage

Machine | `net1`
--- | ---
`PC1` | `10.2.2.1/24`
`PC2` | `10.2.2.2/24`
`PC3` | `10.2.2.3/24`

#### ToDo

* 🌞 mettre en place la topologie ci-dessus
* 🌞 faire communiquer les trois PCs
  * avec des `ping` qui fonctionnent
* 🌞 analyser la table MAC d'un switch
  * `show mac address-table`
  * comprendre/expliquer chaque ligne
* 🐙 en lançant Wireshark sur les liens des switches, il y a des trames CDP qui circulent. Quoi qu'est-ce ?

#### Mise en évidence du Spanning Tree Protocol

STP a été ici automatiquement configuré par les switches eux-mêmes pour éviter une boucle réseau. 

Dans une configuration pareille, les switches ont élu un chemin de préférence.  
Si on considère les trois liens qui unissent les switches :
* `SW1` <> `SW2`
* `SW2` <> `SW3`
* `SW1` <> `SW3`  

**L'un de ces liens a forcément été désactivé.**

On va regarder comment STP a été configuré.

* 🌞 déterminer les informations STP
  * à l'aide des [commandes dédiées au protocole](/memo/cli-cisco.md#stp)
* 🌞 faire un schéma en représentant les informations STP
  * rôle des switches (qui est le root bridge)
  * rôle de chacun des ports
* 🌞 confirmer les informations STP
  * effectuer un `ping` d'une machine à une autre
  * vérifier que les trames passent bien par le chemin attendu (Wireshark)
* 🌞 ainsi, déterminer quel lien a été désactivé par STP
* 🌞 faire un schéma qui explique le trajet d'une requête ARP lorsque PC1 ping PC3, et de sa réponse
  * représenter **TOUTES** les trames ARP (n'oubliez pas les broadcasts)

#### Reconfigurer STP

* 🌞 changer la priorité d'un switch qui n'est pas le *root bridge*
* 🌞 vérifier les changements
  * avec des commandes sur les switches
  * 🐙 capturer les échanges qui suivent une reconfiguration STP avec Wireshark

#### 🐙 STP & Perfs

Si vous avez lancé Wireshark sur un lien entre un PC et un Switch, vous avez vu qu'il y a toujours des trames STP qui circulent...
* un peu con non ? C'est un PC, il enverra jamais de trames STP
* aussi avec STP, quand on branche un PC, le lien mettra plusieurs secondes avant de passer en *forwarding* et ainsi transmettre de la donnée
* l'idéal ça serait de désactiver l'envoi de trames STP sur l'interface du switch (ça évite de cramer de la bande passante et du calcul CPU pour rien, générer du trafic inutile, etc.)
* sauuuuf que si un p'tit malin branche des switches là-dessus, il pourrait tout péter en créant une boucle
* deux fonctionnalités à mettre en place : 
  * `portfast` : marque un port comme *"edge"* dans la topologie STP. Un port *edge* est considéré comme une extrémité de la topologie (= un client branché dessus, port *access*). *Port**fast*** parce que ça va permettre au port de s'allumer plus rapidement (sans passer par les états *listening* et *learning* pendant 15 secondes chacun par défaut) et d'être disponible instantanément
    * on peut voir l'état d'un port (forward, listening, learning, blocking avec `show spanning-tree vlan 1`)
  * `bpduguard` : permet de shutdown le port s'il reçoit des *BPDU* (pour rappel : un *BPDU* c'est un message STP)  
  
Idem pour les trames CDP !

🐙 ToDo :
  * [activer ces fonctionnalités (*portfast* et *bpduguard*) et activer le filtre BPDU](/memo/cli-cisco.md#stp) sur les interfaces où c'est nécessaire (marqué comme *edge* dans la topologie STP)
  * aussi [désactiver l'envoi de trames CDP](/memo/cli-cisco.md#cdp) sur ces ports
    * prouver avec Wireshark que le switch n'envoie plus de BPDU ni de trames CDP
    * faites une capture avant et une capture après les manips pour le prouver :)

# III. Isolation

## 1. Simple
 
#### Topologie
```
+-----+        +-------+        +-----+
| PC1 +--------+  SW1  +--------+ PC3 |
+-----+      10+-------+20      +-----+
                 20|
                   |
                +--+--+
                | PC2 |
                +-----+
```

#### Plan d'adressage

Machine | IP `net1` | VLAN
--- | --- | --- 
`PC1` | `10.2.3.1/24` | 10
`PC2` | `10.2.3.2/24` | 20
`PC3` | `10.2.3.3/24` | 20

#### ToDo

* 🌞 mettre en place la topologie ci-dessus
  * voir [les commandes dédiées à la manipulation de VLANs](/memo/cli-cisco.md#vlan)
* 🌞 faire communiquer les PCs deux à deux
  * vérifier que `PC2` ne peut joindre que `PC3`
  * vérifier que `PC1` ne peut joindre personne alors qu'il est dans le même réseau (sad)

## 2. Avec trunk

#### Topologie

```
+-----+        +-------+        +-------+        +-----+
| PC1 +--------+  SW1  +--------+  SW2  +--------+ PC4 |
+-----+      10+-------+        +-------+20      +-----+
                 20|              10|
                   |                |
                +--+--+          +--+--+
                | PC2 |          | PC3 |
                +-----+          +-----+
```

#### Plan d'adressage

Machine | IP `net1` | IP `net2` | VLAN
--- | --- | --- | ---
`PC1` | `10.2.10.1/24` | X | 10
`PC2` | X | `10.2.20.1/24` | 20
`PC3` | `10.2.10.2/24` | X | 10
`PC4` | X | `10.2.20.2/24` | 20

#### ToDo

* 🌞 mettre en place la topologie ci-dessus
* 🌞 faire communiquer les PCs deux à deux
  * vérifier que `PC1` ne peut joindre que `PC3`
  * vérifier que `PC4` ne peut joindre que `PC2`
* 🌞 mettre en évidence l'utilisation des VLANs avec Wireshark

# IV. Need perfs

#### Topologie

Pareil qu'en [III.2.](#2-avec-trunk) à part le lien entre SW1 et SW2 qui est doublé.

```
+-----+        +-------+--------+-------+        +-----+
| PC1 +--------+  SW1  |        |  SW2  +--------+ PC4 |
+-----+      10+-------+--------+-------+20      +-----+
                 20|              10|
                   |                |
                +--+--+          +--+--+
                | PC2 |          | PC3 |
                +-----+          +-----+

```
#### Plan d'adressage

Pareil qu'en [III.2.](#2-avec-trunk).

Machine | IP `net1` | IP `net2` | VLAN
--- | --- | --- | ---
`PC1` | `10.2.10.1/24` | X | 10
`PC2` | X | `10.2.20.1/24` | 20
`PC3` | `10.2.10.2/24` | X | 10
`PC4` | X | `10.2.20.2/24` | 20

#### ToDo

* 🌞 mettre en place la topologie ci-dessus
  * configurer LACP entre `SW1` et `SW2`
  * utiliser Wireshark pour mettre en évidence l'utilisation de trames LACP
  * **vérifier avec un `show ip interface po1` que la bande passante a bien été doublée**

> Pas de failover possible sur les IOUs malheureusement :( (voir [ce doc](https://www.cisco.com/c/en/us/td/docs/switches/blades/3020/software/release/12-2_58_se/configuration/guide/3020_scg/swethchl.pdf), dernière section. Pas de link state dans les IOUs)
