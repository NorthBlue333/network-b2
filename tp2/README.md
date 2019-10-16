# TP2 : Network low-level, Switching

# Sommaire

- [TP2 : Network low-level, Switching](#tp2--network-low-level-switching)
- [Sommaire](#sommaire)
- [I. Simplest setup](#i-simplest-setup)
      - [Topologie](#topologie)
      - [Plan d'adressage](#plan-dadressage)
- [II. More switches](#ii-more-switches)
      - [Topologie 2](#topologie-2)
      - [Plan d'adressage](#plan-dadressage-1)
      - [Mise en évidence du Spanning Tree Protocol](#mise-en-%c3%a9vidence-du-spanning-tree-protocol)
      - [Route ports](#route-ports)
      - [Reconfigurer STP](#reconfigurer-stp)
      - [🐙 STP & Perfs](#%f0%9f%90%99-stp--perfs)
- [III. Isolation](#iii-isolation)
  - [1. Simple](#1-simple)
      - [Topologie](#topologie-1)
      - [Plan d'adressage](#plan-dadressage-2)
      - [ToDo](#todo)
  - [2. Avec trunk](#2-avec-trunk)
      - [Topologie](#topologie-2)
      - [Plan d'adressage](#plan-dadressage-3)
      - [ToDo](#todo-1)
- [IV. Need perfs](#iv-need-perfs)
      - [Topologie](#topologie-3)
      - [Plan d'adressage](#plan-dadressage-4)
      - [ToDo](#todo-2)

**Dans ce TP, vous pouvez considérez que :**
* les `PC` sont [**des VPCS de GNS3**](/memo/setup-gns3.md#utilisation-dun-vpcs) (sauf indication contraire)
* les `SW` sont des Switches Cisco, virtualisé avec l'IOU L2

# I. Simplest setup

#### Topologie

```
+-----+        +-------+        +-----+
| PC1 +--------+  IOU1 +--------+ PC2 |
+-----+        +-------+        +-----+
```
![screen GNS3](screens/infra1.png)

#### Plan d'adressage

Machine | `net1`
--- | ---
`PC1` | `10.2.1.1/24`
`PC2` | `10.2.1.2/24`

Depuis PC1 : `ping 10.2.1.2`. Le protocole utilisé est ICMP, mais le protocole ARP est utilisé pour trouver la MAC correspondant à l'adresse IP 10.2.1.2.

Dans le fichier [pc1-iou](captures/pc1-iou.pcapng), les lignes 19-20-21 correspondent aux échanges ARP (broadcast pour "Qui est 10.2.1.2", réponse avec la mac de 10.2.1.2). On voit bien ensuite les request/reply du ping. (de même pour le fichier [pc2-iou](captures/pc2-iou)).

Sur PC1 :
```
show arp
00:50:79:66:68:01  10.2.1.2 expires in 117 seconds
```

Sur PC2 :
```
show arp
00:50:79:66:68:00  10.2.1.1 expires in 65 seconds
```

Le switch n'a pas besoin d'IP car il se comporte comme une "multiprise".

# II. More switches

#### Topologie 2

```
                        +-----+
                        | PC4 |
                        +--+--+
                           |
                           |
                       +---+---+
                   +---+  IOU3 +----+
                   |   +-------+    |
                   |                |
                   |                |
+-----+        +---+---+        +---+---+        +-----+
| PC3 +--------+  IOU2 +--------+ IOU4  +--------+ PC5 |
+-----+        +-------+        +-------+        +-----+
```
![screen GNS3](screens/infra2.png)

* **IOU2** : c'est le *Route Bridge*, donc tous ses ports sont forwarded
* **IOU3** : son *Root port* est `eth1/0`
* **IOU4** : son *Root port* est `eth0/0`, son port bloqué est `eth2/0`

[Voir la partie correspondante](#Route-ports)

#### Plan d'adressage

Machine | `net1`
--- | ---
`PC3` | `10.2.2.1/24`
`PC4` | `10.2.2.2/24`
`PC5` | `10.2.2.3/24`

Sur PC3 :
```
show
NAME   IP/MASK              GATEWAY           MAC                LPORT  RHOST:PORT
PC3    10.2.2.1/24          255.255.255.0     00:50:79:66:68:04  10016  127.0.0.1:10017
       fe80::250:79ff:fe66:6804/64
PC3> ping 10.2.2.2
84 bytes from 10.2.2.2 icmp_seq=1 ttl=64 time=0.307 ms
84 bytes from 10.2.2.2 icmp_seq=2 ttl=64 time=0.866 ms
^C
PC3> ping 10.2.2.3
84 bytes from 10.2.2.3 icmp_seq=1 ttl=64 time=0.317 ms
84 bytes from 10.2.2.3 icmp_seq=2 ttl=64 time=0.980 ms
^C
```
Sur SW2 :
```
show mac address-table
          Mac Address Table
-------------------------------------------

Vlan    Mac Address       Type        Ports
----    -----------       --------    -----
   1    0050.7966.6804    DYNAMIC     Et1/0 #MAC de PC3 00:50:79:66:68:04
   1    aabb.cc00.0201    DYNAMIC     Et1/0 #MAC de IOU2 sur le port et1/0
   1    aabb.cc00.0400    DYNAMIC     Et1/0 #MAC du IOU4 sur le port et0/0
   1    aabb.cc00.0402    DYNAMIC     Et2/0 #MAC du IOU4 sur le port et2/0
Total Mac Addresses for this criterion: 4
```

Sur les switches : `show interfaces <interface>` pour avoir toutes les infos (dont la mac). La table comprend une adresse MAC par lien (IOU2<->IOU3, IOU4<->IOU3 et IOU2<->IOU4), et la MAC du PC3 qui a effectué un ping. Les VLAN sont par défaut 1.

Pour les trames CDP : [capture CDP iou2-3](captures/iou2-iou3.pcapng), [capture CDP iou2-4](captures/iou2-iou4.pcapng), [capture CDP iou3-4](captures/iou3-iou4.pcapng). Ces trames utilisent Cisco Discovery Protocol, qui sert à la découverte réseau de niveau 2. Il permet de trouver des périphériques voisins connectés.

#### Mise en évidence du Spanning Tree Protocol

Si on considère les trois liens qui unissent les switches :
* `IOU2` <> `IOU3`
* `IOU3` <> `IOU4`
* `IOU2` <> `IOU4`  

**L'un de ces liens a forcément été désactivé.**

Sur IOU2 : (on peut voir qu'il est le *Root bridge* du VLAN1)
```
show interfaces
Ethernet0/0 is up, line protocol is up (connected)
  Hardware is AmdP2, address is aabb.cc00.0200 (bia aabb.cc00.0200)
  MTU 1500 bytes, BW 10000 Kbit/sec, DLY 1000 usec,
     reliability 255/255, txload 1/255, rxload 1/255
  Encapsulation ARPA, loopback not set
  Keepalive set (10 sec)
  Auto-duplex, Auto-speed, media type is unknown
  input flow-control is off, output flow-control is unsupported
  ARP type: ARPA, ARP Timeout 04:00:00
  Last input 00:00:04, output 00:00:01, output hang never
  Last clearing of "show interface" counters never
  Input queue: 0/2000/0/0 (size/max/drops/flushes); Total output drops: 0
  Queueing strategy: fifo
  Output queue: 0/0 (size/max)
  5 minute input rate 0 bits/sec, 0 packets/sec
  5 minute output rate 0 bits/sec, 0 packets/sec
     235 packets input, 42468 bytes, 0 no buffer
     Received 235 broadcasts (0 multicasts)
     0 runts, 0 giants, 0 throttles
     0 input errors, 0 CRC, 0 frame, 0 overrun, 0 ignored
     0 input packets with dribble condition detected
     2411 packets output, 185796 bytes, 0 underruns
     0 output errors, 0 collisions, 0 interface resets

IOU2#show sp
IOU2#show spanning-tree

VLAN0001
  Spanning tree enabled protocol rstp
  Root ID    Priority    32769
             Address     aabb.cc00.0200
             This bridge is the root
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    32769  (priority 32768 sys-id-ext 1)
             Address     aabb.cc00.0200
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  300 sec

Interface           Role Sts Cost      Prio.Nbr Type
------------------- ---- --- --------- -------- --------------------------------
Et0/0               Desg FWD 100       128.1    Shr
Et0/1               Desg FWD 100       128.2    Shr
Et0/2               Desg FWD 100       128.3    Shr
Et0/3               Desg FWD 100       128.4    Shr
Et1/0               Desg FWD 100       128.5    Shr
Et1/1               Desg FWD 100       128.6    Shr
Et1/2               Desg FWD 100       128.7    Shr
Et1/3               Desg FWD 100       128.8    Shr
Et2/0               Desg FWD 100       128.9    Shr
Et2/1               Desg FWD 100       128.10   Shr
Et2/2               Desg FWD 100       128.11   Shr
Et2/3               Desg FWD 100       128.12   Shr
Et3/0               Desg FWD 100       128.13   Shr
Et3/1               Desg FWD 100       128.14   Shr
Et3/2               Desg FWD 100       128.15   Shr
Et3/3               Desg FWD 100       128.16   Shr
```
Sur IOU3 :
```
show interfaces
Ethernet0/0 is up, line protocol is up (connected)
  Hardware is AmdP2, address is aabb.cc00.0300 (bia aabb.cc00.0300)
  MTU 1500 bytes, BW 10000 Kbit/sec, DLY 1000 usec,
     reliability 255/255, txload 1/255, rxload 1/255
  Encapsulation ARPA, loopback not set
  Keepalive set (10 sec)
  Auto-duplex, Auto-speed, media type is unknown
  input flow-control is off, output flow-control is unsupported
  ARP type: ARPA, ARP Timeout 04:00:00
  Last input never, output 00:00:01, output hang never
  Last clearing of "show interface" counters never
  Input queue: 0/2000/0/0 (size/max/drops/flushes); Total output drops: 0
  Queueing strategy: fifo
  Output queue: 0/0 (size/max)
  5 minute input rate 0 bits/sec, 0 packets/sec
  5 minute output rate 0 bits/sec, 0 packets/sec
     0 packets input, 0 bytes, 0 no buffer
     Received 0 broadcasts (0 multicasts)
     0 runts, 0 giants, 0 throttles
     0 input errors, 0 CRC, 0 frame, 0 overrun, 0 ignored
     0 input packets with dribble condition detected
     2564 packets output, 199122 bytes, 0 underruns
     0 output errors, 0 collisions, 0 interface resets

IOU3#show spanning-tree

VLAN0001
  Spanning tree enabled protocol rstp
  Root ID    Priority    32769
             Address     aabb.cc00.0200
             Cost        100
             Port        5 (Ethernet1/0)
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    32769  (priority 32768 sys-id-ext 1)
             Address     aabb.cc00.0300
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  300 sec

Interface           Role Sts Cost      Prio.Nbr Type
------------------- ---- --- --------- -------- --------------------------------
Et0/0               Desg FWD 100       128.1    Shr
Et0/1               Desg FWD 100       128.2    Shr
Et0/2               Desg FWD 100       128.3    Shr
Et0/3               Desg FWD 100       128.4    Shr
Et1/0               Root FWD 100       128.5    Shr
Et1/1               Desg FWD 100       128.6    Shr
Et1/2               Desg FWD 100       128.7    Shr
Et1/3               Desg FWD 100       128.8    Shr
Et2/0               Desg FWD 100       128.9    Shr
Et2/1               Desg FWD 100       128.10   Shr
Et2/2               Desg FWD 100       128.11   Shr
Et2/3               Desg FWD 100       128.12   Shr
Et3/0               Desg FWD 100       128.13   Shr
Et3/1               Desg FWD 100       128.14   Shr
Et3/2               Desg FWD 100       128.15   Shr
Et3/3               Desg FWD 100       128.16   Shr
```
Sur IOU4 :
```
show interfaces
Ethernet0/0 is up, line protocol is up (connected)
  Hardware is AmdP2, address is aabb.cc00.0400 (bia aabb.cc00.0400)
  MTU 1500 bytes, BW 10000 Kbit/sec, DLY 1000 usec,
     reliability 255/255, txload 1/255, rxload 1/255
  Encapsulation ARPA, loopback not set
  Keepalive set (10 sec)
  Auto-duplex, Auto-speed, media type is unknown
  input flow-control is off, output flow-control is unsupported
  ARP type: ARPA, ARP Timeout 04:00:00
  Last input 00:00:00, output 00:00:06, output hang never
  Last clearing of "show interface" counters never
  Input queue: 0/2000/0/0 (size/max/drops/flushes); Total output drops: 0
  Queueing strategy: fifo
  Output queue: 0/0 (size/max)
  5 minute input rate 0 bits/sec, 0 packets/sec
  5 minute output rate 0 bits/sec, 0 packets/sec
     2085 packets input, 140911 bytes, 0 no buffer
     Received 2085 broadcasts (0 multicasts)
     0 runts, 0 giants, 0 throttles
     0 input errors, 0 CRC, 0 frame, 0 overrun, 0 ignored
     0 input packets with dribble condition detected
     364 packets output, 56478 bytes, 0 underruns
     0 output errors, 0 collisions, 0 interface resets

IOU4#show spanning-tree

VLAN0001
  Spanning tree enabled protocol rstp
  Root ID    Priority    32769
             Address     aabb.cc00.0200
             Cost        100
             Port        1 (Ethernet0/0)
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    32769  (priority 32768 sys-id-ext 1)
             Address     aabb.cc00.0400
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  300 sec

Interface           Role Sts Cost      Prio.Nbr Type
------------------- ---- --- --------- -------- --------------------------------
Et0/0               Root FWD 100       128.1    Shr
Et0/1               Desg FWD 100       128.2    Shr
Et0/2               Desg FWD 100       128.3    Shr
Et0/3               Desg FWD 100       128.4    Shr
Et1/0               Desg FWD 100       128.5    Shr
Et1/1               Desg FWD 100       128.6    Shr
Et1/2               Desg FWD 100       128.7    Shr
Et1/3               Desg FWD 100       128.8    Shr
Et2/0               Altn BLK 100       128.9    Shr
Et2/1               Desg FWD 100       128.10   Shr
Et2/2               Desg FWD 100       128.11   Shr
Et2/3               Desg FWD 100       128.12   Shr
Et3/0               Desg FWD 100       128.13   Shr
Et3/1               Desg FWD 100       128.14   Shr
Et3/2               Desg FWD 100       128.15   Shr
Et3/3               Desg FWD 100       128.16   Shr
```
#### Route ports
[Voir le schéma avec les ports](#Topologie-2)

Ping entre PC3 et PC4 : [voir la capture du lien entre IOU2 et IOU3](captures/iou2-iou3-stp.pcapng). On voit bien le ping passer par ce lien. Le lien désactivé est celui entre IOU3 et IOU4 car aucun des deux n'est le route bridge et il n'est donc pas nécessaire.

Schéma d'une requête ARP lorsque PC3 ping PC5 et captures : [pc3-iou2](captures/pc3-iou2-arp.pcapng), [iou2-iou4](captures/iou2-iou4-arp.pcapng), [iou4-pc5](captures/iou4-pc5-arp.pcapng). Les requêtes ARP passent par tous les liens en broadcast, et la réponse se fait sur le lien le plus court. (Les requêtes sont dupliquées ???)

#### Reconfigurer STP

```
# AVANT
IOU2#show spanning-tree

VLAN0001
  Spanning tree enabled protocol rstp
  Root ID    Priority    32769
             Address     aabb.cc00.0200
             This bridge is the root
# APRES
conf t
spanning-tree vlan 1 priority 4096
IOU4#show spanning-tree

VLAN0001
  Spanning tree enabled protocol rstp
  Root ID    Priority    4097
             Address     aabb.cc00.0400
             This bridge is the root

```

Les captures : [iou2-iou3](captures/iou2-iou3-stpchange.pcapng), [iou3-iou4](captures/iou3-iou4-stpchange.pcapng), [iou2-iou4](captures/iou2-iou4-stpchange.pcapng)

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
