---
sidebar_position: 2
title: Saneado de campos
---

# Saneado de campos del pedido
Cuando entra un pedido del cliente, este se verifica y se sanean los campos.

:::note
Los campos que no están definidos en el protocolo, son eliminados del mensaje, de modo que evitamos consecuencias no esperadas de los mismos.
:::

## Pedido
```js
Pedido:
{
  codigoCliente: <string>,
  numeroPedidoOrigen: <string>,
  codigoAlmacenServicio: <string || undefined>,
  tipoPedido: <string || undefined>,
  lineas: [<Linea>, ...] (1..n),
  direccionEnvio: <string || undefined>,
  codigoAlmacenServicio: <string || undefined>,
  observaciones: <string || undefined>,
  aplazamiento: <number (entero, >0) || undefined>,
  fechaServicio: <DateTime || undefined>
}
```



### Código de cliente
Es un campo definido como obligatorio en el protocolo y se espera que aparezca en el campo `codigoCliente`.

**Verificación:**
- Se verifica que el campo sea un `string` y que al ejecutar la función `trim()` sobre el mismo, el resultado no sea una cadena vacía.

**Transformaciones:**
- Si el código del cliente acaba en "@hefame", se añade un mensaje de advertencia al cliente. El campo no se modifica.
- Se ejecuta la función `trim()` sonbre la cadena para eliminar espacios alrededor.
- Se trunca el código del cliente a 10 caracteres. De lo contrario, SAP explota.


### Número de pedido origen
Es un campo definido como obligatorio en el protocolo y se espera que aparezca en el campo `numeroPedidoOrigen`.

**Verificación:**
- Se verifica que el campo sea un `string` y que al ejecutar la función `trim()` sobre el mismo, el resultado no sea una cadena vacía.

**Transformaciones:**
- Se ejecuta la función `trim()` sonbre la cadena para eliminar espacios alrededor.


### Líneas
Es una lista de la líneas del pedido que se espera en el campo `lineas`. Es obligado que esta sea un `Array` no vacío.
Los elementos que contiene esta lista se verifican independientemente según se especifica mas abajo en este artículo.

### Campos `string` opcionales
La siguiente lista de campos es opcional. De aparecer, y ser un `string` no vacío tras la ejecución de `trim()`, estos campos se mantienen tal cual aparecen a la salida del `trim()`. De no ser un `string` con valor, el campo se elimina del mensaje.
- `direccionEnvio`
- `codigoAlmacenServicio`
- `tipoPedido`
- `observaciones`

### Campos enteros mayor que 0 opcionales
La siguiente lista de campos es opcional. De aparecer, se intenta convertir a un tipo `number` con un valor entero usando `parseInt()`. De ser un entero mayor que 0, el campo se mantiene como entero. De no cumplir los requisitos, el campo se elimina del mensaje.
- `aplazamiento`

### Campos `DateTime` opcionales
La siguiente lista de campos es opcional. De aparecer, y ser un `string` que contiene una fecha o fecha/hora en el formato definido en el protocolo, esto es: `dd/mm/yyyy` para el tipo `Date` y `dd/mm/yyyy HH:MM:ss` para el tipo `DateTime`. De ser una fehca o fecha/hora correcta, estos campos se mantienen tal cual, aplicándoles la función `trim()`. De no ser una fehca o fecha/hora correcta, el campo se elimina del mensaje.
- `fechaServicio` (`DateTime`)

## Línea

```js
Linea:
{
  orden: <number (entero, >0) || autoGenerado>,
  codigoArticulo: <string>,
  cantidad: <number (entero, >0) || 1>,
  valeEstupefaciente: <string || undefined>
  codigoUbicacion: <string || undefined>
  observaciones: <string || undefined>
  fechaLimiteServicio: <DateTime || undefined>
  servicioDemorado: <true || undefined>
  condicion: <{
    codigo: <string>,
    fechaInicio: <DateTime>,
    fechaFin: <DateTime>
  } || undefined>
}
```

### Orden
Es un campo opcional, que se espera en el campo `orden` y que sea un número entero positivo.

**Verificación:**
- Se verifica que sea un número entero mayor que cero.

**Transformaciones:**
- Se convierte a un número enterio usando `parseInt()`.
- Si no es un número entero válido y mayor que cero, se sustituirá por un valor válido que no se haya usado en ninguna otra línea del pedido.


### Código de artículo
Es un campo definido como obligatorio en el protocolo y se espera que aparezca en el campo `codigoArticulo`.

**Verificación:**
- Se verifica que el campo sea un `string` y que al ejecutar la función `trim()` sobre el mismo, el resultado no sea una cadena vacía.

**Transformaciones:**
- Se ejecuta la función `trim()` sonbre la cadena para eliminar espacios alrededor.

### Cantidad
Es un campo obligatorio en el protocolo, que se espera en el campo `cantidad` y que sea un número entero positivo. En caso de no indicarse este campo o no ser un valor válido, se forzará a que tenga el valor igual a 1, para de esta forma, "salvar" la línea.

**Verificación:**
- Se verifica que sea un número entero mayor que cero.

**Transformaciones:**
- Se convierte a un número enterio usando `parseInt()`.
- Si no existe o no es un número entero válido y mayor que cero, se sustituirá por un valor `1`.

### Campo de condición
Es un campo opcional que se espera en el atributo `condicion`. De existir, se comprueba que tenga los tres siguientes campos, o se elimina del mensaje
- **codigo**: Un `string` al que, tras pasarlo por `trim()` es no vacío.
- **fechaInicio**: Un `DateTime` al que se le pasa un `trim()`.
- **fechaFin**: Un `DateTime` al que se le pasa un `trim()`.


### Campos `string` opcionales
La siguiente lista de campos es opcional. De aparecer, y ser un `string` no vacío tras la ejecución de `trim()`, estos campos se mantienen tal cual aparecen a la salida del `trim()`. De no ser un `string` con valor, el campo se elimina del mensaje.
- `codigoUbicacion`
- `valeEstupefaciente`
- `observaciones`

### Campos `boolean` opcionales
La siguiente lista de campos es opcional. De aparecer, si cumplen que `Boolean(valor) === true`, se fuerza el valor `true` en los mismos. De no existir o de no cumplir la condición, se eliminan del mensaje.
- `servicioDemorado`


### Campos `DateTime` opcionales
La siguiente lista de campos es opcional. De aparecer, y ser un `string` que contiene una fecha o fecha/hora en el formato definido en el protocolo, esto es: `dd/mm/yyyy` para el tipo `Date` y `dd/mm/yyyy HH:MM:ss` para el tipo `DateTime`. De ser una fehca o fecha/hora correcta, estos campos se mantienen tal cual, aplicándoles la función `trim()`. De no ser una fehca o fecha/hora correcta, el campo se elimina del mensaje.
- `fechaLimiteServicio` (`DateTime`)