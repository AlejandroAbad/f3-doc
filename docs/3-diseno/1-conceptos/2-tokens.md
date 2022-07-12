---
sidebar_position: 2
title: Tokens
---

# Autenticación basada en tokens
El protocolo Fedicom v3 utiliza autenticación basada en tokens, en concreto, utiliza tokens en formato [*JSON Web Token* (JWT)](https://jwt.io). A continuación veremos en qué consiste la autenticación basada en tokens y cómo se compone un JWT.

Este tipo de autenticación consiste en que los clientes, para autenticarse en cada petición que realizan al servidor, envían lo que se denomina un **token** que contiene toda la información de autenticación del cliente que el servidor necesita.

El funcionamiento de la autenticación con tokens, a grandes rasgos, es el siguiente: 

1. El usuario se autentica en nuestra aplicación mediante un usuario y contraseña utilizando un servicio habilitado para este fin. 
2. El servidor verifica las credenciales del usuario y en caso de ser correctas genera un token que contiene la información del usuario y lo firma digitalmente. Envía al cliente este token.
3. A partir de este instante, cada petición que haga el usuario va acompañada de este token. El servidor, al recibir el token, verifica que la firma digital es correcta y lee el contenido del mismo para obtener la información del usuario que está utilizando el servicio.

Este mecanismo provee ciertas ventajas:
- El token en sí mismo contiene toda la información necesaria del usuario al que pertenece. Por este motivo, el servidor no tiene que recordar nada de a que usuario pertenece el token. Por esto se dice la autenticación con tokens es ***stateless***.
- El token, cuando se genera, se firma digitalmente con una clave que solo los servidores conocen. Por este motivo, sabremos que un token es correcto y fue generado en uno de nuestros servidores si está firmado por esta clave.
- Dado que el token es *stateless* y cualquiera de nuestros servidores conoce la clave para verificarlo, un token generado en un servidor es válido para autenticarse en cualquier otro de nuestros servidores.


## JWT: JSON Web Token
JWT es un estándar abierto basado en JSON para crear tokens que permiten el uso de recursos de una aplicación o API de una forma segura. Este token llevará incorporada la información del usuario que necesita el servidor para identificarlo, así como información adicional que pueda serle útil (roles, permisos, etc.). Una ventaja de utilizar JWT es que ya existem librerías hechas para la generación y verificación de estos tokens.

Un JWT está compuesto en 3 partes codificadas en base 64, separadas por un punto (`.`):
```
{header}.{payload}.{signature}
```
Por ejemplo:
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.
eyJpc3MiOiJIRUZBTUVAZjNkZXYiLCJzdWIiOiIxMDEwNzUwNkBoZWZhbWUiLCJhdWQiOiJGRURJQ09NIiwiZXhwIjoxNTY5OTQzNTc5NDc3LCJpYXQiOjE1Njk5Mzk5Nzk0NzcsImp0aSI6IjVkOTM2MjBiNTQzZTE0MDdjODk5ZDk5NyJ9.
WL1dnpnNNpby3pN2YCLHzvMSIauVoGAkei2ZI4AKHo0
```


### Header
Es la primera parte del token que contiene metainformacion sobre el propio token como el formato del mismo, el algoritmo de seguridad utilizado, etc ... En nuestro concentrador generamos tokens estandard `JWT` con firma `HS256` (HMAC with SHA-256)
```
{
    "typ": "JWT",
    "alg": "HS256"
}
```

Para el caso que nos atiende, el `Header` no nos interesa mucho ya que usaremos una [librería](https://www.npmjs.com/package/jsonwebtoken) para manejar los JWT que ya gestiona este campo, abstrayéndonos de estos detalles a bajo nivel.

### Payload
Esta segunda parte es el payload, está compuesto por los llamados JWT Claims donde irán colocados los atributos que componen nuestro token. Estos datos son los que le interesan al servidor para saber con que usuario está tratando. Existe múltiple información que podemos proporcionar, en nuestro caso los tokens generados contendrán los siguientes campos:
- `sub` - **El código del usuario**. Es el código del usuario Fedicom para el que se expidió el token.

- `aud` - **El domino del usuario**. Es el [dominio de autenticación](/funcional/conceptos/dominio). A grandes rasgos, indica el tipo de usuario del que se trata. P.e. `FEDICOM` para usuarios Fedicom, `EMPLEADO` para vales de empleado, `FMAS` para llamadas desde F+Online, etc...
- `grupos` - **Grupos del usuario**. Es la lista de los grupos a los que pertenece el usuario. El servidor verificará este campo para realizar ciertas operaciones que requieren permisos específicos. _Este campo es opcional_.
- `exp` - **Fecha de expedición**. Indica el instante en el que se generó el token.
- `iat` - **Valido hasta**. Indica el instante en el que el token debe dejar de aceptarse.

Por ejemplo:
```json
{
  "sub": "Alejandro_AC",
  "aud": "HEFAME",
  "exp": 1557206010,
  "grupos": [
    "FED3_CONSULTAS",
    "FED3_SIMULADOR"
  ],
  "iat": 1557187609
}
```

En el futuro podrían añadirse mas campos a los tokens y esto no afectaría al funcionamiento del código anterior.

### Signature
La tercera parte del token es la firma del mismo, esta se genera utilizando los componentes anteriores (header y payload) más una clave secreta que únicamente conocerá nuestro concentrador. 
En nuestro caso, utilizamos HMAC con SHA256 ([Literatura interesante sobre que es un MAC](http://www.crypto-it.net/eng/theory/mac.html)):

```
HMAC = HMACSHA256(
    base64(header) + "." + base64(payload), <secret_key>
)
Signature = base64(HMAC)
```

Para el caso que nos atiende, el `Signature` no nos interesa en detalle ya que usaremos una [librería](https://www.npmjs.com/package/jsonwebtoken) que realizará los procesos de verificación y firma de JWT, abstrayéndonos de los detalles a bajo nivel.


## JWT en Fedicom3
Siempre que el usuario necesite acceder a un recurso protegido (por ejemplo, para crear un pedido), deberá enviar un token con la solicitud, de manera que el servidor pueda conocer que el usuario es válido.

:::note
Para obtener un token, el usuario deberá utilizar el servicio específico para obtenerlo. Existe una sección donde se detalla el [proceso de obtención de un token por parte de un cliente](/docs/diseno/autenticacion)).
:::

La manera de informar el token en cada llamada a los serivicios es mediante el uso de la cabecera HTTP llamada `Authorization`, con la siguiente forma:

```
Authorization: Bearer <token>
```

El concentrador, al recibir el token, debe comprobar que la firma es válida y verificar que el usuario tiene los permisos necesarios para realizar la operación solicitada.

## Tokens permanentes

Existen casos especiales de pedidos para los que no es posible la autenticación normal de los usuarios. En estos casos, los pedidos se realizan desde una aplicación específica, como por ejemplo, la APP del empleado o F+Online.

Estos tokens identifican el dominio de la aplicación y se expiden sin fecha de caducidad, por lo que todos los pedidos de dichas aplicaciones vienen siempre con el mismo token.
```json
{
	"exp": 9999999999,
	"iat": 0
}
```

:::caution
Inicialmente se expidieron varios tokens permanentes de forma incorrecta. En concreto, los tokens de `SAP`, `F+Online`, `Portal Web Hefame` y `empleado`. Estos se generaron con `exp = 9999999999999` (lleva tres '9' de más), `iat = 1` (en lugar de 0). No existe problema funcional, pero estos casos han de ser tenidos en cuenta en el código, como por ejemplo:

```js
// NO podemos verificar si un token es permanente así:
let esPermanente = (token) => (token.exp === 9999999999 && token.iat === 0) 
// Sino que deberemos hacerlo así:
let esPermanente = (token) => (token.exp >= 9999999999 && token.iat <= 1) 
```
:::


