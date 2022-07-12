---
sidebar_position: 1
title: Concentrador
---

# Despliegue de un concentrador
Partimos de una máquina SLES 12.5 recien instalada. A continuación detallaremos la configuración necesaria.


## Filesystems
Aparte de los filesystems habituales de los sistemas Linux, un concentrador únicamente tendrá un volúmen exclusivo para la base de datos MongoDB.
Lo recomendado para la version de mongo que vamos a instalar, es utilizar un filesystem XFS para albergar la base de datos. 
En concreto, crearemos un VG con un LV exclusivo para el directorio `/var/lib/mongo`, donde se almacena la base de datos. Tal que:

```
$ lsblk
NAME              MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                 8:0    0  512M  0 disk
└─sda1              8:1    0  511M  0 part /boot
sdb                 8:16   0   20G  0 disk
├─rootvg-lvroot   254:0    0  2.5G  0 lvm  /
├─rootvg-lvusr    254:1    0    8G  0 lvm  /usr
├─rootvg-lvhome   254:3    0  2.5G  0 lvm  /home
├─rootvg-lvopt    254:4    0    1G  0 lvm  /opt
├─rootvg-lvtmp    254:5    0    1G  0 lvm  /tmp
├─rootvg-lvvar    254:6    0    2G  0 lvm  /var
└─rootvg-lvsrv    254:7    0   32M  0 lvm  /srv
sdc                 8:32   0    8G  0 disk
└─swapvg-lvswap   254:2    0    8G  0 lvm  [SWAP]
sdd                 8:48   0   50G  0 disk
└─mongovg-lvmongo 254:8    0   40G  0 lvm  /var/lib/mongo
```

```
# cat /etc/fstab | grep mongo
/dev/mongovg/lvmongo    /var/lib/mongo  xfs     defaults        1 2

# mount | grep mongo
/dev/mapper/mongovg-lvmongo on /var/lib/mongo type xfs (rw,relatime,attr2,inode64,noquota)
```


## Tunning de memoria
Es recomendable deshabilitar el uso de HugePages para el uso de MongoDB. Para esto:

```bash
Yast >
	System >
  	Boot Loader >
    	Kernel Parameters
    		- En la opción "Optional Kernel Command Line Parameter", añadimos "transparent_hugepage=never"
```

Si queremos aplicar la configuración sin reiniciar, podemos ejecutar lo siguiente:
:::caution
 **Importante**: Esta accicón NO guarda la configuración para el próximo reinicio
:::

```
echo never > /sys/kernel/mm/transparent_hugepage/enabled
```

## Interfaces de red
Como ya vimos en el capítulo de [arquitectura de red](/docs/sistemas/arquitectura/red), un concentrador cuenta con dos interfaces de red.
Configuraremos las interfaces tal que:
- **eth0**: Interfaz conectada a la red LAN de HEFAME de la sede donde se encuentre el servidor. 
- **eth1**: Interfaz conectada a la red DMZ de la sede donde se encuentre el servidor.

El servidor tendrá como gateway por defecto el gateway de la red LAN HEFAME donde esté conectado, y tendrá rutas establecidas para alcanzar al resto de redes INTERSAP. Por ejemplo en Santomera:

```
# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         (oculto)        0.0.0.0         UG    0      0        0 eth0
(oculto)        0.0.0.0         255.255.255.0   U     0      0        0 eth0
192.168.10.0    0.0.0.0         255.255.255.240 U     0      0        0 eth1
```

### DNS y NTP
Los concentradores usarán los servidores DNS y NTP internos de la red de HEFAME.

#### Configuración `/etc/hosts`

Definiremos en el fichero de hosts:
- El nombre especial `proxysap` apuntando al host local, que es donde se encontrará escuchando el servidor Apache para balanceo a SAP. *En el caso de desarrollo, usaremos el nombre `proxysap-dev`*.
- Los nombres de los cortafuegos de la red INTERFEDI del nodo.


```bash
127.0.0.1       localhost

# Direcciones de los cortafuegos en la red interna
192.168.10.3   f3san1-fw f3san1-fw.hefame.es
192.168.10.4   f3san2-fw f3san2-fw.hefame.es
192.168.10.18  f3mad1-fw f3mad1-fw.hefame.es

# Servidores SAP (Para resolverlos sin depender del DNS)
x.x.x.1  sap1p01
x.x.x.2  sap2p01
...
x.x.x.7  sap7p01
x.x.x.8  sap8p01

# Alias para el Proxy Balanceador a SAP local
127.0.0.1       proxysap proxysap.hefame.es
```


### Cortafuegos
Configuraremos el cortafuegos del servidor para proteger el acceso desde el interfaz de las redes INTERNAS. Esto es, protegeremos el servidor del tráfico proviniente del interfaz `eth1`.


```bash
Yast > 
  System > 
    Security and Users > 
      Firewall
        - En "Start-Up" activamos "Enable Firewall Automatic Starting"
        - En "Interfaces" clasificamos de la siguiente manera:
            Device           │ Interface │ Configured In
            VMXNET3 Ethernet │ eth0      │ Internal Zone
            VMXNET3 Ethernet │ eth1      │ External Zone
        - En "Allowed Services" para la zona "External Zone":
          - Añadimos el servicio "SMTP with Postfix". # Para hacer forward del correo de los balanceadores
          - En "Advanced ...":
            · TCP 5000 # Para recibir peticiones Fedicom3.
            · TCP 5001 # Para recibir peticiones de monitorización.
```

En resumen, el sumario de la configuración debe quedar tal que:

```bash
Firewall Starting
  *  Enable firewall automatic starting
  *  Firewall starts after the configuration has been written

Internal Zone
  Interfaces:
    +  eth0
  Open Services, Ports, and Protocols:
    +  Internal zone is unprotected. All ports are open.

Demilitarized Zone:
  *  No interfaces assigned to this zone.

External Zone
  Interfaces:
    +  eth1
  Open Services, Ports, and Protocols:
    +  SMTP with Postfix
	+  TCP Ports: 5000, 5001
```



### Envío de correos
Para que pueda enviar correos, configuraremos el servidor `postfix` para que use el relay de correo de Hefame. Para esto, modificaremos el fichero de configuración `/etc/postfix/main.cf`. También modificaremos el parámetro `inet_interfaces` para permitir que los balanceadores usen al concetrador como relay de correo.

```
inet_interfaces = all
myhostname = f3san1.hefame.es # Cambiar por el nombre del concentrador actual
relayhost = correo.hefame.es
```

Para aplicar los cambios, reiniciaremos el servicio `postfix`:

```bash
systemctl restart postfix
```

## Software
Un concentrador Fedicom 3 contendrá el siguiente software estándard instalado:

- Runtime JavaScript **Node.js v16**
```bash
SUSEConnect -p sle-module-web-scripting/12/x86_64
zypper install nodejs16
```

- Control de versiones GIT
```bash
zypper install git-core
```

El resto de software necesario para el funcionamiento del concentrador los detallaremos en siguientes capítulos:
- [Base de datos MongoDB](/docs/sistemas/mongodb)
- [Servidor Apache2 con módulos para el balanceo de carga](/docs/sistemas/haslb)
- [Servidor de aplicación Fedicom3](/docs/sistemas/aplicacion)







