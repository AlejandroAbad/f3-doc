---
sidebar_position: 5
title: Usuarios y roles
---

# Usuarios y roles


## Bases de datos
Aunque el objetivo principal de este capítulo es explicar los usuarios y los roles, vamos a explicar primero el concepto de base de datos en MongoDB. Las bases de datos son los contenedores principales de las instancias `mongod`. Podrían verse como que son "tenants", cada una tiene sus usuarios, roles y datos (en lo que se conoce como colección).

Inicialmente veremos que solo existen las bases de datos `admin`, `config` y `local`.

```javascript
fedicom3:PRIMARY> show dbs
admin   0.000GB
config  0.000GB
local   0.000GB
```

Con el comando `use <db>` podemos cambiar de una base de datos a otra.
Para crear una base de datos no existe ningún comando explícito, simplemente, por el hecho de crear una colección o usuario dentro de una base de datos, esta se crea. Así por ejemplo, crearemos la colección `transmisiones` en la base de datos `fedicom3`.

```javascript
fedicom3:PRIMARY> use fedicom3
switched to db fedicom3
fedicom3:PRIMARY> db.createCollection('transmisiones')
```

:::tip
De hecho, ni siquiera tenemos que crear una colección. Al insertar datos sobre un nombre de colección que no existe, esta se crea con la configuración por defecto.
:::


Una de las cosas que podemos hacer con MongoDB es crear una colección "capada" que se limita a un máximo de tamaño (por ejemplo, 1Gb o 1000 registros). 
Esta colección elimina las entradas mas antiguas para no superar nunca la restricción que se le haya puesto.
(Mas info: [https://docs.mongodb.com/manual/core/capped-collections/](https://docs.mongodb.com/manual/core/capped-collections/)).

```javascript
fedicom3:PRIMARY> db.createCollection('logs', { capped : true, size : 1073741824 });
```


## Usuarios
Para permitir el acceso desde la aplicación, vamos a crear el usuario `fedicom3` con plenos permisos sobre la base de datos `fedicom3`, y para que el mismo usuario pueda acceder a datos básicos de monitorización del `ReplicaSet`, le daremos el rol `clusterMonitor` sobre la base de datos `admin`.

```javascript
> use fedicom3
> db.createUser({
  user: "fedicom3",
  pwd: "password",
  roles: [
    { role: "readWrite", db: "fedicom3" }
    { role: "clusterMonitor", db: "admin" }
  ]
})
```

Para acceso de monotirización de solo lectura, vamos a crear el usuario `monitor` con el rol `read` sobre la base de datos `fedicom3`:

```javascript
> use fedicom3
> db.createUser({
  user: "monitor",
  pwd: "password",
  roles: [
    { role: "read", db: "fedicom3" }
  ]
})
```

## Roles

:::note
Por el momento no usamos roles personalizados en la aplicación. Solo usaremos [roles predefinidos en Mongo](https://docs.mongodb.com/manual/reference/built-in-roles/).
:::

Si quisieramos crear usuarios con permisos especificos, deberemos usar roles. 
Por el momento no crearemos ningún rol para la aplicación Fedicom3, pero dejamos esta información por si fuera conveniente en el futuro.

A continuación mostramos un ejemplo de un rol que solo puede ejecutar la operación `find` sobre la colección `transmisiones` y las operaciones `find` y `insert` sobre la colección `logs` de la base de datos `fedicom3`:

```javascript
> db.createRole({
  role: "soloLectura",
  privileges: [
    {
      resource: { db: "fedicom3", collection: "transmisiones" },
      actions: [ "find" ]
    },
    {
      resource: { db: "fedicom3", collection: "logs" },
      actions: [ "find" ]
    }
  ],
  roles: [ ]
})

> db.createUser({
  user: "usuarioVisor",
  pwd: "12345678",
  roles: [
    { role: "soloLectura", db: "fedicom3" }
  ]
})
```
