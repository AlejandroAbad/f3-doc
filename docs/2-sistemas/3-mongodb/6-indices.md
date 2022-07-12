---
sidebar_position: 6
title: \@Índices
---

# Índices

:::danger
**Este documento es OBSOLETO ya que solo aplica a la versión 1.x del concentrador.**
:::

Vamos a comentar los distintos índices que se crean sobre la colección `tx` que es la colección donde se almacenarán las transmisiones de los clientes.

:::tip
[Documentación oficial sobre índices en MongoDB](https://docs.mongodb.com/manual/indexes/)
:::

En la aplicación NodeJS, las consultas de transmisiones están definidas en el fichero `interfaces/imongo/iMongoConsultaTx.js`.

# Consultas por ID
Las consultas por ID de transmisión utilizan el campo `_id` como único filtro.
Por defecto, MongoDB crea un índice por el campo `_id`, por lo tanto, este ya lo tenemos.

# Consultas por CRC
Como veremos, internamente se almacena el CRC de las transmisiones de dos maneras: en el campo `crc` como un ObjectID y en el campo `sapCrc` que es un integer.

```json
db.tx.createIndex( {
	crc: 1
}, {
	name: "crc",
	partialFilterExpression: { crc: {$exists: true} }
})
```

```json
db.tx.createIndex( {
	crcSap: 1
}, {
	name: "crcSap",
	partialFilterExpression: { crcSap: {$exists: true} }
})
```


# Consultas por fecha
```json
db.tx.createIndex( {
    createdAt: -1
}, {
    name: "createdAt"
})
```

# Consultas por tipo de transmisión
```json
db.tx.createIndex( {
    type: 1
}, {
    name: "type"
})
```

# Consultas por estado de transmisión
```json
db.tx.createIndex( {
    status: 1
}, {
    name: "status"
})
```

# Consultas por código de artículo
```json
db.tx.createIndex( {
    "clientRequest.body.lineas.codigoArticulo": 1
}, {
    name: "codigoArticulo",
    partialFilterExpression: { type: 10 }
})
```

# Consultas por FLAGS
```json
db.tx.createIndex( {
    "flags.$**": 1
}, {
    name: "flags"
})
```