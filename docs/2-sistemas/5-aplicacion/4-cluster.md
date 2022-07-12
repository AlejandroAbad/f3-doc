---
sidebar_position: 4
title: Clúster
---

# Configuración del clúster
La configuración del clúster es aquella configuración que es común a todas las instancias Fedicom 3 del clúster. Para mantener esta configuración sincronizada entre las instancias, esta se almacena en la base de datos MongoDB, concretamente en la colección llamada `configuracion`.

Dentro de esta colección se encuentran una serie de documentos, cuya clave es el campo `_id`, y cada cual hace referencia a un aspecto de la configuración disntinto. Por ejemplo, para la configuración de los destinos SAP, el documento de la colección es el que tiene `_id: 'sap'`, el de la configuración de LDAP sería `_id: 'ldap'`, etc ...

A continuación, detallamos cada documento:

## JWT 
Especifica la configuración de cómo se generan y validan los tokens JWT con los que trabaja la aplicación. 

```js
{
    _id : "jwt",
    clave : "UnChurracoDeClaveQueLoFlipas",
    ttl : 3600,
    tiempoDeGracia : 60
}
```

| Parámetro                     | Obligatorio   | Defecto | Descripción        |
| ------------------            |:-:| :------- | ------------------ |
| `clave`                  | no            | `null`        | La clave de cifrado de los JWT que se emitan por la aplicación. |
| `ttl`                     | no            | `3600`     | Tiempo de validez de los tokens generados, en **segundos**. |
| `tiempoDeGracia`                      | no            | `60`     | Un token caducado por menos de este valor, en **segundos**, seguirá siendo aceptado. |


## Destino SAP

```js
{
  _id: "sap",
  destino: {
    servidor: "proxysap-dev",
    puerto: 80,
    https: false,
    usuario: "fedicom3",
    password: "chachiPass",
    prefijo: "/t01"
  },
  timeout : {
    verificarCredenciales: 5,
    realizarPedido: 30,
    realizarLogistica: 30,
    realizarDevolucion: 15,
    consultaDevolucionPDF: 10,
    consultaAlbaranJSON: 10,
    consultaAlbaranPDF: 10,
    listadoAlbaranes: 30
  }
}
```


| Parámetro | Obligatorio | Defecto | Descripción |
| - | - | - | - |
| `destino.servidor` | si | | Nombre del host del sistema SAP. |
| `destino.puerto` | si | | Puerto de acceso HTTP al sistema SAP. |
| `destino.https` | no | `false` | `true` si se utiliza HTTPS, `false` de lo contrario. |
| `destino.usuario` | si | | Usuario a utilizar para la autenticación contra el sistema SAP. |
| `destino.password` | si | | Contraseña para autenticación contra el sistema SAP. |
| `destino.prefijo` | no | `""` | Prefijo a incluir delante de todas las peticiones HTTP que se dirijan a este sistema SAP. Por ejemplo `/t01`. |
| `timeout` | no | | Es un objeto donde se especifican los diferentes tiempos de timeout para las llamadas de la aplicación Fedicom3 a SAP en **segundos**. Las opciones disponibles y sus valores por defecto son los que pueden verse en el ejemplo de configuración. |

## LDAP (Active Directory) 

Especifica la configuración de cómo se podrán autenticar los usuarios con credenciales de Active Directory. 

```json
{
  _id : "ldap",
  servidor : "ldaps://activedirectory.hefame.es",
  baseBusqueda : "DC=hefame,DC=es",
  prefijoGrupos : "FED3_",
  certificados : [
    "-----BEGIN CERTIFICATE-----\r\nMIIDhDCCAmygAwIBAgIQU62pA4nkZZ5OL4pSSinXV ... dAEM0+jQ==\r\n-----END CERTIFICATE-----",
    "-----BEGIN CERTIFICATE-----\r\nMIIF6jCCBNKgAwIBAgITHQAAADqPmtePOmjLZgAAA ... nkZZ5OL4==\r\n-----END CERTIFICATE-----"
  ]
}
```

| Parámetro | Obligatorio | Defecto | Descripción |
| - |:-:| - | - |
| `servidor` | si | | La ruta al servidor LDAP. |
| `baseBusqueda` | si | | El FQDN base para la búsqueda de usuarios. Por ejemplo `DC=hefame,DC=es`. |
| `prefijoGrupos` | no | `FED3_` | Los grupos del usuario del Active Directory que comiencen con este prefijo son los que se tendrán en cuenta por la aplicación. |
| `certificados` | no | `[]` | Lista de certificados de CA para validar el certificado del servidor Active Directory **(\*)**. Solo se aplica si el protocolo del AD es `ldaps`. |

:::caution
**(\*)** Nótese que al almacenar el certificado:
1. Debe ir codificado en **BASE64**.
2. deben incluirse los tags `-----BEGIN CERTIFICATE-----` y `-----END CERTIFICATE-----` al inicio y fin del mismo
3. Los retornos de carro deben ir marcados como `\r\n`.
:::


## Pedidos 

Configura ciertos aspectos del tratamiento de las transmisiones de creación de pedidos de los clientes

```js
{
  _id : "pedidos",
  umbralLineasCrc : 10,
  antiguedadDuplicadosMaxima : 10080,
  antiguedadDuplicadosPorLineas : 180,
  tipificadoFaltas : {
    BAJA : "desconocido",
    BAJA HEFAME : "desconocido",
    ...
    SERVICIO PARCIAL : "suministro",
    SIN UNIDADES PTES : "suministro"
  }
}
```

| Parámetro | Obligatorio | Defecto | Descripción |
| - |:-:| - | - |
| `umbralLineasCrc`                  | no            | `10`        | Indica el número de líneas mínimo que una transmisión de pedido debe tener para generar el CRC en base a las líneas. Si la transmisión lleva un número de líneas inferior a este valor, el CRC se genera usando el campo `numeroPedidoOrigen` de la misma. |
| `antiguedadDuplicadosMaxima` | no | `10080` | A la hora de determinar si una transmisión es un duplicado de otra, entre estas debe haber una diferencia de tiempo igual o inferior a este valor, en **minutos**. Este parámetro afecta a aquellos pedidos cuyo CRC se calcula en base al campo `numeroPedidoOrigen`. |
| `antiguedadDuplicadosPorLineas` | no | `180` | A la hora de determinar si una transmisión es un duplicado de otra, entre estas debe haber una diferencia de tiempo igual o inferior a este valor, en **minutos**. Este parámetro afecta a aquellos pedidos cuyo CRC se calcula en función de las líneas del mismo. |
| `tipificadoFaltas` | no | `{}` | Permite definir un mapa clave-valor para "simplificar" el motivo de las faltas. La clave debe indicar el motivo de falta indicado por SAP y el valor el motivo Fedicom. Este mapeo se hace para hacer estadísticas de faltas, y trabajamos con 5 motivos que explicamos bajo esta tabla. |

:::info
**Tipificación de faltas:**
- **desconocido**: Agrupa todos los motivos de falta que indican en definitiva que el artículo solicitado no existe en el catálogo. P.e: `BAJA` o `DESCONOCIDO`.
- **stock**: Agrupa todos los motivos de falta que indican que el artículo solicitado no se puede servir por falta de stock. P.e: `SIN EXISTENCIAS`.
- **noPermitido**: Agrupa todos los motivos de falta que indican que el artículo solicitado no se puede servir al cliente de esta manera. P.e: `POR ENCARGO` o `POR OPERADOR/WEB`.
- **suministro**: Agrupa todos los motivos de falta que indican que el artículo solicitado no se puede servir por falta o limitaciones de suministro. P.e: `EXCESO UNIDADES POR LINEA` o `SIN UNIDADES PTES`.
- **estupe**: Agrupa todos los motivos de falta que indican que el artículo solicitado no se puede servir por ser estupefaciente. P.e: `NUMERO VALE INCORRECTO` o `ESTUPEFACIENTE`.
:::


## Devoluciones 
Configura ciertos aspectos del tratamiento de las transmisiones de creación de devoluciones de los clientes.

```js
{
  _id : "devoluciones",
  motivos : {
    "01" : "Caducidad del producto",
    "02" : "Retirado por alerta sanitaria",
    ...
    "10" : "Otros",
    "11" : "Estupefaciente caducado"
  },
  motivosExtentosAlbaran : [ "01", "02", "11" ]
}
```

| Parámetro | Obligatorio | Defecto | Descripción |
| - |:-:| - | - |
| `motivos` | si | | Define un mapa clave-valor donde se definen los motivos de devolución válidos definidos en la norma Fedicom3 y su descripción asociada. Las transmisiones de devolución que se realicen deben incluir un motivo válido de esta lista. |
| `motivosExtentosAlbaran` | no | `[]` | Indica una lista de los motivos de devolución que están exentos de indicar el número y fecha del albarán y por tanto no serán tomadas como incorrectas a la hora de procesar el mismo. Los motivos exentos se definen en la norma Fedicom3. |


## Logística 

Configura ciertos aspectos del tratamiento de las transmisiones de creación de pedidos logísticos de los clientes.

```js
{
  _id : "logistica",
  tiposAdmitidos : {
    I : "inversa",
    D : "directa",
    C : "cross",
  }
}
```

| Parámetro | Obligatorio | Defecto | Descripción |
| - |:-:| - | - |
| `tiposAdmitidos` | no | `{}` | Indica la lista de los posibles tipos de logística que admite el concentrador. El tipo de logística que debe indicar el cliente debe coincidir con alguna de las claves de este objeto |



## Watchdog de Pedidos
Define el comportamiento de los procesos `watchdogPedidos`.

```js
{
  _id : "watchdogPedidos",
  servidor : "f3dev2",
  intervalo : 20,
  antiguedadMinima : 30,
  transmisionesSimultaneas : 10,
  numeroPingsSap : 3,
  intervaloPingsSap : 5,
  maximoReintentos : 10
}
```

| Parámetro | Obligatorio | Defecto | Descripción |
| -| - | - | - |
| `servidor` | no | `""` | Indica el nombre del host que ejecutará la función de watchdog de pedidos. Si el nombre del host no coincide con este valor, el proceso `watchdogPedidos` no hará nada. Este valor es para garantizar que solo un nodo ejecuta las funciones del `watchdogPedidos`. |
| `intervalo` | no | `5` | Intervalo, en **segundos**, entre escaneos de la base de datos. |
| `antiguedadMinima` | no | `600` | Tiempo, en **segudos**, tras el cual una transmisión no confirmada se considerará en error. |
| `transmisionesSimultaneas` | no | `10` | Número de transmisiones que el proceso va a tratar simultáneamente como máximo. |
| `numeroPingsSap` | no | `3` | Para garantizar la disponibilidad de SAP, este debe responder a este número de *pings* sucesivos. |
| `intervaloPingsSap` | no | `5` | Intervalo, en **segundos**, entre los distintos *pings* realizados a SAP para comprobar su disponibilidad. |
| `maximoReintentos` | no | `5` | Indica el número de veces que se va a intentar recuperar una transmisión que se encuentre en error. |



## Watchdog de SQLite
Define el comportamiento de los procesos `watchdogSqlite`.

```js
{
  _id : "watchdogSqlite",
  maximoReintentos : 10,
  insercionesSimultaneas : 10,
  intervalo : 10
}
```

| Parámetro | Obligatorio | Defecto | Descripción |
| -| - | - | - |
| `maximoReintentos` | no | `10` | Indica el número de veces que se va a intentar recuperar cada entrada que se encuentre en SQLite. |
| `transmisionesSimultaneas` | no | `10` | Número máximo de transmisiones que el proceso `watchdogSqlite` va a tratar de escribir a MongoDB simultáneamente. |
| `intervalo` | no | `10` | Intervalo, en **segundos**, entre escaneos de la base de datos SQLite. |


## Balanaceadores Apache

Este documento permite definir la topología de balanceadores de carga Apache que haya en el sistema. En concreto, permite definir una lista de `balanceadores` donde se indica la configuración y modo de acceso de cada balanceador en un objeto de tipo `Balanceador` que definimos a continuación.

```js
{
  _id : "balanceador",
  balanceadores : [ 
    {
      nombre : "f3dev1-fw",
      base : "https://f3dev1-fw.hefame.es",
      proxy : "f3dev1",
      tipo : "fedicom"
    }, 
    {
      nombre : "f3dev1",
      base : "http://f3dev1.hefame.es",
      proxy : null,
      tipo : "sap"
    },
	{ Balanceador }
  ]
}
```

### Balanceador
Objeto que representa un balanceador de carga concreto.

```js
{
  nombre : "f3dev2-fw",
  base : "https://f3dev2-fw.hefame.es",
  proxy : "f3dev2",
  tipo : "fedicom"
}
```

| Parámetro | Obligatorio | Defecto | Descripción |
| - |:-:| - | - |
| `nombre` | si | | Define el nombre identificativo del balanceador. Debe ser único en el sistema. |
| `base` | si | | Determina el prefijo para construir las URLs de acceso HTTP al balanceador. Por ejemplo: `https://f3dev2-fw.hefame.es`. Aquí puede definirse si se usa HTTP o HTTPS. |
| `proxy` | no | `null` | Indica si el balanceador es accesible directamente (si valor es `null`), o este debe accederse a través de otra instancia Fedicom que hace de proxy. En este último caso se indicaría el nombre del servidor que hace de proxy. |
| `tipo` | si | | Debe ser `fedicom` si el balanceador es para el balanceo de peticiones fedicom de los clientes, o `sap` si es para el balanceo de las peticiones contra SAP. |
