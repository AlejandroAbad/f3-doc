---
sidebar_position: 3
title: CRC
---

# CRC
El CRC es un valor que se calcula en base a determinados campos del pedido y que sirve para identificar a cada pedido "único" y seguir si ciclo de vida. Este CRC es un valor de 12 bytes (96bits), lo que nos deja casi con un 8 con 28 ceros detrás de posibles CRCs.

En breve explicaremos como generamos este CRC, pero por el momento imaginemos que entra un pedide de PEPE, y a este pedido se le calcula un valor de CRC de `"123crc"`. Este pedido se procesa en el sistema y acaba, por ejemplo, en estado `RECHAZADO SAP`, porque resulta que el usuario PEPE es un nuevo cliente y no está dado de alta aún en SAP.

Inmediatamente, el departamento de ventas da de alta al usuario en el sistema, y le comunica a PEPE que ya puede realizar su pedido, quien lo vuelve a intentar. Como el pedido transmitido es "el mismo" que intentó transmitir con anterioridad, el concentrador le calcula el mismo CRC de `"123crc"`. Así, el sistema sabe que el pedido no es nuevo y actua en consecuencia. En este caso , como el pedido no terminó correctamente, el concetrador lo vuelve a transmitir a SAP para crearlo. Como ahora está todo en orden en SAP, el pedido se crea correctamente y este acaba en estado `COMPLETADO`.

Ahora imaginemos que PEPE, por equivocación, vuelve a enviar el mismo pedido. De nuevo, el concentrador le calculará el mismo CRC de `"123crc"` y podrá comprobar inmediatamente que el pedido ya está en el sistema y que está `COMPLETADO`. En este caso, el concentrador indica a PEPE que ya tiene este pedido y registra la petición de PEPE como un `DUPLICADO`.



## Cálculo del CRC
Comencemos por la función que genera un CRC para cualquier valor que se le pase. Es simple: Se concatenan todos los valores que queramos que formen parte del CRC, y se hace un HASH SHA1 de la suma. Luego se convierte a formato hexadecimal y se crea un `ObjectID` (objeto de ID estandard de MongoDB) que es de los 12 bytes que queremos.

```js
function generarCRC(...valores) {
	let base = valores.reduce((acumulado, actual) => {
		return acumulado + actual;
	}, ''); // Poner '' como valor inicial nos garantiza un string a la salida
	let hash = crypto.createHash('sha1');
	return new ObjectID(hash.update(base).digest('hex').substring(1, 25));
}
```

Originalmente, el CRC de cada pedido se calculaba en base únicamente al día de creación, al `codigoCliente` y al valor de `numeroPedidoOrigen`.
```js
let crc = generarCRC('yyyymmdd', codigoCliente, numeroPedidoOrigen)
```

Dado que el `numeroPedidoOrigen` debe ser único para cada cliente, en teoría esto basta para detectar que el cliente está retransmitiendo un pedido. En la práctica esto no siempre se cumple, y los programas de farmacia en ocasiones, para un mismo pedido, repiten el `numeroPedidoOrigen`, dando lugar a duplicados.

Por esto, se adoptó un enfoque similar al que toma Fedicom2 para esto: Para pedidos de mas de N líneas (el valor de N definido en la configuración del clúster), el CRC se calcula en base a las líneas del pedido, de modo que, para cada línea se calcula un CRC, tal que:

```js
let crcLinea = generarCRC(codigoArticulo, cantidad, cantidadBonificacion, valeEstupefaciente)
```

Finalmente se acumulan todos los CRCs y se añaden los campos del día de creación, el `codigoCliente`, el valor de `numeroPedidoOrigen` y el valor de `codigoAlmacenServicio`.

```js
let crcAcumuladoLineas = lineas.reduce( (acumulado, linea ) => {
	return generarCRC(acumulado, linea.crc)
}, '')
let crc = generarCRC(crcAcumuladoLineas, 'yyyymmdd', codigoCliente, numeroPedidoOrigen, codigoAlmacenServicio)

```

## Gestión de duplicados
A la hora de definir si una transmisión es un duplicado, lo que hace el concentrador es buscar en MongoDB por pedidos del día actual (recordemos que el cálculo del CRC incluye el día) y que tengan el mismo CRC.

En caso de encontrarse, se decide si el pedido actual debe finalizarse como `DUPLICADO` o si debe seguir su normal procesamiento. Esto de hace en función del estado en el que esté el pedido anterior. Los estados que permiten que el pedido continue ejecutandose se definen en la configuración del clúster.


