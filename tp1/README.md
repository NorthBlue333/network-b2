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

Il y a plusieurs méthodes pour savoir si l'IP est attribuée par un DHCP, comme par exemple regarder le BOOTPROTO des fichiers correspondants aux interfaces réseaux dans `/etc/sysconfig/network-scripts/`, qui sera `dhcp` dans ce cas, ou encore regarder la liste des baux dans `/var/lib/NetworkManager`. Les IPs sont attribuées dynamiquement (mot clé `dynamic` dans le résultat de la commande `ip a`).

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

**[Routage] Commande :** `ip route`

**[Routage] Résultat :**
```
default via 10.0.2.2 dev enp0s3 proto dhcp metric 100 #cette route est celle par défaut de la carte enp0s3, elle est utilisée pour une connexion externe, la passerelle de cette route est à l'IP 10.0.2.2 et cette IP est portée par le routeur Ynov
10.0.2.0/24 dev enp0s3 proto kernel scope link src 10.0.2.15 metric 100 #cette route est celle de la carte enp0s3 pour des IPs dans le réseau 10.0.2.0/24, elle est utilisée pour une connexion locale, la passerelle de cette route est à l'IP 10.0.2.2
192.168.56.0/24 dev enp0s8 proto kernel scope link src 192.168.56.102 metric 101 #cette route est celle de la carte enp0s8 pour des IPs dans le réseau 192.168.56.0/24, elle est utilisée pour une connexion locale, la passerelle de cette route est à l'IP 192.168.56.1 et elle est portée par la carte VHOST de mon pc
```
**[ARP] Commande :** `ip neigh`

**[ARP] Résultat :**
```
192.168.56.1 dev enp0s8 lladdr 0a:00:27:00:00:2e DELAY #c'est l'IP de la carte VHOST de mon pc, connue car je suis connectée à la VM en SSH sur cette IP
10.0.2.2 dev enp0s3 lladdr 52:54:00:12:35:02 REACHABLE #c'est l'IP du routeur Ynov, qui permet l'accès à internet, donc connue
```

**Commande :** `sudo netstat -tulnp`

**Résultat :**
```
sudo netstat -tulpn
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      841/sshd
tcp6       0      0 :::22                   :::*                    LISTEN      841/sshd
udp        0      0 192.168.56.102:68       0.0.0.0:*                           825/NetworkManager
udp        0      0 10.0.2.15:68            0.0.0.0:*                           825/NetworkManager
udp        0      0 127.0.0.1:323           0.0.0.0:*                           771/chronyd
udp6       0      0 ::1:323                 :::*                                771/chronyd
```

**[Liste DNS] Commande :** `cat /etc/resolv.conf`

**[Liste DNS] Résultat :**
```
# Generated by NetworkManager
search auvence.co
nameserver 10.33.10.20
nameserver 10.33.10.2
nameserver 8.8.8.8
# NOTE: the libc resolver may not support more than 3 nameservers.
# The nameservers listed below may not be recognized.
nameserver 8.8.4.4
```
**[Reddit] Commande :** `dig www.reddit.com +short`

**[Reddit] Résultat :**
```
reddit.map.fastly.net.
151.101.193.140       
151.101.65.140        
151.101.129.140       
151.101.1.140                                            
```

**Commandes :** `systemctl status firewalld` ou `firewall-cmd --state` (running)

  * quelles interfaces sont filtrées ?
  * quel port TCP/UDP sont autorisés/filtrés ?

**Commande :** `firewall-cmd --list-all`

**Résultat :**
```
public (active)                      
  target: default                    
  icmp-block-inversion: no           
  interfaces: enp0s3 enp0s8 #interfaces
  sources:                           
  services: cockpit dhcpv6-client ssh
  ports: #pas de ports
  protocols:                         
  masquerade: no                     
  forward-ports:                     
  source-ports:                      
  icmp-blocks:                       
  rich rules:
```

  * 🐙 sous CentOS8, ce n'est plus `iptables` qui est utilisé pour manipuler le filtrage réseau mais `nftables`. Jouez un peu avec `nft` et affichez les "vraies" règles firewall (`firewalld`, manipulé avec `firewall-cmd` n'est qu'une surcouche à `nft`)

**Commande :** `sudo nft list table filter` ou avec n'importe quelle table dans `sudo nft list tables`
**Résultat :**
```
table inet firewalld {
        chain raw_PREROUTING {
                type filter hook prerouting priority -290; policy accept;
                icmpv6 type { nd-router-advert, nd-neighbor-solicit } accept
                meta nfproto ipv6 fib saddr . iif oif missing drop
                jump raw_PREROUTING_ZONES_SOURCE
                jump raw_PREROUTING_ZONES
        }

        chain raw_PREROUTING_ZONES_SOURCE {
        }

        chain raw_PREROUTING_ZONES {
                iifname "enp0s9" goto raw_PRE_public
                iifname "bond0" goto raw_PRE_public
                iifname "enp0s8" goto raw_PRE_public
                iifname "enp0s3" goto raw_PRE_public
                goto raw_PRE_public
        }

        chain mangle_PREROUTING {
                type filter hook prerouting priority -140; policy accept;
                jump mangle_PREROUTING_ZONES_SOURCE
                jump mangle_PREROUTING_ZONES
        }

        chain mangle_PREROUTING_ZONES_SOURCE {
        }

        chain mangle_PREROUTING_ZONES {
                iifname "enp0s9" goto mangle_PRE_public
                iifname "bond0" goto mangle_PRE_public
                iifname "enp0s8" goto mangle_PRE_public
                iifname "enp0s3" goto mangle_PRE_public
                goto mangle_PRE_public
        }

        chain filter_INPUT {
                type filter hook input priority 10; policy accept;
                ct state established,related accept
                iifname "lo" accept
                jump filter_INPUT_ZONES_SOURCE
                jump filter_INPUT_ZONES
                ct state invalid drop
                reject with icmpx type admin-prohibited
        }

        chain filter_FORWARD {
                type filter hook forward priority 10; policy accept;
                ct state established,related accept
                iifname "lo" accept
                ip6 daddr { ::/96, ::ffff:0.0.0.0/96, 2002::/24, 2002:a00::/24, 2002:7f00::/24, 2002:a9fe::/32, 2002:ac10::/28, 2002:c0a8::/32, 2002:e000::/19 } reject with icmpv6 type addr-unreachable
                jump filter_FORWARD_IN_ZONES_SOURCE
                jump filter_FORWARD_IN_ZONES
                jump filter_FORWARD_OUT_ZONES_SOURCE
                jump filter_FORWARD_OUT_ZONES
                ct state invalid drop
                reject with icmpx type admin-prohibited
        }

        chain filter_OUTPUT {
                type filter hook output priority 10; policy accept;
                oifname "lo" accept
                ip6 daddr { ::/96, ::ffff:0.0.0.0/96, 2002::/24, 2002:a00::/24, 2002:7f00::/24, 2002:a9fe::/32, 2002:ac10::/28, 2002:c0a8::/32, 2002:e000::/19 } reject with icmpv6 type addr-unreachable
        }

        chain filter_INPUT_ZONES_SOURCE {
        }

        [...] #j'ai coupé une partie c'était trop long

        chain mangle_PRE_public_pre {
        }

        chain mangle_PRE_public_log {
        }

        chain mangle_PRE_public_deny {
        }

        chain mangle_PRE_public_allow {
        }

        chain mangle_PRE_public_post {
        }

        chain filter_FWDO_public {
                jump filter_FWDO_public_pre
                jump filter_FWDO_public_log
                jump filter_FWDO_public_deny
                jump filter_FWDO_public_allow
                jump filter_FWDO_public_post
        }

        chain filter_FWDO_public_pre {
        }

        chain filter_FWDO_public_log {
        }

        chain filter_FWDO_public_deny {
        }

        chain filter_FWDO_public_allow {
        }

        chain filter_FWDO_public_post {
        }
}
```

## II. Edit configuration

**Deuxièmement : Modifier la configuration existante**

### 1. Configuration cartes réseau

*(j'ai regardé viteuf pour nmcli mais flemme là)*

```
cat /etc/sysconfig/network-scripts/ifcfg-enp0s8
TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=static
DEFROUTE=yes
IPADDR=192.168.56.103
NETMASK=255.255.255.0
NAME=enp0s8
UUID=8ca712aa-07de-4659-9174-67d4b1e4823d
DEVICE=enp0s8
ONBOOT=yes
nmcli c reload
nmcli con up enp0s8
```

Sur VirtualBox, on définit une nouvelle carte VHOST, ensuite on reboot la VM en ajoutant la carte. On crée le fichier `/etc/sysconfig/network-scripts/ifcfg-enp0s9`
```
TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=static
DEFROUTE=yes
IPADDR=192.168.212.103
NETMASK=255.255.255.0
NAME=enp0s9
DEVICE=enp0s9
ONBOOT=yes
nmcli c reload
nmcli con up enp0s8
```

```
ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:34:f0:bc brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic noprefixroute enp0s3
       valid_lft 85883sec preferred_lft 85883sec
    inet6 fe80::fd60:7592:daf0:6335/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:6f:54:c9 brd ff:ff:ff:ff:ff:ff
    inet 192.168.56.103/24 brd 192.168.56.255 scope global noprefixroute enp0s8
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe6f:54c9/64 scope link
       valid_lft forever preferred_lft forever
4: enp0s9: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000 #nouvelle
    link/ether 08:00:27:03:8d:10 brd ff:ff:ff:ff:ff:ff
    inet 192.168.212.103/24 brd 192.168.212.255 scope global noprefixroute enp0s9
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe03:8d10/64 scope link
       valid_lft forever preferred_lft forever
ip neigh
192.168.56.1 dev enp0s8 lladdr 0a:00:27:00:00:2e REACHABLE
10.0.2.2 dev enp0s3 lladdr 52:54:00:12:35:02 REACHABLE
ip route
default via 10.0.2.2 dev enp0s3 proto dhcp metric 100
10.0.2.0/24 dev enp0s3 proto kernel scope link src 10.0.2.15 metric 100
192.168.56.0/24 dev enp0s8 proto kernel scope link src 192.168.56.103 metric 103
192.168.212.0/24 dev enp0s9 proto kernel scope link src 192.168.212.103 metric 104 #nouvelle
```

NIC *teaming* (ou *bonding*)

Création du fichier `/etc/sysconfig/network-scripts/ifcfg-bond0`.
```
DEVICE=bond0
BONDING_OPTS="miimon=1 updelay=0 downdelay=0 mode=active-backup" TYPE=Bond
BONDING_MASTER=yes
BOOTPROTO=none
IPADDR=192.168.2.12
PREFIX=24
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=bond0
ONBOOT=yes
```
Ajout des lignes suivantes :
```
MASTER=bond0
SLAVE=yes
```
Dans les fichiers ifcfg-enp0s8 et ifcfg-enp0s9. On reload ensuite avec les commandes :
```
nmcli c reload
nmcli con up bond0
nmcli con up enp0s8
nmcli con up enp0s9
```
Pour vérifier le bond :
```
cat /proc/net/bonding/bond0
Ethernet Channel Bonding Driver: v3.7.1 (April 27, 2011)

Bonding Mode: load balancing (round-robin)
MII Status: up
MII Polling Interval (ms): 100
Up Delay (ms): 0
Down Delay (ms): 0

Slave Interface: enp0s9
MII Status: up
Speed: 1000 Mbps
Duplex: full
Link Failure Count: 0
Permanent HW addr: 08:00:27:03:8d:10
Slave queue ID: 0

Slave Interface: enp0s8
MII Status: up
Speed: 1000 Mbps
Duplex: full
Link Failure Count: 0
Permanent HW addr: 08:00:27:6f:54:c9
Slave queue ID: 0
```
Et dans les tables ARP, on voit que les réseaux 192.168.56.0/24 et 192.168.212.0/24 (enp0s8 et enp0s9) ne sont plus là, on voit seulement le réseau de bond0 192.168.2.0/24, de plus enp0s8 et enp0s9 n'ont plus d'IP.
```
ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:34:f0:bc brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic noprefixroute enp0s3
       valid_lft 84966sec preferred_lft 84966sec
    inet6 fe80::fd60:7592:daf0:6335/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
3: enp0s8: <BROADCAST,MULTICAST,SLAVE,UP,LOWER_UP> mtu 1500 qdisc fq_codel master bond0 state UP group default qlen 1000
    link/ether 08:00:27:6f:54:c9 brd ff:ff:ff:ff:ff:ff
4: enp0s9: <BROADCAST,MULTICAST,SLAVE,UP,LOWER_UP> mtu 1500 qdisc fq_codel master bond0 state UP group default qlen 1000
    link/ether 08:00:27:6f:54:c9 brd ff:ff:ff:ff:ff:ff
5: bond0: <BROADCAST,MULTICAST,MASTER,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 08:00:27:6f:54:c9 brd ff:ff:ff:ff:ff:ff
    inet 192.168.2.12/24 brd 192.168.2.255 scope global noprefixroute bond0
       valid_lft forever preferred_lft forever
    inet6 fe80::693e:18bd:d85d:ae13/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
```
Ouais il manque une partie de la démo de "ça marche YAY", sorry

---

### 2. Serveur SSH

```
firewall-cmd --remove-port=22/tcp --permanent
firewall-cmd --add-port=2222/tcp --permanent
firewall-cmd --reload
vim /etc/ssh/sshd_config # on modifie la ligne "# Port 22" et on met Port 2222
semanage port -a -t ssh_port_t -p tcp 2222
systemctl restart sshd
```

Avec une deuxième VM :
```
sudo tcpdump -i enp0s8
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on enp0s8, link-type EN10MB (Ethernet), capture size 262144 bytes
16:37:03.103407 ARP, Request who-has localhost.localdomain tell 192.168.56.104, length 46 #Demande ARP
16:37:03.103421 ARP, Reply localhost.localdomain is-at 08:00:27:6f:54:c9 (oui Unknown), length 28 #Réponse ARP
16:37:03.103612 IP 192.168.56.104.50800 > localhost.localdomain.ssh: Flags [S], seq 3590438451, win 29200, options [mss 1460,sackOK,TS val 2376396728 ecr 0,nop,wscale 7], length 0
16:37:03.103643 IP localhost.localdomain.ssh > 192.168.56.104.50800: Flags [R.], seq 0, ack 3590438452, win 0, length 0
16:37:08.550845 ARP, Request who-has 192.168.56.104 tell localhost.localdomain, length 28 #Demande ARP
16:37:08.551000 ARP, Reply 192.168.56.104 is-at 08:00:27:a3:c2:75 (oui Unknown), length 46 #Réponse ARP
16:37:10.213625 IP 192.168.56.104.52182 > localhost.localdomain.EtherNet/IP-1: Flags [S], seq 3763174050, win 29200, options [mss 1460,sackOK,TS val 2376403838 ecr 0,nop,wscale 7], length 0
16:37:10.213672 IP localhost.localdomain.EtherNet/IP-1 > 192.168.56.104.52182: Flags [S.], seq 636941886, ack 3763174051, win 28960, options [mss 1460,sackOK,TS val 3474527237 ecr 2376403838,nop,wscale 7], length 0
16:37:10.214022 IP 192.168.56.104.52182 > localhost.localdomain.EtherNet/IP-1: Flags [.], ack 1, win 229, options [nop,nop,TS val 2376403838 ecr 3474527237], length 0 #1er handshake
16:37:10.225965 IP 192.168.56.104.52182 > localhost.localdomain.EtherNet/IP-1: Flags [P.], seq 1:22, ack 1, win 229, options [nop,nop,TS val 2376403850 ecr 3474527237], length 21 #2ème handshake
16:37:10.225991 IP localhost.localdomain.EtherNet/IP-1 > 192.168.56.104.52182: Flags [.], ack 22, win 227, options [nop,nop,TS val 3474527250 ecr 2376403850], length 0 #3ème handshake (on voit bien ack 1, puis seq 1:22, puis ack 22)
16:37:10.226876 IP localhost.localdomain.EtherNet/IP-1 > 192.168.56.104.52182: Flags [P.], seq 1:22, ack 22, win 227, options [nop,nop,TS val 3474527250 ecr 2376403850], length 21
16:37:10.227324 IP 192.168.56.104.52182 > localhost.localdomain.EtherNet/IP-1: Flags [.], ack 22, win 229, options [nop,nop,TS val 2376403852 ecr 3474527250], length 0
16:37:10.227603 IP 192.168.56.104.52182 > localhost.localdomain.EtherNet/IP-1: Flags [P.], seq 22:1366, ack 22, win 229, options [nop,nop,TS val 2376403852 ecr 3474527250], length 1344
[...]
16:37:21.124545 IP 192.168.56.104.52182 > localhost.localdomain.EtherNet/IP-1: Flags [F.], seq 2562, ack 3014, win 309, options [nop,nop,TS val 2376414749 ecr 3474538148], length 0
16:37:21.125163 IP localhost.localdomain.EtherNet/IP-1 > 192.168.56.104.52182: Flags [.], ack 2563, win 291, options [nop,nop,TS val 3474538149 ecr 2376414749], length 0
16:37:21.132364 IP localhost.localdomain.EtherNet/IP-1 > 192.168.56.104.52182: Flags [F.], seq 3014, ack 2563, win 291, options [nop,nop,TS val 3474538156 ecr 2376414749], length 0
16:37:21.132618 IP 192.168.56.104.52182 > localhost.localdomain.EtherNet/IP-1: Flags [.], ack 3015, win 309, options [nop,nop,TS val 2376414757 ecr 3474538156], length 0
16:37:35.063166 IP 192.168.56.1.57281 > 239.255.255.250.ssdp: UDP, length 173
16:37:36.064024 IP 192.168.56.1.57281 > 239.255.255.250.ssdp: UDP, length 173
```

Avec un échange de clés : 

```
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on enp0s8, link-type EN10MB (Ethernet), capture size 262144 bytes
17:05:55.095554 IP 192.168.56.104.52194 > localhost.localdomain.EtherNet/IP-1: Flags [S], seq 1854099786, win 29200, options [mss 1460,sackOK,TS val 2378128736 ecr 0,nop,wscale 7], length 0
17:05:55.095605 ARP, Request who-has 192.168.56.104 tell localhost.localdomain, length 28 #Arp
17:05:55.095855 ARP, Reply 192.168.56.104 is-at 08:00:27:a3:c2:75 (oui Unknown), length 46 #Arp
17:05:55.095858 IP localhost.localdomain.EtherNet/IP-1 > 192.168.56.104.52194: Flags [S.], seq 924898072, ack 1854099787, win 28960, options [mss 1460,sackOK,TS val 3476252136 ecr 2378128736,nop,wscale 7], length 0
17:05:55.096109 IP 192.168.56.104.52194 > localhost.localdomain.EtherNet/IP-1: Flags [.], ack 1, win 229, options [nop,nop,TS val 2378128737 ecr 3476252136], length 0
17:05:55.096350 IP 192.168.56.104.52194 > localhost.localdomain.EtherNet/IP-1: Flags [P.], seq 1:22, ack 1, win 229, options [nop,nop,TS val 2378128737 ecr 3476252136], length 21
17:05:55.096361 IP localhost.localdomain.EtherNet/IP-1 > 192.168.56.104.52194: Flags [.], ack 22, win 227, options [nop,nop,TS val 3476252137 ecr 2378128737], length 0
17:05:55.108736 IP localhost.localdomain.EtherNet/IP-1 > 192.168.56.104.52194: Flags [P.], seq 1:22, ack 22, win 227, options [nop,nop,TS val 3476252149 ecr 2378128737], length 21
17:05:55.109086 IP 192.168.56.104.52194 > localhost.localdomain.EtherNet/IP-1: Flags [.], ack 22, win 229, options [nop,nop,TS val 2378128750 ecr 3476252149], length 0
17:05:55.109516 IP 192.168.56.104.52194 > localhost.localdomain.EtherNet/IP-1: Flags [P.], seq 22:1366, ack 22, win 229, options [nop,nop,TS val 2378128750 ecr 3476252149], length 1344
17:05:55.110525 IP localhost.localdomain.EtherNet/IP-1 > 192.168.56.104.52194: Flags [P.], seq 22:1054, ack 1366, win 249, options [nop,nop,TS val 3476252151 ecr 2378128750], length 1032
17:05:55.112876 IP 192.168.56.104.52194 > localhost.localdomain.EtherNet/IP-1: Flags [P.], seq 1366:1414, ack 1054, win 245, options [nop,nop,TS val 2378128754 ecr 3476252151], length 48
17:05:55.118644 IP localhost.localdomain.EtherNet/IP-1 > 192.168.56.104.52194: Flags [P.], seq 1054:1514, ack 1414, win 249, options [nop,nop,TS val 3476252159 ecr 2378128754], length 460
17:06:00.408815 IP localhost.localdomain.EtherNet/IP-1 > 192.168.56.104.52194: Flags [P.], seq 3422:3578, ack 3342, win 312, options [nop,nop,TS val 3476257449 ecr 2378134049], length 156
17:06:00.408918 IP localhost.localdomain.EtherNet/IP-1 > 192.168.56.104.52194: Flags [P.], seq 3578:3650, ack 3342, win 312, options [nop,nop,TS val 3476257449 ecr 2378134049], length 72
17:06:00.409116 IP 192.168.56.104.52194 > localhost.localdomain.EtherNet/IP-1: Flags [.], ack 3578, win 341, options [nop,nop,TS val 2378134050 ecr 3476257449], length 0
17:06:00.409129 IP 192.168.56.104.52194 > localhost.localdomain.EtherNet/IP-1: Flags [.], ack 3650, win 341, options [nop,nop,TS val 2378134050 ecr 3476257449], length 0
17:06:00.409253 IP 192.168.56.104.52194 > localhost.localdomain.EtherNet/IP-1: Flags [P.], seq 3342:3378, ack 3650, win 341, options [nop,nop,TS val 2378134050 ecr 3476257449], length 36
17:06:00.409328 IP 192.168.56.104.52194 > localhost.localdomain.EtherNet/IP-1: Flags [P.], seq 3378:3446, ack 3650, win 341, options [nop,nop,TS val 2378134050 ecr 3476257449], length 68
17:06:00.409398 IP 192.168.56.104.52194 > localhost.localdomain.EtherNet/IP-1: Flags [F.], seq 3446, ack 3650, win 341, options [nop,nop,TS val 2378134050 ecr 3476257449], length 0
17:06:00.409874 IP localhost.localdomain.EtherNet/IP-1 > 192.168.56.104.52194: Flags [.], ack 3447, win 312, options [nop,nop,TS val 3476257450 ecr 2378134050], length 0
17:06:00.417236 IP localhost.localdomain.EtherNet/IP-1 > 192.168.56.104.52194: Flags [F.], seq 3650, ack 3447, win 312, options [nop,nop,TS val 3476257457 ecr 2378134050], length 0
17:06:00.417536 IP 192.168.56.104.52194 > localhost.localdomain.EtherNet/IP-1: Flags [.], ack 3651, win 341, options [nop,nop,TS val 2378134058 ecr 3476257457], length 0
```
Je trouve pas l'échange de la clé différent par rapport à l'authentification avec mot de passe

# III. Routage simple

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
| hostname           | ip sur 192.168.56.0/24 | ip sur 192.168.212.0/24 | NAT |
|--------------------|------------------------|-------------------------|-----|
| vm1.centos8.tp1    |                        | 192.168.212.103         |     |
| vm2.centos8.tp1    | 192.168.56.104         |                         |     |
| router.centos8.tp1 | 192.168.56.102         | 192.168.212.102         | YES |

VM1 :
```
cat /etc/sysconfig/network-scripts/route-enp0s9
192.168.56.0/24 via 192.168.212.102 dev enp0s9
cat /etc/sysconfig/network-scripts/ifcfg-enp0s9
TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=static
DEFROUTE=yes
IPADDR=192.168.212.103
NETMASK=255.255.255.0
NAME=enp0s9
DEVICE=enp0s9
ONBOOT=yes
GATEWAY=192.168.212.102
ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:34:f0:bc brd ff:ff:ff:ff:ff:ff
3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:6f:54:c9 brd ff:ff:ff:ff:ff:ff
4: enp0s9: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:03:8d:10 brd ff:ff:ff:ff:ff:ff
    inet 192.168.212.103/24 brd 192.168.212.255 scope global noprefixroute enp0s9
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe03:8d10/64 scope link
       valid_lft forever preferred_lft forever
ip route
default via 192.168.212.102 dev enp0s9 proto static metric 100
192.168.56.0/24 via 192.168.212.102 dev enp0s9 proto static metric 100
192.168.212.0/24 dev enp0s9 proto kernel scope link src 192.168.212.103 metric 100
tracepath 192.168.56.104 -n
1?: [LOCALHOST]                      pmtu 1500
 1:  192.168.212.102                                       0.375ms
 1:  192.168.212.102                                       0.227ms
 2:  192.168.212.102                                       0.174ms !H
tracepath 8.8.8.8 -n
1?: [LOCALHOST]                      pmtu 1500
 1:  192.168.212.102                                       0.410ms
 1:  192.168.212.102                                       0.434ms
 2:  192.168.212.102                                       0.274ms !H
```
VM2 :
```
cat /etc/sysconfig/network-scripts/route-enp0s8
192.168.212.0/24 via 192.168.56.102 dev enp0s8
cat /etc/sysconfig/network-scripts/ifcfg-enp0s8
TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=static
IPADDR=192.168.56.104
MASK=255.255.255.0
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
NAME=enp0s8
UUID=8ca712aa-07de-4659-9174-67d4b1e4823d
DEVICE=enp0s8
ONBOOT=yes
GATEWAY=192.168.56.102
ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:58:75:fd brd ff:ff:ff:ff:ff:ff
3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:a3:c2:75 brd ff:ff:ff:ff:ff:ff
    inet 192.168.56.104/24 brd 192.168.56.255 scope global noprefixroute enp0s8
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fea3:c275/64 scope link
       valid_lft forever preferred_lft forever
ip route
default via 192.168.56.102 dev enp0s8 proto static metric 100
192.168.56.0/24 dev enp0s8 proto kernel scope link src 192.168.56.104 metric 100
192.168.212.0/24 via 192.168.56.102 dev enp0s8 proto static metric 100
tracepath 192.168.212.103 -n
 1?: [LOCALHOST]                      pmtu 1500
 1:  192.168.56.102                                        0.296ms
 1:  192.168.56.102                                        0.230ms
 2:  192.168.56.102                                        0.438ms !H
 tracepath 8.8.8.8 -n
 1?: [LOCALHOST]                      pmtu 1500
 1:  192.168.56.102                                        0.460ms
 1:  192.168.56.102                                        0.506ms
 2:  192.168.56.102                                        0.194ms !H
```
Captures réseaux : sur route `tcpdump -i enp0s9`, sur VM1 :
```
ping 192.168.56.104
PING 192.168.56.104 (192.168.56.104) 56(84) bytes of data.
64 bytes from 192.168.56.104: icmp_seq=1 ttl=63 time=0.537 ms
64 bytes from 192.168.56.104: icmp_seq=2 ttl=63 time=0.658 ms
64 bytes from 192.168.56.104: icmp_seq=3 ttl=63 time=0.609 ms
64 bytes from 192.168.56.104: icmp_seq=4 ttl=63 time=0.736 ms
```
La capture :
```
tcpdump: verbose output suppressed, use -v or -vv for full protocol decodelistening on enp0s9, link-type EN10MB (Ethernet), capture size 262144 bytes
21:24:49.144411 IP 192.168.212.103 > 192.168.56.104: ICMP echo request, id 1620, seq 1, length 64
21:24:49.144698 IP 192.168.56.104 > 192.168.212.103: ICMP echo reply, id 1620, seq 1, length 64
21:24:50.208224 IP 192.168.212.103 > 192.168.56.104: ICMP echo request, id 1620, seq 2, length 64
21:24:50.208546 IP 192.168.56.104 > 192.168.212.103: ICMP echo reply, id 1620, seq 2, length 64
21:24:51.232481 IP 192.168.212.103 > 192.168.56.104: ICMP echo request, id 1620, seq 3, length 64
21:24:51.232802 IP 192.168.56.104 > 192.168.212.103: ICMP echo reply, id 1620, seq 3, length 64
21:24:52.255664 IP 192.168.212.103 > 192.168.56.104: ICMP echo request, id 1620, seq 4, length 64
21:24:52.256021 IP 192.168.56.104 > 192.168.212.103: ICMP echo reply, id 1620, seq 4, length 64
21:24:54.592696 ARP, Request who-has 192.168.212.103 tell router.centos8.tp1, length 28
21:24:54.593092 ARP, Reply 192.168.212.103 is-at 08:00:27:03:8d:10 (oui Unknown), length 46
```
De même avec un ping 8.8.8.8 :
```
tcpdump: verbose output suppressed, use -v or -vv for full protocol decodelistening on enp0s9, link-type EN10MB (Ethernet), capture size 262144 bytes
21:26:40.040926 IP 192.168.212.103 > dns.google: ICMP echo request, id 1626, seq 1, length 64
21:26:40.058123 IP dns.google > 192.168.212.103: ICMP echo reply, id 1626, seq 1, length 64
21:26:41.042728 IP 192.168.212.103 > dns.google: ICMP echo request, id 1626, seq 2, length 64
21:26:41.066243 IP dns.google > 192.168.212.103: ICMP echo reply, id 1626, seq 2, length 64
21:26:42.044308 IP 192.168.212.103 > dns.google: ICMP echo request, id 1626, seq 3, length 64
21:26:42.061404 IP dns.google > 192.168.212.103: ICMP echo reply, id 1626, seq 3, length 64
21:26:43.046576 IP 192.168.212.103 > dns.google: ICMP echo request, id 1626, seq 4, length 64
21:26:43.063942 IP dns.google > 192.168.212.103: ICMP echo reply, id 1626, seq 4, length 64
21:26:44.048991 IP 192.168.212.103 > dns.google: ICMP echo request, id 1626, seq 5, length 64
21:26:44.066766 IP dns.google > 192.168.212.103: ICMP echo reply, id 1626, seq 5, length 64
```
On voit bien l'envoi du ping (request) et la réponse (reply) dans les deux captures.

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
