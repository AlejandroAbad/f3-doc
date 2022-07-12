---
sidebar_position: 4
title: KeepAlived
---

# VRRP con KeepAlived
*Virtual Router Redundancy Protocol* (VRRP) es un protocolo diseñado para la alta disponibilidad del gateway (router de salida) de una subred. Para esto se utiliza una IP virtual como la dirección del gateway en lugar de la IP del router físico, y varios routers se configuran para que, usando VRRP, solo uno de ellos tenga la IP virtual a la vez. 
:::tip
En nuestro caso, vamos a utilizar este protocolo para la alta disponibilidad entre dos máquinas Linux que hacen de balanceadores de carga a la entrada de pedidos.
:::

## Las tripas de VRRP
Los servidores pertenecientes a un mismo grupo virtual se intercambian mensajes del protocolo VRRP, típicamente cada segundo. A través de estos mensajes los servidores del grupo pueden saber el estado del resto de miembros del grupo:
- Los miembros pueden detectar si el nodo que actúa como maestro ha caido al no recibir mensajes de este tras cierto tiempo.
- El miembro que actúa como maestro puede notificar al resto si ha detectado algún fallo que le impida operar y por tanto no puede seguir como maestro.

Una vez se determina la necesidad de conmutación, el miembro con mayor prioridad tomará el papel de maestro y a partir de este momento presentará la IP virtual a la red.

La gran ventaja de VRRP es que es totalmente transparente a los nodos y equipos de la red: En el caso de que un equipo envíe un paquete a la dirección MAC asociada a la IP virtual, pero el maestro haya cambiado, no recibirá respuesta pues la MAC asociada es ahora la del nuevo maestro. Por tanto, enviará una petición ARP para descubrirla de nuevo y en este momento el nuevo maestro le devolverá la dirección MAC de su interfaz de conexión a la red local. A partir de este momento. este equipo de red ya cursará todo el tráfico a otras redes a través del nuevo servidor.


## Instalación de KeepAlived
Para implementar el protocolo VRRP entre los balanceadores de carga de Santomera (`f3san1-fw` y `f3san2-fw`) vamos a usar el demonio Linux llamado `keepalived`.

Para instalarlo en SUSE 12.5 deberemos tener activada la extensión `SUSE Package Hub 12 SP5 x86_64`, que podemos activar de la siguiente manera:

```bash
SUSEConnect -p PackageHub/12.5/x86_64
zypper ref
```

Una vez hecho esto, ya tendremos acceso al paquete de instalación del demonio en la herramienta `zypper`. Podemos instalarlo y configurarlo para que arranque automáticamente:

```bash
zypper install keepalived
systemctl enable keepalived
```

## Configuración de KeepAlived
Una vez el paquete ha sido instalado en las máquinas que van a formar el grupo podemos configurarlas editando el fichero `/etc/keepalived/keepalived.conf`.
Deberemos elegir uno de los nodos como **MAESTRO**, pues recordemos que VRRP es activo-pasivo.

En el nodo designado como **MAESTRO**, la configuración será:

```
global_defs {
    router_id F3SAN1-FW
    notification_email {
        alejandro.abad@hefame.es
        operacion@hefame.es
    }
    notification_email_from cortafuegos@fedicom3.hefame.es
    smtp_server localhost
    enable_script_security
    # Para evitar fallos, envio de Gratuitous ARPs:
    vrrp_garp_lower_prio_delay 5
    vrrp_garp_lower_prio_repeat 5
    vrrp_garp_master_refresh 30
    vrrp_garp_master_refresh_repeat 2
    vrrp_higher_prio_send_advert true
}

vrrp_script chk_apache {
    script "/sbin/pidof httpd-worker"
    interval 2
    user root root
}

vrrp_instance FED_FW {
    state MASTER
    interface eth0
    virtual_router_id 42
    priority 200
    advert_int 1
    smtp_alert
    authentication {
        auth_type PASS
        auth_pass f3san-fw
    }
    virtual_ipaddress {
        185.103.124.150/22 dev eth1
    }
    track_script {
       chk_apache
    }
    notify /usr/local/bin/keepalived.sh root
}
```

En el nodo de **BACKUP** la configuración difiere en las siguientes líneas:

- `router_id`: Pondremos el nombre del segundo nodo: ~~`F3SAN1-FW`~~ ➡ `F3SAN2-FW`
- `state`: El nodo es un nodo de respaldo: ~~`MASTER`~~ ➡ `BACKUP`
- `priority`: La prioridad será inferior a la del maestro: ~~`200`~~ ➡ `100`

```
router_id F3SAN2-FW
state BACKUP
priority 100
```

Adicionalmente, cuando el estado de un nodo se modifique, usando la siguiente directiva podemos hacer lo que queramos. En nuestro caso, simplemente el script deja un log en un fichero.

```
notify /usr/local/bin/keepalived.sh root
```

Esta hace que este script `/usr/local/bin/keepalived.sh` se ejecute con el usuario `root` cada vez que el estado del nodo cambia. La llamada se hace pasando 4 parámetros al script:
- <kbd>$1</kbd> - Indica el tipo de objeto sobre el que ocurre el evento. Generalmente tendrá el valor `INSTANCE` indicando que ha ocurrido en una instancia VRRP (i.e. el bloque `vrrp_instance` de la configuración)
- <kbd>$2</kbd> - Indica el nombre del objeto sobre el que ocurre el evento. Generalmente tendrá el valor `VI_1` ya que es el nombre definido para la instancia VRRP en la configuración.
- <kbd>$3</kbd> - Indica el estado al que transiciona la instancia. Por ejemplo `MASTER`, `BACKUP`, `STOP`.
- <kbd>$4</kbd> - Indica la prioridad de la instancia (el campo `priority` en la configuración de la instancia)

El contenido del script es:

```bash
#!/bin/sh

echo $(date) '|' $@ >> /tmp/keepalive.events

```

:::danger
Este fichero debe tener permisos de **ejecución**:
```
chown root:root /usr/local/bin/keepalived.sh
chmod 744 /usr/local/bin/keepalived.sh
```
:::

## Pruebas

Probar que el protocolo está funcionando como Dios manda es muy sencillo: Solo tenemos que detener el servicio `keepalived` en el nodo maestro y ver que el nodo backup levanta la IP de servicio y es capaz de recibir peticiones a la misma.

Al volver a levantar el servicio en el nodo maestro, este debe recuperar la IP de servicio y seguir operando como si nada.

```
systemctl [start|stop|restart] keepalived
```

También deberíamos poder ver en el log `/tmp/keepalive.events`. Por ejemplo:

Instancia 1 (MASTER)
```
Fri Aug 7 12:31:03 CEST 2020 | INSTANCE VI_1 STOP 200
Fri Aug 7 12:36:54 CEST 2020 | INSTANCE VI_1 BACKUP 200
Fri Aug 7 12:36:57 CEST 2020 | INSTANCE VI_1 MASTER 200
```

Instancia 2 (BACKUP)
```
Fri Aug 7 12:30:54 CEST 2020 | INSTANCE VI_1 MASTER 100
Fri Aug 7 12:36:57 CEST 2020 | INSTANCE VI_1 BACKUP 100
```




