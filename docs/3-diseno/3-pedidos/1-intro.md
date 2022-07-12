---
sidebar_position: 1
title: Procesamiento
---

# Flujo de pedidos
El flujo de entrada de pedidos es el que permite que los usuarios solicitar la creación de un pedido en el sistema.

## Llamada al servicio
Los clientes deberán llamar al endpoint `POST /pedidos`. El cuerpo del mensaje de esta llamada se puede consultar en la [definición del protocolo](http://fedifar.net/fedicomv3/protocolo/), aquí nos centraremos en los campos obligatorios o que sean de especial relevancia para la creación del pedido en el backend.

```js
Pedido:
{
  codigoCliente: <string>,
  numeroPedidoOrigen: <string>,
  codigoAlmacenServicio: <string, opcional>,
  tipoPedido: <string, opcional>,
  lineas: [<Linea>, ...]
}
```

```js
Linea:
{
  orden: <number, opcional>,
  codigoArticulo: <string>,
  cantidad: <number>,
  valeEstupefaciente: <string, opcional>
}
```


Adicionalmente, en la creación del pedido es relevante el `usuario` y `dominio` al que pertenece el token de autenticación enviado por el cliente.



## Procesamiento
Una vez recepcionada la llamada y registrada la transmisión correspondiente en el sistema, se procede al procesamiento de la petición.


1. Se verifica que el cuerpo del mensaje que envía el cliente contiene los valores obligados según el protocolo. Se verifica campo a campo que cumpla con los requisitos y se sanean los campos de entrada (se eliminan campos no definidos, se eliminan espacios al inicio y fin de los *strings*, se convierten los valores al tipo de dato definido en el protocolo, etc...). En la entrada de pedidos trataremos siempre de "arreglar" la entrada del pedido si no es correcta para que este pueda procesarse. Por ejemplo, si no se indica el campo cantidad, obligatorio en el objeto `Linea`, la estableceremos a 1, o si una línea contiene errores, anulamos dicha línea, pero no el resto. Solo en casos extremos rechazaremos el mismo. [Detalles del saneamiento de campos](./saneamiento)

2. Se calcula el CRC del pedido. Los detalles de este cálculo y las implicaciones merecen un [capítulo propio](./crc). 

3. En caso de que la verificación del paso 1 no fuera correcta, la transmisión finaliza indicando al cliente el error encontrado, y marcándose la transmisión como `PETICION INCORRECTA`.

4. Se comprueba si el pedido es un duplicado de otro. Para esto se utiliza el CRC generado del mismo y explicamos en detalle el proceso de detección de duplicados en [el capítulo del CRC](./crc). En caso de encontrar que la solicitud de crear pedido que estamos tratando es un duplicado de otra se le responderá al cliente una copia de la respuesta que se dio con el pedido original, a la cual se le adjunta una incidencia informando de que el pedido es un duplicado de otro. La transmisión finaliza en con estado `DUPLICADO`.