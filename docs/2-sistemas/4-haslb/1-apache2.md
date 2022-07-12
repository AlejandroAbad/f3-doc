---
sidebar_position: 1
title: Apache2
---

# Instalación base de Apache2
Tanto en las máquinas usadas como cortafuegos como en los concentradores, instalaremos servidores **Apache2.4** para hacer la función de proxy inverso, funcionando como un balanceador de carga.

:::info
Vamos a instalar Apache con el MPM *(Multi Processing Module)* denominado `worker` ([Documentación oficial](https://httpd.apache.org/docs/2.4/mod/worker.html)). Este MPM permite que las peticiones de los clientes puedan atenderse dentro de hilos de un mismo proceso, reduciendo notablemente el número de procesos Apache necesarios, y por lo tanto, reduciendo enormemente el gasto de recursos, ya que los hilos de un mismo proceso comparten RAM y se reducen los saltos de contexto del procesador.
:::

A continuación describiremos el proceso de instalación y configuración de una instancia de Apache2 preparada para hacer de proxy/balanceador. En capítulos posteriores veremos como configurar estas instancias según la función que vayan a desempeñar.


## Instalación de Apache y módulos necesarios
```bash
zypper install apache2-worker
systemctl enable apache2
```


## Configuración general

### Despliegue de los certificados del servidor
Desplegaremos el certificado SSL que el servidor presentará a los clientes.

La estructura de directorios queda tal que así:
```
/etc/apache2/
    ssl.crt/
        ca-chain.crt
		server.crt
    ssl.key/
        server.key
```

```bash
chmod 600 /etc/apache2/ssl.key/server.key
```


### Configuración de módulos a cargar
Hay ciertos módulos que vienen activos por defecto en la instalación de Apache2 que no necesitaremos y vamos a desactivar para evitar que expongan cualquier tipo de vulnerabilidad del servidor. También, hay otros que deberemos activar para el balanceo de carga.

Para esto, haremos los siguientes cambios en el fichero `/etc/sysconfig/apache2`:

```javascript
APACHE_CONF_INCLUDE_FILES="/etc/apache2/mod_proxy_balancer.conf"
APACHE_MODULES="alias authz_host authz_core env include log_config mime setenvif ssl reqtimeout authn_core rewrite proxy proxy_http status proxy_balancer mod_slotmem_shm lbmethod_bybusyness headers watchdog proxy_hcheck"
APACHE_SERVER_FLAGS="SSL STATUS"
APACHE_MPM="worker"
APACHE_SERVERNAME="<<hostname>>"
```

:::caution
El módulo `watchdog` debe ir antes que el módulo `proxy_hcheck` para que estos módulos funcionen correctamente.
:::

:::note
El fichero `/etc/apache2/mod_proxy_balancer.conf` lo crearemos mas adelante.
:::

Como hemos excluido módulos de la lista de carga del servicio, comentaremos las siguientes líneas en `/etc/apache2/httpd.conf` para que no haya errores:

```
#Include /etc/apache2/mod_info.conf
#Include /etc/apache2/mod_cgid-timeout.conf
#Include /etc/apache2/mod_usertrack.conf
#Include /etc/apache2/mod_autoindex-defaults.conf
#TypesConfig /etc/apache2/mime.types
#Include /etc/apache2/mod_mime-defaults.conf
#DirectoryIndex index.html index.html.var
#Include /etc/apache2/default-server.conf
```


### Panel de control del balanceador
Configuraremos un *handler* para que el módulo de balanceo de carga de Apache pueda gestionarse a través de la página en la ruta `/balancer-manager`. También haremos un control de acceso por IP de manera que solo las peticiones de las redes internas puedan acceder a la misma.
Para esto, creamos el fichero `/etc/apache2/mod_proxy_balancer.conf` con el siguiente contenido:

```apacheconf title="/etc/apache2/mod_proxy_balancer.conf"
<IfModule mod_proxy_balancer.c>
    <Location "/balancer-manager">
        SetHandler balancer-manager
        Require ip 192.168.10.0/24 172.30.0.0/16
        Require local
    </Location>
</IfModule>
```


### Directivas `mod_access_compat`

Por último, un paso opcional que me gusta hacer para dejar la configuración de Apache lo mas limpia posible, es eliminar las directivas `mod_access_compat`, que son para dar compatibilidad con la antigua nomenclatura del *mod_authz* de Apache2.2 y por tanto no necesitamos

Para esto, donde veamos el siguiente patrón:

```apacheconf
<IfModule !mod_access_compat.c>
    Require local
</IfModule>
<IfModule mod_access_compat.c>
    Order deny,allow
    Deny from all
    Allow from localhost
</IfModule>
```

Podemos eliminarlo todo y dejar solo lo que está entre `<IfModule !mod_access_compat.c>`, que en el ejemplo anterior sería:

```apacheconf
Require local
```

Así queda mas claro que lo que tiene que hacer ¿No crees?


### Configuración de logrotate

Vamos a configurar la rotación de logs de Apache con la utilidad `logrotate` de SUSE.

Comenzamos por editar el fichero de configuración de logrotate para apache `/etc/logrotate.d/apache2` tal que:

```bash title="/etc/logrotate.d/apache2"
/var/log/apache2/*log {
    copytruncate
    compress
    dateext
    maxage 30
    rotate 99
    size=+1024k
    notifempty
    missingok
    create 644 root root
}
```

Añadimos la siguiente línea en el crontab del usuario <kbd>root</kbd>:

```bash
# Rotación de logs de Apache2
00 04 * * * root /usr/sbin/logrotate /etc/logrotate.conf >/dev/null 2>/dev/null
```



## Configuración específica
Según la función que vaya a cumplir el servicio Apache, lo terminaremos de configurar específicamente:

- [Configurar Apache2 para un **nodo concentrador**](/docs/sistemas/haslb/apache2-concentrador).
- [Configurar Apache2 como **balanceador a SAP**](/docs/sistemas/haslb/apache2-balanceador-sap).