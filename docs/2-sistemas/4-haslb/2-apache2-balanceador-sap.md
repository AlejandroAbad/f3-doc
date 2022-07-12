---
sidebar_position: 2
title: Balanceador SAP
---

# Apache2 como balanceador SAP
En las máquinas usadas como concentradores, configuraremos el servidor Apache para hacer de proxy-balanceador entre los concentradores Fedicom3 del servidor local y los distintos destinos SAP. Para esto, haremos que Apache escuche a peticiones locales y que las redirija al sistema SAP que le indiquemos.

Para lograr este comportamiento, usaremos un servidor virtual de Apache (*vhost*) que llamaremos `proxysap`, y que se accederá solo desde el servidor local. Este servidor virtual se accederá con el nombre de `proxysap`, nombre que ya habíamos configurado en el `/etc/hosts` del servidor durante la instalación del mismo. A modo de recordatorio, el fichero de hosts debe tener esta entrada:

```bash
# Nombre para el Proxy Balanceador a SAP (El servidor Apache local)
127.0.0.1       proxysap proxysap.hefame.es
```

Para configurar el servicio en Apache, crearemos el fichero `/etc/apache2/vhosts.d/proxysap.conf` con el contenido que iremos detallando.

## Configuración del VHOST

El servidor virtual está pensado sólo para escuchar peticiones desde el servidor local, por lo que no vemos necesario protegerlo con SSL. Por esto, el VHOST escuchará únicamente en el puerto 80. Comenzaremos por definir el servidor virtual tal que así:

```apacheconf
<VirtualHost _default_:80>
	ServerName proxysap.hefame.es

	ErrorLog /var/log/apache2/error_log
	TransferLog /var/log/apache2/access_log
```

### Establecimiento de cabeceras
Con esta directiva conseguimos que en las respuestas que el concentrador recibe de SAP aparezca el valor del campo `route` del servidor SAP elegido. El valor del campo `route` lo definiremos en el siguiente paso de modo que se indique el nombre del servidor SAP al que se lanzó la petición.

```apacheconf
Header set X-SERVIDOR-SAP "%{BALANCER_WORKER_ROUTE}e"
```

### Definición de los grupos de balanceo
Un grupo de balanceo permite definir una serie de servidores de destino donde se irán balanceando las peticiones que se envíen al grupo. Nos dá flexibilidad para definir, por ejemplo, cómo deben repartirse las peticiones entre los miembros del grupo, qué hacer en caso de caída de alguno de los miembros, el método para determinar si un servidor se considera vivo o muerto, etc...

Definimos un grupo de balanceo por sistema SAP al que queramos que este proxy pueda hacer peticiones. Las opciones de configuración del balanceador son un mundo aparte, pero en principio, para P01 usaremos los nodos pares de manera general y los impares solo en el caso de que no haya nodo pares disponibles (La opción `status=+H` indica que son *Hot-Standby*).

Adicionalmente, definiremos los parámetros de `HealthCheck` que definen cómo y cuando se realiza el control del estado de los servidores de backend.

```
hcmethod=GET 
hcuri=/sap/public/ping 
hcinterval=10
hcpasses=1 
hcfails=1 
connectiontimeout=5
```
Básicamente, las opciones que empiezan por `hc*` pertenecen al módulo `proxy_hcheck` y lo que le estamos indicando es que cada 10 segundos se realice una llamada `GET /sap/publi/ping` a cada servidor SAP. En caso de que una llamada falle (`hcfails = 1`) el servidor se asumirá caído hasta que unoa llamada de un resultado OK (`hcpasses = 1`). Se asume un valor OK siempre que SAP responda un código de estado 2xx o 3xx.

La opción `connectiontimeout` especifica el tiempo en segundos que el balanceador va a esperar durante el establecimiento de la conexión antes de considerar al miembro del grupo de balanceo como en error. No debe confundirse con la directiva `timeout`, que especifica el tiempo tras el cual la conexión se cierra.

La opción `retry` indica que, en el caso de que un miembro de un error, este será considerado en error durante 30 segundos. Transcurrido este tiempo, se volverá a intentar enviar peticiones por el mismo.

La configuración para P01 quedaría tal que así:

```apacheconf
Define ParametrosHC "hcmethod=GET hcuri=/sap/public/ping hcinterval=5 hcpasses=1 hcfails=1 connectiontimeout=5 retry=30"
Define ParametrosPS "lbmethod=bybusyness growth=8"

<Proxy balancer://P01>
	BalancerMember http://sap1p01:8011 route=sap1p01 ${ParametrosHC} status=+H
	BalancerMember http://sap2p01:8012 route=sap2p01 ${ParametrosHC} 
	BalancerMember http://sap3p01:8013 route=sap3p01 ${ParametrosHC} status=+H
	BalancerMember http://sap4p01:8014 route=sap4p01 ${ParametrosHC} 
	BalancerMember http://sap5p01:8015 route=sap5p01 ${ParametrosHC} status=+H
	BalancerMember http://sap6p01:8016 route=sap6p01 ${ParametrosHC} 
	BalancerMember http://sap7p01:8017 route=sap7p01 ${ParametrosHC} status=+H
	BalancerMember http://sap8p01:8018 route=sap8p01 ${ParametrosHC} 
	ProxySet ${ParametrosPS}
</Proxy>
```

De las opciones utilizadas, estas son las destacables:

- Indicando la opción `status=+H`, estamos indicando que el nodo es un *Hot Standby*, luego que solo se enviarán peticiones al mismo si el resto de nodos está caído. Podemos elegir en que modo arranca cada nodo con este parámetro
	- `D`: Worker is disabled and will not accept any requests.
	- `S`: Worker is administratively stopped.
	- `I`: Worker is in ignore-errors mode and will always be considered available.
	- `R`: Worker is a hot spare. For each worker in a given lbset that is unusable (draining, stopped, in error, etc.), a usable hot spare with the same lbset will be used in its place. Hot spares can help ensure that a specific number of workers are always available for use by a balancer.
	- `H`: Worker is in hot-standby mode and will only be used if no other viable workers or spares are available in the balancer set.
	- `E`: Worker is in an error state.
	- `N`: Worker is in drain mode and will only accept existing sticky sessions destined for itself and ignore all other requests.
- Adicionalemente, podemos usar la opción  `loadfactor` entre 1 y 100 para dar distinto peso a los servidores, en nuestro caso no nos interesa.
- Usamos `lbmethod=bybusyness` para indicar el algoritmo de balanceo que vamos a usar. Este creo que es el mas adecuado para nuestro caso, ya que mantiene la cuenta de peticiones activas de cada servidor y se las manda al que menos tenga.
- La opción `growth` de la directiva `ProxySet` nos permite añadir en caliente nuevos servidores al balanceo de carga.

:::tip
Información en detalle de estos parámetros:
- [Información completa de las opciones de la directiva BalancerMember](https://httpd.apache.org/docs/2.4/mod/mod_proxy.html#BalancerMember)
- [Información completa de los distintos métodos de balanceo de carga](https://httpd.apache.org/docs/2.4/mod/mod_proxy_balancer.html)
:::




### Mapeo de peticiones a los distintos sistemas SAP

Para indicarle a Apache a cual balanceador debe mandar cada petición (y por tanto que sistema SAP va a dirigirse), vamos a usar el prefijo en la URL. 

Para nuestro caso:
- Las peticiones que comiencen por `/p01` se enviarán al balanceador de P01.
- ~~Las peticiones que comiencen por `/t01` se enviarán al balanceador de T01.~~
- ~~Las peticiones que comiencen por `/d01` se enviarán al balanceador de D01.~~

```apacheconf
	ProxyRequests Off
  
	<Location "/p01">
		ProxyPass balancer://P01
	</Location>

	ProxyPass / !
</VirtualHost>
```

:::note
Nótese que no se pasa el prefijo `/p01` al dirigir la petición a SAP. 
Imaginemos el siguiente ejemplo:
1. El balanceador recibe una petición dirigida a la siguiente URL: `http://proxysap/p01/sap/public/ping`.
1. Como la URL comienza por `/p01`, el balanceador escogerá un miembro activo del balanceador `sapp01`, por ejemplo, `http://sap6p01:8016`.
1. El balanceador redirecciona la petición HTTP contra `http://sap6p01:8016/sap/public/ping`, eliminando el `/p01` en el proceso. 
:::



## Prueba de funcionamiento

Desde la consola de comandos del servidor podemos realizar una simple prueba de acceso al servicio SAP a través del balanceador. Para esto, haremos la siguiente petición HTTP con la herramienta `curl`:

```xml title="curl http://proxysap/p01/sap/public/info"
<?xml version="1.0" encoding="UTF-8"?>
<SOAP-ENV:Envelope xmlns:SOAP-ENV="http://schemas.xmlsoap.org/soap/envelope/">
  <SOAP-ENV:Body>
    <rfc:RFC_SYSTEM_INFO.Response xmlns:rfc="urn:sap-com:document:sap:rfc:functions">
      <RFCSI>
        <RFCPROTO>011</RFCPROTO>
        <RFCCHARTYP>1100</RFCCHARTYP>
        <RFCINTTYP>BIG</RFCINTTYP>
        <RFCFLOTYP>IE3</RFCFLOTYP>
        <RFCDEST>sap4p01_P01_14</RFCDEST>
        <RFCHOST>sap4p01</RFCHOST>
        <RFCSYSID>P01</RFCSYSID>
        <RFCDATABS>P01</RFCDATABS>
        <RFCDBHOST>orap01</RFCDBHOST>
        <RFCDBSYS>ORACLE</RFCDBSYS>
        <RFCSAPRL>740</RFCSAPRL>
        <RFCMACH>  324</RFCMACH>
        <RFCOPSYS>AIX</RFCOPSYS>
        <RFCTZONE>  3600</RFCTZONE>
        <RFCDAYST>X</RFCDAYST>
        <RFCIPADDR>10.1.1.4</RFCIPADDR>
        <RFCKERNRL>753</RFCKERNRL>
        <RFCHOST2>sap4p01</RFCHOST2>
        <RFCSI_RESV></RFCSI_RESV>
        <RFCIPV6ADDR>10.1.1.4</RFCIPV6ADDR>
      </RFCSI>
    </rfc:RFC_SYSTEM_INFO.Response>
  </SOAP-ENV:Body>
</SOAP-ENV:Envelope>
```

En el ejemplo anterior, si nos fijamos en la respuesta obtenida, vemos que se ha realizado la petición a la instancia SAP `sap4p01_P01_14`. Si repetimos la llamada, probablemente esta sea atendida por otro servidor de P01... ¿sí o qué?