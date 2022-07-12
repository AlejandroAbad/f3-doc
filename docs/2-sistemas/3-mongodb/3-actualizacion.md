---
sidebar_position: 3
title: Actualización de MongoDB
---

# Actualización de MongoDB
Generalmente, el proceso de actualización de las instancias MongoDB es siempre el mismo. En este documento vamos a describir el proceso de actualización de los nodos de MongoDB de la versión 4.2 a la última disponible, la 4.4.

> **IMPORTANTE**. Revisa las guías oficiales de actualización están disponibles en los siguientes enlaces:
> - [Servidores Standalone](https://docs.mongodb.com/manual/release-notes/4.4-upgrade-standalone/)
> - [Servidores en ReplicaSet](https://docs.mongodb.com/manual/release-notes/4.4-upgrade-replica-set/)


# Comprobaciones previas de compatibilidad

Asumimos que la version antigua es la 4.2 y corriendo en todos los nodos del _ReplicaSet_. 
Si esto es así, __el cambio de 4.2 a 4.4 es directo__, de lo contrario, debemos buscar información al respecto.

```
# mongo
MongoDB shell version: 4.2.12
MongoDB server version: 4.2.12
```

Comprobamos la versión de características de la base de datos.
Para esto, ejecutamos el siguiente comando en el mongo shell:
```javascript
> db.adminCommand( { getParameter: 1, featureCompatibilityVersion: 1 } )
{ "featureCompatibilityVersion" : { "version" : "4.2" } }
```

Si da un valor inferior a `4.2` deberemos cambiarlo para dejarlo a la versión correcta. Para cambiarlo ejecutaremos el siguiente comando:

```javascript
> db.adminCommand( { setFeatureCompatibilityVersion: "4.2" } )
```

:::tip
La lista de cambios de características de la 4.2 está disponible en [este enlace](https://docs.mongodb.com/manual/release-notes/4.2-compatibility).
:::

# Configuración de repositorios
También asumimos estará disponible el repositorio de la versión antigua. Deberemos eliminar el repositorio antiguo para evitar conflictos durante la actualización:

```
# zypper repos | grep mongo
8 | mongodb-4.2                                                          | mongodb-4.2

# zypper removerepo mongodb-4.2
```

Instalaremos el repositorio de la versión 4.4 de la siguiente manera:

```
# rpm --import https://www.mongodb.org/static/pgp/server-4.4.asc
# zypper addrepo --gpgcheck "https://repo.mongodb.com/zypper/suse/12/mongodb-enterprise/4.4/x86_64/" mongodb-4.4
Adding repository 'mongodb-4.4' ..............................................................[done]
Repository 'mongodb-4.4' successfully added

URI         : https://repo.mongodb.com/zypper/suse/12/mongodb-enterprise/4.4/x86_64/
Enabled     : Yes
GPG Check   : Yes
Autorefresh : No
Priority    : 99 (default priority)

Repository priorities are without effect. All enabled repositories share the same priority.
```


# Actualización de los binarios

:::danger
**IMPORTANTE**: Actualizaremos primero todos los nodos que **NO SON PRIMARIOS**.
:::

```
# zypper ref
# zypper update mongodb-enterprise
```

Y tras la actualización, el nodo ya estará corriendo en la versión actualizada:

```
# mongo
MongoDB shell version v4.4.3
MongoDB server version: 4.4.3
```

Cuando ya estén actualizados todos los nodos menos el primario, forzaremos la elección de un nuevo nodo primario (convirtiendo el actual nodo primario en un nodo secundario) y acto seguido procederemos a actualizarlo al igual que el resto.

Para forzar la reelección, ejecturaremos en el mongo shell:

```javascript
MongoDB Enterprise fedicom3:PRIMARY> rs.stepDown(180, 10)
MongoDB Enterprise fedicom3:SECONDARY> // ¡ El nodo ya no es primario ! 
```

:::tip
Información adicional del comando [rs.stepDown()](https://docs.mongodb.com/manual/reference/method/rs.stepDown/)
:::

# Activación de características 4.4
En este momento, estaremos ejecutando los binarios de la versión 4.4, pero con las características de la 4.2.
Cuando estemos seguros de que los binarios en versión 4.4 están funcionando correctamente, podemos avanzar el `featureCompatibilityVersion` a la nueva versión.

```javascript
> db.adminCommand( { setFeatureCompatibilityVersion: "4.4" } )
```

:::caution
Al realizar esta acción, ya no hay rollback posible de los binarios. Todos los ejecutables de los nodos del clúster deberán estar en versión 4.4 o superior.
:::

La lista de cambios de características de la 4.4 está disponible en [este enlace](https://docs.mongodb.com/manual/release-notes/4.4-compatibility).

