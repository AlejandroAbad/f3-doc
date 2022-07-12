---
sidebar_position: 2
title: Instalación de MongoDB
---

# Instalación de MongoDB
Vamos a instalar mongo enterprise en la versión 4.4 (la última a 31 de marzo de 2021)

:::tip
Para obterner información adicional, puedes seguir [Enlace al manual oficial de instalación en SUSE Linux](https://docs.mongodb.com/manual/tutorial/install-mongodb-enterprise-on-suse/).
:::

## Instalación con zypper

Para tener acceso a los binarios de MongoDB 4.4, debemos instalar los repositorios para esta versión:

```bash
rpm --import https://www.mongodb.org/static/pgp/server-4.4.asc
zypper addrepo --gpgcheck "https://repo.mongodb.com/zypper/suse/12/mongodb-enterprise/4.4/x86_64/" mongodb-4.4
```

Ya podemos instalar el servicio de mongod con zypper.

```
# zypper install mongodb-enterprise
( . . . )
The following 5 NEW packages are going to be installed:
  mongodb-enterprise mongodb-enterprise-mongos mongodb-enterprise-server mongodb-enterprise-shell mongodb-enterprise-tools
```

Una vez instalado, lo configuramos para que el servicio arranque con el servidor:

```
chkconfig mongod on
systemctl enable mongod
```

Y ya podemos arrancar y parar la base de datos con systemctl

```bash
systemctl start/stop/status mongod
```

Podemos comprobar que el servicio está arrancado en el log `/var/log/mongodb/mongod.log`. 
Únicamente deberemos ver un WARNING, indicando que el control de acceso esta deshabilitado, lo cual es normal en una instalación desde cero como la que acabamos de hacer:

```bash
I CONTROL  [initandlisten]
I CONTROL  [initandlisten] ** WARNING: Access control is not enabled for the database.
I CONTROL  [initandlisten] **          Read and write access to data and configuration is unrestricted.
I CONTROL  [initandlisten]
I NETWORK  [thread1] waiting for connections on port 27017
```


## Tareas post-instalacion

### Creación de clave de interconexión

:::caution
Esta tarea solo debemos ejecutarla una vez en un nodo del _ReplicaSet. Si ya hemos generado el fichero con la clave en otra máquina, simplemente tenemos que copiar el fichero `/var/lib/mongo/mongo.key` del servidor donde se generó en este servidor **respetando los permisos y propietario del fichero**.
:::

Por defecto, la instancia MongoDB no cifra las claves de los usuario. Al crear el fichero `/var/lib/mongo/mongo.key` con una clave de cifrado, la instancia la usará automáticamente para almacenar los datos de autenticación de los usuarios de la base de datos y para securizar la comunicación entre las distintas réplicas de la base de datos. Para generar el fichero `/var/lib/mongo/mongo.key` con una clave de 1024 bits, haremos lo siguiente:

```bash
openssl rand -base64 756 > /var/lib/mongo/mongo.key
chmod 400 /var/lib/mongo/mongo.key
chown mongod:mongod /var/lib/mongo/mongo.key
```


### Configuración del servicio

Editamos el fichero de configuración `/etc/mongod.conf` para realizar los siguientes cambios:

1. Que el servicio escuche en todas las interfaces, cambiando el parámetro `net.bindIp` para que escuche en `0.0.0.0`.
1. Indicaremos a la instancia de Mongo que use el fichero `/var/lib/mongo/mongo.key` estableciendo el parámetro `security.keyFile`.

El fichero queda tal que:

```yaml title="/etc/mongod.conf"

systemLog:
  destination: file
  logAppend: true
  path: /var/log/mongodb/mongod.log

storage:
  dbPath: /var/lib/mongo
  journal:
    enabled: true

processManagement:
  fork: true  # fork and run in background
  pidFilePath: /var/run/mongodb/mongod.pid  # location of pidfile
  timeZoneInfo: /usr/share/zoneinfo

net:
  port: 27017
  bindIp: 0.0.0.0

security:
  keyFile: /var/lib/mongo/mongo.key

replication:
  replSetName: "fedicom3"
```

### Creación del usuario admin

Creamos un usuario `admin` en la base de datos del mismo nombre, con rol de administrador. Para conectarnos a la consola de mongodb, usamos el comando `mongo`.

```
# mongo
use admin
db.createUser( {
	user: "admin",
	pwd: "password",
	roles: [ { role: "root", db: "admin" } ]
});
```

- Una vez creado el primer usuario, se considera que todo acceso a la base de datos debe autenticarse, por lo que a partir de este punto, para poder realizar acciones de administración deberemos autenticarnos. 
- Cada usuarios va ligado a una base de datos, por lo que para autenticarnos con los mismos, deberemos estar en dicha base de datos. Usamos el comando `use <db>` para cambiar de una base de datos a otra.
- Como este usuario tiene el rol `root` de la base de datos `admin`, se considera un superusuario.

Para autenticarnos con el usuario `admin` de la base de datos `admin` haremos lo siguiente:

```bash
> use admin
> db.auth("admin", "password")
```

### Rotación de logs de MongoDB
MongoDB va dejando un log en `/var/log/mongodb/mongod.log`. Este log va llenandose hasta el infinito si no se controla.

Una de las maneras que tiene MongoDB para indicarle al proceso de que debe rotar el log es mandándole la señal <kbd>SIGUSR1</kbd> al mismo.
Esto hace que el antiguo fichero `mongod.log` se renombre con la forma `mongod.log.yyyy-mm-ssThh:MM:ss`, y se genere un nuevo `mongod.log` vacío.
La documentación detallada de la rotación de logs en MongoDB se explica [aquí](https://docs.mongodb.com/manual/tutorial/rotate-log-files/).

Para lograr la rotación diaria de los logs, hemos creado un script en `/usr/local/bin/mongod-logrotate` que provoca la rotación de log
en el proceso mongod, lo comprime y elimina los anteriores a 30 días.

```bash
#!/bin/sh

PIDFILE=/var/run/mongodb/mongod.pid
LOGDIR=/var/log/mongodb

# Indicamos al proceso de mongod que realize la rotacion del log
kill -10 $(cat $PIDFILE) 2>/dev/null

# Comprimimos todos los ficheros de log no comprimidos, excluyendo el actual "mongod.log"
find $LOGDIR -name "mongod.log.*" -and ! -name "*.gz" -exec gzip {} \;

# Eliminamos los anteriores a 30 dias
find $LOGDIR -name "mongod.log.*.gz" -mtime +30 -exec rm -f {} \;
```

Este script se ejecutará a diario en cada instancia de MongoDB a las 4 de la madrugada mediante el `crontab` del usuario `root`.

```bash
# Rotación de logs de MongoDB
00 04 * * * /usr/local/bin/mongod-logrotate >/dev/null 2>/dev/null
```




## Siguientes pasos:
- [Inclusión del nodo en el RéplicaSet](/docs/sistemas/mongodb/replicaset)
- [Creación de usuarios](/docs/sistemas/mongodb/usuarios)
- [Creación de índices](/docs/sistemas/mongodb/indices)