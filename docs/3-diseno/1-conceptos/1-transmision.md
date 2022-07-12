---
sidebar_position: 1
title: Transmisión
---

# La transmisión
Dentro del diseño de la aplicación, llamaremos **TRANSMISION** a la entidad que se encarga de albergar y mantener toda la información relativa a una petición de un cliente, ya sea una petición para hacer un pedido, una devolución, una petición incorrecta, o cualquier otro tipo de peticiones de los clientes, y de mantener registro del resultado del procesamiento de la misma.

## Ciclo de vida
1. Se determina el `TIPO` de la tranmisión. Esto se hace en base al endpoint que llama el cliente. Por ejemplo, peticiones `POST /pedidos` son de tipo *CREAR PEDIDO*, o las peticiones `GET /albaran/:idAlbaran` son de tipo *CONSULTA ALBARAN*.

2. Cuando una petición de un cliente se recibe en la aplicación, esta genera inmediatamente una nueva **Transmision con un ID único** y con la información de la petición HTTP.

3. La información contenida hasta este momento en la transmisión se almacena en la base de datos. Esta característica es lo que nos brinda la funcionalidad de recuperar el estado inicial de la misma si fuera necesario, por ejemplo, ante un fallo del sistema.

4. Se lleva a cabo la acción asociada a la transmisión, según su tipo. Por ejemplo, crear el pedido, consultar el albarán. Una vez realizada la acción pertinente, se responde al cliente con los datos que sean procedentes. Por ejemplo, la información del pedido creado o del albarán consultado.

5. Una vez ha finalizado toda la acción, se guardan los resultados obtenidos de la propia ejecución y en la propia entidad de la transmisión y esta se actualiza en la base de datos. Esta información incluye valores como la respuesta HTTP dada al cliente, datos de la interacción con SAP (si ha existido), metadatos sobre la propia acción realizada, registros de log para depuración, etc ...

Por ver un ejemplo de los datos incluidos en una transmisión, aqui tendríamos el esquema de lo que sería un pedido:

```js
{
  "_id" : ObjectId("617177c3775a4865498d371f"),
  "fechaCreacion" : ISODate("2021-10-21T14:22:59.522Z"),
  "tipo" : 10,
  "v" : 20000,
  "estado" : 9900,
  "conexion" : {
    "metadatos" : { "ip" : "1.25.35.18" },
    "solicitud" : {...},
    "respuesta" : {...}
  },
  "pedido" : {
    "clienteSap" : "0010104999",
    "codigoAlmacenServicio" : "RG06",
    "codigoCliente" : 4999,
    "crc" : ObjectId("a727dca3ec78093ecf2397dc"),
    "pedidosAsociadosSap" : [ NumberLong(2057925991) ]
  },
  "sap" : {
    "metadatos" : {...},
    "solicitud" : {...},
    "respuesta" : {...}
  }
}
```

Donde podemos observar las siguientes propiedades:

- `_id` - **ID de transmisión** (en adelante `txId`). Es el identificador único en todo el sistema para cada transmisión. El formato del mismo es el de un [ObjectID](https://docs.mongodb.com/manual/reference/method/ObjectId/) (una cadena hexadecimal de 24 caracteres) 
- `fechaCreacion` - **Fecha de entrada de la transmisión**.
- `tipo` - **Tipo de transmisión**. Es un valor numérico que identifica la naturaleza de la transmisión. Por ejemplo, si la transmisión es para crear un pedido, consultar una factura, ... o si está descartada por no haber podido identificar el tipo.
- `v` - Indica la varsión del formato de la transmisión.
- `estado` - **Estado de la transmisión**. Es un valor numérico que indica el estado actual de procesamiento de la transmisión. Por ejemplo, _enviado a SAP_, _fallo de autenticación_, _esperando incidencias de SAP_, ...
- `conexion` - **Datos de conexión**. Incluye toda la inforamción referida con la transmisión HTTP, como la petición y respuesta, metadatos como la IP de origen, la autenticación, las opciones de TLS, balanceador/concentrador que atiende la petición ...
- `sap` - **Datos del intercambio de datos con SAP**. Similar a los datos de conexión, pero referido a la comunicación con SAP.
- `pedido` - **Datos específicos de la accion realizada**. En este caso, son datos del pedido creado, como el código de cliente, el almacén, números de pedido, etc... 
