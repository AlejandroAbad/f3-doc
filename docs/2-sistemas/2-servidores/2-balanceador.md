---
sidebar_position: 2
title: Balanceador
---

# Despliegue de un balanceador de carga
Partimos de una máquina SLES 12.5 recien instalada. A continuación detallaremos la configuración necesaria.

## Filesystems
Aparte de los filesystems habituales de los sistemas Linux, un balanceador no necesita ninguna configuración especial de filesystems.

```
# lsblk
NAME            MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda               8:0    0  512M  0 disk
└─sda1            8:1    0  511M  0 part /boot
sdb               8:16   0   20G  0 disk
├─rootvg-lvroot 254:0    0    2G  0 lvm  /
├─rootvg-lvusr  254:1    0    8G  0 lvm  /usr
├─rootvg-lvhome 254:3    0  512M  0 lvm  /home
├─rootvg-lvopt  254:4    0    1G  0 lvm  /opt
├─rootvg-lvtmp  254:5    0    1G  0 lvm  /tmp
├─rootvg-lvvar  254:6    0    2G  0 lvm  /var
└─rootvg-lvsrv  254:7    0   32M  0 lvm  /srv
sdc               8:32   0    2G  0 disk
└─swapvg-lvswap 254:2    0    2G  0 lvm  [SWAP]
```


## Interfaces de red
Como ya vimos en el capítulo de [arquitectura de red](/docs/sistemas/arquitectura/red), un balanceador cuenta con dos interfaces de red.
Configuraremos las interfaces tal que:
- **eth0**: Interfaz conectada a la red INTERNA de la sede donde se encuentre el servidor *(Por ejemplo, en Santomera: `192.168.10.0/28`)*. 
- **eth1**: Interfaz conectada a la red EXTERNA. La configuración específica de este interzaz depende de si el balanceador va a ubicarse en Madrid o en Santomera. 

### Configuración de red EXTERNA
#### Configuración de red EXTERNA en Madrid
Como en Madrid únicamente tenemos un balanceador, la configuración de red EXTERNA para este no tiene nada especial. Simplemente tendrá establecida en el interfaz **eth1** la dirección IP de acceso publica del servicio Fedicom.

El servidor tendrá como gateway por defecto el gateway de la red EXTERNA de Madrid. La configuración quedará tal que:

```bash
# route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
default         (oculto)        0.0.0.0         UG    0      0        0 eth1
(oculto)        *               255.255.254.0   U     0      0        0 eth1
192.168.10.16   *               255.255.255.240 U     0      0        0 eth0
```


:::note
Como se puede deducir de lo comentado anteriormente, el balanceador de Madrid no tiene acceso a la red salvo cuando la red de Santomera se encuentra caída. Por lo tanto, para poder actualizar el nodo, deberemos añadirle temporalmente un interfaz **eth2** con una dirección de la red HEFAME.
:::

#### Configuración de red EXTERNA en Santomera
En el caso de Santomera, los servidores de balanceo ejecutan el protocolo VRRP para dar alta disponibilidad con otros balanceadores de la sede, luego el manejo de esta IP virtual queda en manos de el demonio [`keepalived`](/docs/sistemas/haslb/keepalived). Por este motivo, las interfaces de acceso a la red EXTERNA, las **eth1**, se configurarán con una dirección IP de mantenimiento que no es la que los clientes usaran para acceder al servicio.

Cada servidor tendrá como puerta de enlace por defecto al router de salida a internet:

```bash
# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         (oculto)        0.0.0.0         UG    0      0        0 eth1
(oculto)        0.0.0.0         255.255.252.0   U     0      0        0 eth1
192.168.100.0   0.0.0.0         255.255.255.240 U     0      0        0 eth0

```
:::tip
La configuración del interfaz `eth1` la administra el demonio *Keepalived* de manera dinámica, en función de si el servidor es o no es el nodo activo en la red.
:::


### DNS y NTP
Los balanceadores usaran los DNS públicos de google: `8.8.8.8` y `8.8.4.4`.
Para mantener la hora del servidor, configuraremos el servidor NTP `es.pool.ntp.org`.



#### Fichero `/etc/hosts`

Definiremos en el fichero de hosts local las siguientes direcciones:

```bash
127.0.0.1       localhost

192.168.10.1    f3san1
192.168.10.2    f3san2
192.168.10.3    f3san1-fw
192.168.10.4    f3san2-fw

192.168.10.17   f3mad1
192.168.10.18   f3mad1-fw
```



### Cortafuegos
Configuraremos el cortafuegos del servidor para proteger el acceso desde el interfaz de la red de VODAFONE. Esto es, protegeremos el servidor del tráfico proviniente del interfaz `eth1`.


```bash
Yast > 
  System > 
  Security and Users > 
    Firewall
      - En "Start-Up" marcamos la "Enable Firewall Automatic Starting"
      - En "Interfaces" marcamos las interfaces tal que así:
          Device            │ Interface │ Configured In
          VMXNET3 Ethernet  │ eth0      │ Internal Zone
          VMXNET3 Ethernet  │ eth1      │ External Zone
      - En "Allowed Services"
        · Para la zona "External Zone", habilitamos:
          + HTTP Server     │ Opens ports for Apache Web Server.   
          + HTTPS Server     │ Opens ports for Apache Web Server.     
```

En resumen, el sumario de la configuración debe quedar tal que:

```
Firewall Starting
  *  Enable firewall automatic starting
  *  Firewall starts after the configuration has been written

Internal Zone
  Interfaces:
    +  eth0
  Open Services, Ports, and Protocols:
    +  Internal zone is unprotected. All ports are open.
    
Demilitarized Zone
  *  No interfaces assigned to this zone.
  
External Zone
  Interfaces:
    +  eth1
      Open Services, Ports, and Protocols:
        +  HTTP Server
        +  HTTPS Server
```

### Envío de correos
Para que pueda enviar correos, configuraremos el servidor `postfix` para que use un concentrador como relay. Para esto, modificaremos el fichero de configuración `/etc/postfix/main.cf`:

```bash
inet_protocols = ipv4
relayhost = 192.168.10.1 # Usar la IP de un concentrador alcanzable
myhostname = f3san1-fw.hefame.es # Usar el nombre del conentrador
```

Para aplicar los cambios, reiniciaremos el servicio `postfix`:

```bash
systemctl restart postfix
```

### Software
Un concentrador Fedicom 3 contendrá el siguiente software instalado:

- [Servidor Apache2 con módulos para el balanceo de carga](/docs/sistemas/haslb/apache2)
- **(Opcional)** [Demonio Keepalived para la implementación de VRRP](/docs/sistemas/haslb/keepalived)

