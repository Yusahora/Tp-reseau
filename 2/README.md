# Sommaire

* [I. Mise en place du lab](#i-mise-en-place-du-lab)
  * [1. Routage statique](#2-routage-statique)
  * [2. Visualisation du routage avec Wireshark](#3-visualisation-du-routage-avec-wireshark)
* [II. NAT et services d'infra](#ii-nat-et-services-dinfra)
  * [1. Mise en place du NAT](#1-mise-en-place-du-nat)
  * [2. Serveur DHCP](#2-dhcp-server)
  * [3. Serveur NTP](#3-ntp-server)
  * [4. Serveur Web](#4-web-server)

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
`ZONE=trusted` dans le fichier /etc/sysconfig/network-scripts/ifcfg-interface_voulue sauf pour l'interface nat de route1 qui lui sera dans la zone public il suffit d'écrire `ZONE=public`.
 
 * Il faut aussi activer le masquerading sur la zone public pour router1 il faut faire les commandes suivantes
 `sudo firewall-cmd --add-masquerade --zone=public --permanent
  sudo firewall-cmd --reload`
 
* Pour ne pas avoir de problème de DNS vous pouvez ajouter dans le même fichier la ligne `DNS1=8.8.8.8`
 
* Puis pour avoir une route par défault qui vous permet d'atteindre internet ajouter aussi la ligne 'GATEWAY=10.2.12.2` il faut bien sur adapter l'adresse de la gateway a celle que vous utilisez.
 
* Il ne manque plus qu'a faire :
`sudo ifdown <interface voulue> 
 sudo ifup <interface voulue>`
 
 * Ces commandes permettent de recharger les configurations des cartes réseau pour prendre en compte les changements qui viennent d'être fait.
 
* On vérifie que tout fonctionne bien en faisant :
`curl google.com
<HTML><HEAD><meta http-equiv="content-type" content="text/html;charset=utf-8">
<TITLE>301 Moved</TITLE></HEAD><BODY>
<H1>301 Moved</H1>
The document has moved
<A HREF="http://www.google.com/">here</A>.
</BODY></HTML>`

* On à bien un retour donc tout fonctionne correctement.

## 2 Serveur DHCP

* Pour commencer on fait un clone de client1 que l'on va appeller dhcp1 et qui aura l'adresse ip `10.1.2.11`.

* Ensuite on installe un server dhcp :
`sudo yum install -y dhcp`

* Puis on écrit dans /ect/dhcp/dhcpd.conf la config suivante :
`
option domain-name "net1.tp2";

default-lease-time 600;
max-lease-time 7200;

authoritative;


log-facility local7;

subnet 10.2.1.0 netmask 255.255.255.0 {
  range 10.2.1.50 10.2.1.70;
  option domain-name "net1.tp2";
  option routers 10.2.1.254;
  option broadcast-address 10.2.1.255;
}``

* On démarre le serveur dhcp :
`sudo systemctl start dhcpd`

* Pour tester si il fonctionne on créé une vm avec un interface sur le réseau géré par notre dhcp avec la configuration suivante dans /etc/sysconfig/network-scripts/ifcfg-interface :
`TYPE="Ethernet"
BOOTPROTO=dhcp
NAME=interface
DEVICE=interface
ONBOOT="yes"

GATEWAY=10.2.12.2
DNS1=8.8.8.8
ZONE=public`

* Il ne reste plus qu'a faire `sudo dhclient pour récuperer une ip qui normalement sera 10.2.1.50 car vous êtes le premier client dhcp et que c'est la première adresse adressable par le server dhcp dans cette configuration.

## 3. Serveur NTP

* On commence par installer chrony sur toute les machines qui utilisent ntp :
`sudo yum install -y chrony`

* On ajoute ensuite le port 123 en UDP sur toute les machines pour puvoir utiliser chrony :
`sudo firewall-cmd --add-port=123/udp --permanent`

* On reload ensuite le firewall :
`sudo firewall-cmd --reload`

* Sur router1 qui nous servira de référence on modifie le fichier /etc/chrony.conf :
`
        server 0.fr.pool.ntp.org
        server 1.fr.pool.ntp.org
        server 2.fr.pool.ntp.org
        server 3.fr.pool.ntp.org

driftfile /var/lib/chrony/drift


makestep 1.0 3

rtcsync

allow 10.2.1.0/24

local stratum 10

keyfile /etc/chrony.keys

logdir /var/log/chrony

log measurements statistics tracking`

* On start chrony :
`sudo systemctl start chronyd`

* on vérifie si tout fonctionne corectement :
`chronyc tracking
Reference ID    : 33FFC594 (ovh.sqlserver.express)
Stratum         : 3
Ref time (UTC)  : Sun Mar 10 15:37:02 2019
System time     : 0.000114980 seconds fast of NTP time
Last offset     : +0.000336725 seconds
RMS offset      : 0.000482424 seconds
Frequency       : 21.904 ppm fast
Residual freq   : +0.002 ppm
Skew            : 0.046 ppm
Root delay      : 0.046410799 seconds
Root dispersion : 0.003142896 seconds
Update interval : 1032.0 seconds
Leap status     : Normal`

* On à bien réussi à se synchro avec les serveurs français.

* Sur toute les autres machine on modifie le fichier /etc/chrony.conf pour se synchro sur router1 :
`
server router1.net12.tp2 prefer

initstepslew 20 router1.net12.tp2

allow 10.2.1.254

driftfile /var/lib/chrony/drift

makestep 1.0 3

rtcsync

local stratum 10

keyfile /etc/chrony.keys

logdir /var/log/chrony

log measurements statistics tracking`

* On start chrony :
`sudo systemctl start chronyd`

* on vérifie si tout fonctionne corectement : 
` chronyc tracking
Reference ID    : 7F7F0101 ()
Stratum         : 10
Ref time (UTC)  : Sun Mar 10 15:48:59 2019
System time     : 0.000000000 seconds fast of NTP time
Last offset     : +0.000000000 seconds
RMS offset      : 0.000000000 seconds
Frequency       : 0.000 ppm slow
Residual freq   : +0.000 ppm
Skew            : 0.000 ppm
Root delay      : 0.000000000 seconds
Root dispersion : 0.000000000 seconds
Update interval : 0.0 seconds
Leap status     : Normal`

* Toute nos machine sont maintenant synchro sur router1 qui lui est synchro avec les serveurs français.

## 4 Serveur web

* On installe nginx comme serveur web sur server1 :
`sudo yum install -y epel-release
sudo yum install -y nginx`

* On ouvre le port 80 en tcp pour laisser passer les connexions à notre serveur :

`sudo firewall-cmd --add-port=80/tcp --permanent
 sudo firewall-cmd --reload`
 
 * On lance le serveur :
 `sudo systemctl start nginx`
 
 * on vérifie que tout fonctionne :
 `sudo systemctl status nginx`
 'sudo ss -altnp4
[sudo] Mot de passe de felix : 
State       Recv-Q Send-Q Local Address:Port               Peer Address:Port              
LISTEN      0      128         *:22                      *:*                   users:(("sshd",pid=976,fd=3))
LISTEN      0      100    127.0.0.1:25                      *:*                   users:(("master",pid=1207,fd=13))`

* On écoute bien sur le port 80

* On vérifie l'accès en faiant un curl depuis client 1 :
`curl server1
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.1//EN" "http://www.w3.org/TR/xhtml11/DTD/xhtml11.dtd">

<html xmlns="http://www.w3.org/1999/xhtml" xml:lang="en">
    <head>
        <title>Test Page for the Nginx HTTP Server on Fedora</title>
        <meta http-equiv="Content-Type" content="text/html; charset=UTF-8" />
        <style type="text/css">
            /*<![CDATA[*/
            body {
                background-color: #fff;
                color: #000;
                font-size: 0.9em;
                font-family: sans-serif,helvetica;
                margin: 0;
                padding: 0;
            }
            :link {
                color: #c00;
            }
            :visited {
                color: #c00;
            }
            a:hover {
                color: #f50;
            }
            h1 {
                text-align: center;
                margin: 0;
                padding: 0.6em 2em 0.4em;
                background-color: #294172;
                color: #fff;
                font-weight: normal;
                font-size: 1.75em;
                border-bottom: 2px solid #000;
            }
            h1 strong {
                font-weight: bold;
                font-size: 1.5em;
            }
            h2 {
                text-align: center;
                background-color: #3C6EB4;
                font-size: 1.1em;
                font-weight: bold;
                color: #fff;
                margin: 0;
                padding: 0.5em;
                border-bottom: 2px solid #294172;
            }
            hr {
                display: none;
            }
            .content {
                padding: 1em 5em;
            }
            .alert {
                border: 2px solid #000;
            }

            img {
                border: 2px solid #fff;
                padding: 2px;
                margin: 2px;
            }
            a:hover img {
                border: 2px solid #294172;
            }
            .logos {
                margin: 1em;
                text-align: center;
            }
            /*]]>*/
        </style>
    </head>

    <body>
        <h1>Welcome to <strong>nginx</strong> on Fedora!</h1>

        <div class="content">
            <p>This page is used to test the proper operation of the
            <strong>nginx</strong> HTTP server after it has been
            installed. If you can read this page, it means that the
            web server installed at this site is working
            properly.</p>

            <div class="alert">
                <h2>Website Administrator</h2>
                <div class="content">
                    <p>This is the default <tt>index.html</tt> page that
                    is distributed with <strong>nginx</strong> on
                    Fedora.  It is located in
                    <tt>/usr/share/nginx/html</tt>.</p>

                    <p>You should now put your content in a location of
                    your choice and edit the <tt>root</tt> configuration
                    directive in the <strong>nginx</strong>
                    configuration file
                    <tt>/etc/nginx/nginx.conf</tt>.</p>

                </div>
            </div>

            <div class="logos">
                <a href="http://nginx.net/"><img
                    src="nginx-logo.png" 
                    alt="[ Powered by nginx ]"
                    width="121" height="32" /></a>

                <a href="http://fedoraproject.org/"><img 
                    src="poweredby.png" 
                    alt="[ Powered by Fedora ]" 
                    width="88" height="31" /></a>
            </div>
        </div>
    </body>
</html>`

* On accède bien à la page html sur notre server web.

