# TP3 : Routage INTER-VLAN + mise en situation

# Sommaire

- [TP3 : Routage INTER-VLAN + mise en situation](#tp3--routage-inter-vlan--mise-en-situation)
- [Sommaire](#sommaire)
- [Intro](#intro)
- [0. Etapes préliminaires](#0-etapes-pr%c3%a9liminaires)
- [I. *Router-on-a-stick*](#i-router-on-a-stick)
- [II. Cas concret](#ii-cas-concret)

# Intro

On va aborder un cas un peu plus concret dans ce TP. Vous aurez besoin d'un peu de votre créativité pour arriver jusqu'au bout.  

Vous aller avoir besoin d'un routeur dans ce TP et de quelques petits détails de conf qu'on a pas encore vu.  

Afin de vous refaire pratiquer un peu, et de vous faire aborder ces quelques nouvelles notions, **le TP se découpe en deux temps** :
* **appréhension d'une topologie classique : *Router-on-a-stick***
  * on remet un peu du routeur là dedans, et on aborde le routage inter-VLAN
  * partie courte, c'est une intro à la seconde partie
* **cas concret**
  * je vous soumets un problème, vous me répondez avec une infra :)

Comme toujours...  
**Allez à votre rythme, prenez le temps de comprendre.**  
**Posez des questions.**  
**Prenez des notes au fur et à mesure.**  
**Lisez les parties en entier avant de commencer à travailler dessus.**

> **Référez-vous [au README des TPs](/tp/README.md) pour des infos sur le déroulement et le rendu des TPs.**

# 0. Etapes préliminaires

* avoir lu [le README des TPs](/tp/README.md)
* **Wireshark** installé
* GNS3 fonctionnel (lecture du [mémo/setup GNS3](/memo/setup-gns3.md))
* Lecture du [mémo CLI Cisco](/memo/cli-cisco.md)

**Dans ce TP, vous pouvez considérez que :**
* les `PC` sont [des VPCS de GNS3](/memo/setup-gns3.md#utilisation-dun-vpcs) (sauf indication contraire)
* les `P` sont des imprimantes, on les simulera avec des VPCS aussi
* les `SRV` sont ds serveurs, VPCS again
* les `SW` sont des Switches Cisco, virtualisé avec [l'IOU L2](/memo/setup-gns3.md#get-and-setup-iou)
* les `R` sont des routeurs, virtualisé avec l'iOS dispo ici : [Cisco 3640](https://drive.google.com/drive/folders/1DFe2u5tZldL_y_UYm32ZbmT0cIfgQM2p)

# I. *Router-on-a-stick*

C'est le cas d'école typique pour mettre en place du routage inter-VLAN.  

L'idée est de pouvoir autoriser certains VLANs à se joindre, mais pas d'autres :
* avec les VLANs on isole les gens au niveau 2 (Ethernet)
* avec le routage inter-VLAN, on permet de passer outre les VLANs en faisant appel au niveau 3
* **l'idée c'est qu'à aucun moment on change le fonctionnement des VLANs, on autorise juste un routeur à faire son taff : router entre deux réseaux**, qu'ils correspondent à des VLANs différents ou non
* let's goooo

Schéma moche ftw :

```
             +--+
             |R1|
             +-++
               |
               |                    +---+
               |          +---------+PC4|
+---+        +-+-+      +---+       +---+
|PC1+--------+SW1+------+SW2|
+---+        +-+-+      +-+--+
               |          |  |
               |          |  +------+--+
               |          |         |P1|
             +-+-+      +-+-+       +--+
             |PC2|      |PC3|
             +---+      +---+
```

**Tableau des réseaux utilisés**

Réseau | Adresse | VLAN | Description
--- | --- | --- | ---
`net1` | `10.3.10.0/24` | 10 | Utilisateurs
`net2` | `10.3.20.0/24` | 20 | Admins
`net3` | `10.3.30.0/24` | 30 | Visiteurs
`netP` | `10.3.40.0/24` | 40 | Imprimantes

**Qui peut joindre qui ?**

✅ = peuvent se joindre
❌ = ne peuvent pas se joindre

Réseaux | `net1` |  `net2` |  `net3` |  `netP`
--- | --- | --- | --- | ---
 `net1` | ✅ | ❌ | ❌ | ✅
 `net2` | ❌ | ✅ | ❌ | ✅
 `net3` | ❌ | ❌ | ✅ | ✅
 `netP` | ✅ | ✅ | ✅ | ✅

**Tableau d'adressage**

Machine | VLAN | IP `net1` | IP `net2` | IP `net3` |  IP `netP`
--- | --- | --- | --- | --- | ---
PC1 | 10 | `10.3.10.1/24` | x | x | x
PC2 | 20 | x | `10.3.20.2/24` | x | x | x
PC3 | 30 | x | `10.3.20.3/24` | x | x | x
PC4 | 30 | x | x |  `10.3.30.4/24` | x | x
P1 | 40 | x | x | x | `10.3.40.1/24` 
R1 | x |  `10.3.10.254/24` | `10.3.20.254/24` | `10.3.30.254/24` | `10.3.40.254/24` 

**Instructions** (pretty straightforward) :
* Setup this shit
* You'll need inter-VLAN routing to make it work properly
  * se référer au [mémo Cisco section sous-interface](/memo/cli-cisco.md#sous-interface)
* 🌞 Prove me that your setup is actually working
  * think about VLANs, `ping`, etc.

**CHECK MATE !**

# II. Cas concret

> C'est cool si vous jouez un peu le jeu et que vous imaginez quelque chose d'original. *Vous pouvez (comme aux autres TPs) pomper sur les autres mais ça a encore moins d'intérêt que d'habitude n_n !*

**Creusez-vous un peu la tête.**  

Le but est de mettre en place une infra qui répond au besoin des bureaux représentés ci-dessous :

![Yo](./pics/schema-II.png)

* `R1` `R3` `R4` et `R5` sont des bureaux avec des utilisateurs
* `R2` est une salle serveur 
* le bâtiment a une taille de 20m x 20m (approximativement, vous en aurez besoin sur la fin)

**C'est quoi ces machines ?**

Type | Nom | Rôle | Dans GNS 
--- | --- | --- | ---
`A` | Admins | Accès à tout à frer. Full power. | VPCS
`U` | Users | Accès à un peu moins. | VPCS
`S` | Stagiaires | Encore un peu moins. | VPCS
`SRV` | Serveurs | Services hébergés en local. Ceux encadrés en rouge sont des **serveurs sensibles ou SS** | VPCS (ou autre si explicitement demandé)
`P` | Imprimantes | Imprimantes dispo en réseau

**Qui a accès à qui exactement ?**

✅ = peuvent se joindre
❌ = ne peuvent pas se joindre

X | Admins | Users | Stagiaires | Serveurs | SS | Imprimantes
--- | --- | --- | --- | --- | --- | --- | 
Admins | ✅ | ❌ | ❌ | ✅ | ✅ | ✅ |
Users | ❌ | ✅ | ❌ | ✅ | ❌ | ✅ |
Stagiaires | ❌ | ❌ | ✅ | ❌ | ❌ | ✅ |
Serveurs | ✅ | ✅ | ❌ | ✅ | ❌ | ✅ |
Serveurs sensibles | ✅ | ❌ | ❌ | ❌ | ✅ | ❌ |
Imprimantes | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ |

**Exceptions** *(ce sont des bonus, voir la fin du TP*)
* tous les postes ne peuvent joindre que l'imprimante de leur propre salle
* les serveurs sensibles n'ont pas accès à internet
* seul l'admin 1 (`A1`) a accès au serveur 4 (`SRV4`)

---

**TODO**
* setup this shit in GNS3
  * matériel autorisé : routeurs (Cisco 3640), switches (IOU L2 Cisco), VPCS
  * outils : routage statique, VLAN, votre talent
* pour la partie soft
  * 🌞 dimensionnez intelligemment les réseaux
    * prévoyez une augmentation légère
  * 🌞 permettre un accès internet à tout le monde
* pour la partie hard
  * 🌞 proposez un nombre de routeur et de switches et précisez à quel endroit physique ils se trouveront
  * 🌞 précisez le nombre de câbles nécessaires et une longueur (approximative)
    * court : moins de 1m
    * moyen : entre 1 et 5m
    * long : 5m+
    * **le but c'est d'avoir un ordre de grandeur**, on s'en fout complet des tailles exactes pour ce TP
* 🌞 livrer, en plus de l'infra, des éléments qui rendent compte de l'infra (de façon simple)
  * schéma réseau (screen GNS ?)
  * référez-vous à la partie I. (tableau des réseaux utilisés, tableau d'adressage)
* **être en mesure de prouver que l'infra fonctionne comme demandé**

**Conseils**
* **avant de vous lancer** réfléchissez aux différentes étapes qui vous permettront de réaliser le TP
  * je vous conseille par exemple de faire un schéma et un plan d'adressage **en premier**
* documentez ce que vous faites au fur et à mesure
* n'oubliez pas de sauvegarder la configuration des équipements réseau et celle des VPCS

---

**Bonus**
* 🐙 mettre en place les exceptions
  * documentez-vous, proposez des choses
* 🐙 mettre en place un serveur DHCP 
  * il devra 
    * s'intégrer à l'existant
    * être installé sur une VM dédiée (Virtualbox, Workstation)
    * permettre l'attribution d'IPs pour tous les PCs clients (admins, users, stagiaires)
    * libre choix de l'OS (m'enfin, déconnez pas, on va pas mettre un Windows Server 2016 si ?...)
  * mise en place d'un test avec l'ajout d'un nouveau client
