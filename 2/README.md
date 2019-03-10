# Sommaire

* [I. Mise en place du lab](#i-mise-en-place-du-lab)
  * [1. Routage statique](#2-routage-statique)
  * [2. Visualisation du routage avec Wireshark](#3-visualisation-du-routage-avec-wireshark)
* [II. NAT et services d'infra](#ii-nat-et-services-dinfra)
  * [1. Mise en place du NAT](#1-mise-en-place-du-nat)
  * [2. Serveur DHCP](#2-dhcp-server)
  * [3. Serveur NTP](#3-ntp-server)
  * [4. Serveur Web](#4-web-server)
* [Bilan](#bilan)

# I Mise en place du lab

## 1. Routage statique

* On active l'ipv4 forwarding en ajoutant la ligne `net.ipv4.conf.all.forwarding=1`dans le fichier /etc/sysctl.conf sur router1 et router2 on fait aussi la commande `sudo sysctl -w net.ipv4.conf.all.forwarding=1`.

* On ajoute sur toute les machine les routes dont on à besoin en écrivant dans le fichier /etc/sysconfig/network-scripts/route-'interface voulue' la ligne suivante `<adresse réseau/netmask> via <ip_gateway> dev <interface>`

* On vérifie en faisant un `ping`de client1 vers server1 


`ping server1
PING server1 (10.2.2.10) 56(84) bytes of data.
64 bytes from server1 (10.2.2.10): icmp_seq=1 ttl=62 time=1.13 ms
64 bytes from server1 (10.2.2.10): icmp_seq=2 ttl=62 time=1.07 ms
64 bytes from server1 (10.2.2.10): icmp_seq=3 ttl=62 time=1.24 ms
64 bytes from server1 (10.2.2.10): icmp_seq=4 ttl=62 time=1.10 ms`

* On teste de ping server1 depuis router1

`ping server1
PING server1 (10.2.2.10) 56(84) bytes of data.
^C
--- server1 ping statistics ---
3 packets transmitted, 0 received, 100% packet loss, time 2001ms`
* Le `ping`ne fonctionne pas car server1 ne connais pas la route vers le router1 donc il reçois les ping mais ne peux pas renvoyer pong.

## 2. Visualisation du routage avec Wireshark

* On ping server1 depuis client1 et on observe le trafique qui passe sur router2
en faisant des captures.

`sudo tcpdump -i enp0s8 -w ping-enp0s8-r2.pcap`
![alt text](/2/screens/ping-enp0s8-r2.png "Whireshark")

`sudo tcpdump -i enp0s3 -w ping-enp0s3-r2.pcap`
![alt text](/2/screens/ping-enp0s3-r2.png "Whireshark")

* Tram net12 
![alt text](/2/screens/mac-enp0s8-r2.png "Whireshark")
 
* Tram net 2
 
![alt text](/2/screens/mac-enp0s3-r2.png "Whireshark")
 
* On constate que les adresses mac on changer c'est le router qui s'en ai charger pour faire passer le trafic sur le réseau.

# II NAT et services d'infra

## 1 Mise en place du NAT

* On vérifie si router1 accède à internet
`ping google.com
PING google.com (216.58.204.238) 56(84) bytes of data.
64 bytes from par21s06-in-f14.1e100.net (216.58.204.238): icmp_seq=1 ttl=63 time=15.1 ms
64 bytes from par21s06-in-f14.1e100.net (216.58.204.238): icmp_seq=2 ttl=63 time=18.1 ms
64 bytes from par21s06-in-f14.1e100.net (216.58.204.238): icmp_seq=3 ttl=63 time=18.2 ms
64 bytes from par21s06-in-f14.1e100.net (216.58.204.238): icmp_seq=4 ttl=63 time=17.9 ms
^C
--- google.com ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3006ms
rtt min/avg/max/mdev = 15.144/17.390/18.255/1.311 ms`

* C'est bon le nat est fonctionnel sur router1.

* On met toute les interfaces de toutes les vms dans la zone trusted en ajoutant la ligne
`ZONE=trusted` dans le fichier /etc/sysconfig/network-scripts/ifcfg-<interface voulue> sauf pour l'interface nat de route1 qui lui sera dans la zone public il suffit d'écrire `ZONE=public`.

