# Sommaire

* [I. Exploration du réseau d'une machine CentOS](#i-exploration-du-réseau-dune-machine-centos)
  * [1. Mise en place](#1-mise-en-place)
  * [2. Basics](#2-basics)
    * [Routes](#routes)
    * [Table ARP](#table-arp)
    * [Capture réseau](#capture-réseau)
* [II. Communication simple entre deux machines](#ii-communication-simple-entre-deux-machines)
  * [1. Mise en place](#1-mise-en-place-1)
  * [2. Basics](#2-basics-1)
    * [`ping` et ARP](#ping-et-arp)
    * [`netcat` : TCP et UDP](#netcat)
  * [3. Bonus : ARP spoofing](#3-bonus--arp-spoofing)
* [III. Routage statique simple](#iii-routage-statique-simple)

# I. Exploration du réseau d'une machine CentOS

## 1. Mise en place

* Combien y a-t-il d'adresses disponibles dans un `/24` ?
   * Il y a 254 adresses disponibles

* Combien y a-t-il d'adresses disponibles dans un `/30` ?
   * Il y a 2 adresses disponibles
  
* Quelle est l'utilité d'un /30 ?
   * Un /30 n'a que 2 adresses disponibles il est donc facile de voir qui est sur ce réseau de manière claire

* NAT : 
    * `curl google.com` 
    * le code d'erreur 301 signifie une **redirection permanente**.
    
* NET1 : 
    * `ping 10.1.1.1`
    * ça fonctionne
    
* NET2 : 
    * `ping 10.1.2.1`
    * ça fonctionne
    
## 1. Basics

`ip route show
10.0.2.0/24 dev enp0s3 proto kernel scope link src 10.0.2.15 metric 100 
10.1.1.0/24 dev enp0s8 proto kernel scope link src 10.1.1.2 metric 100 
10.1.2.0/30 dev enp0s9 proto kernel scope link src 10.1.2.2 metric 100 `

* Route 1 via  enp0s3 (NAT) permet l'accès à `10.0.2.0/24 `.

* Route 2 via  enp0s8 (NET1) permet l'accès à `10.1.1.0/24 `

* Route 3 via  enp0s9 (NET2) permet l'accès à `10.1.2.0/30 `

`[felix@localhost ~]$ sudo ip route del 10.1.2.0/30
[felix@localhost ~]$ ip route show
10.0.2.0/24 dev enp0s3 proto kernel scope link src 10.0.2.15 metric 100 
10.1.1.0/24 dev enp0s8 proto kernel scope link src 10.1.1.2 metric 100 `

* La route de NET2 n'est plus la

`[felix@localhost ~]$ ping 10.2.1.1
connect: Le réseau n'est pas accessible`

* L'accès à NET2 n'est plus disponible

`ip route show
default via 10.0.2.2 dev enp0s3 proto static metric 100 
10.0.2.0/24 dev enp0s3 proto kernel scope link src 10.0.2.15 metric 100 
10.1.1.0/24 dev enp0s8 proto kernel scope link src 10.1.1.2 metric 100 
10.1.2.0/30 dev enp0s9 proto kernel scope link src 10.1.2.2 metric 100 `

* Les routes sont à nouveau toutes présente

`ping 10.2.1.1
PING 10.2.1.1 (10.2.1.1) 56(84) bytes of data.
64 bytes from 10.2.1.1: icmp_seq=1 ttl=63 time=0.427 ms`
* NET2 est à nouveau disponible

`ip neigh show
10.1.1.1 dev enp0s8 lladdr 0a:00:27:00:00:00 REACHABLE`

* Cette ligne correspond à l'adresse ip de mon hôte, c'est la seul adresse présente car je n'ai eu de contact que avec mon hôte

`sudo ip neigh flush all
ip neigh show`

* Rien ne s'affiche la table arp est donc vide

`10.1.1.1
PING 10.1.1.1 (10.1.1.1) 56(84) bytes of data.
64 bytes from 10.1.1.1: icmp_seq=1 ttl=64 time=0.365 ms
ip neigh show
10.1.1.1 dev enp0s8 lladdr 0a:00:27:00:00:00 REACHABLE`

### Capture réseau

* On lance la capture puis un ping puis on arrete la capture

`
sudo tcpdump -i enp0s9 -w ping.pcap
[sudo] Mot de passe de felix : 
tcpdump: listening on enp0s9, link-type EN10MB (Ethernet), capture size 262144 bytes
^C10 packets captured
10 packets received by filter
0 packets dropped by kernel`

![alt text](/1/screens/ping.png "Whireshark")

## Communication simple entre deux machines

## 1. Mise en place

`ping -c 4 10.1.1.3
PING 10.1.1.3 (10.1.1.3) 56(84) bytes of data.
64 bytes from 10.1.1.3: icmp_seq=1 ttl=64 time=0.928 ms
64 bytes from 10.1.1.3: icmp_seq=2 ttl=64 time=0.699 ms
64 bytes from 10.1.1.3: icmp_seq=3 ttl=64 time=0.841 ms
64 bytes from 10.1.1.3: icmp_seq=4 ttl=64 time=0.589 ms`

* On ping client2 après avoir vider la table arp

`ip neigh show
10.1.1.3 dev enp0s8 lladdr 08:00:27:5c:6c:40 REACHABLE
10.1.2.1 dev enp0s9 lladdr 0a:00:27:00:00:02 REACHABLE`

* Client2 est maintenant dans notre table arp


`sudo tcpdump -i enp0s8 -w ping2.pcap
tcpdump: listening on enp0s8, link-type EN10MB (Ethernet), capture size 262144 bytes
^C10 packets captured
10 packets received by filter
0 packets dropped by kernel`


![alt text](/1/screens/ping2.png "Whireshark")

##udp

`sudo firewall-cmd --add-port=8888/udp --permanent
[sudo] Mot de passe de felix : 
success`

* on ouvre le port 8888 en udp sur client1

`sudo firewall-cmd --add-port=8888/tcp --permanent
success
[felix@localhost ~]$ sudo firewall-cmd --reload`

* on fait pareille en tcp pour la suite du tp puis on reload le firewall pour que les changements sois pris en compte

![alt text](/1/screens/chat.png "Whireshark")

* le chat fonctionne

` ss -unp
Recv-Q Send-Q Local Address:Port               Peer Address:Port              
0      0          10.1.1.3:47110                  10.1.1.2:8888                users:(("nc",pid=1442,fd=3))`

* ss sur client2

`ss -unp
Recv-Q Send-Q Local Address:Port               Peer Address:Port              
0      0          10.1.1.2:8888                   10.1.1.3:47110               users:(("nc",pid=9440,fd=4))`

* ss sur client1

![alt text](/1/screens/nc-udp.png "Whireshark")

* On peut voir la connexion en udp

![alt text](/1/screens/chattcp.png "Whireshark")

* Le chat fonctionne bien en tcp

![alt text](/1/screens/nc-tcp.png "Whireshark")

* On peut voir la mise en place de la connexion tcp avec le syn ack

`sudo firewall-cmd --remove-port=8888/tcp --permanent
success
sudo firewall-cmd --reload
success`

* on enlève le tcp de firewall et on le reload

`nc  10.1.1.2 8888
Ncat: No route to host.`

* Client2 ne peut plus se connecter en tcp à client1

![alt text](/1/screens/firewall.png "Whireshark")

* Le firewall a bien empêché la connexion

# III. Routage statique simple 

`sudo sysctl -w net.ipv4.ip_forward=1
[sudo] Mot de passe de felix : 
net.ipv4.ip_forward = 1`

* On active le forward d'IP sur client1

`sudo ip route add 10.1.2.0/30 via 10.1.2.2 dev enp0s8`

* on ajoute la route vers net2 

`ping 10.1.2.2
PING 10.1.2.2 (10.1.2.2) 56(84) bytes of data.
64 bytes from 10.1.2.2: icmp_seq=1 ttl=64 time=0.915 ms
64 bytes from 10.1.2.2: icmp_seq=2 ttl=64 time=0.629 ms
64 bytes from 10.1.2.2: icmp_seq=3 ttl=64 time=0.730 ms
64 bytes from 10.1.2.2: icmp_seq=4 ttl=64 time=0.984 ms`

* On peut bien ping sur net2 avec client2

