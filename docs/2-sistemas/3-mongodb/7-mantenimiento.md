---
sidebar_position: 7
title: \@Mantenimiento
---

# Mantenimiento de la base de datos

:::danger
**Este documento es OBSOLETO ya que solo aplica a la versión 1.x del concentrador.**
:::

## Purga de transmisiones antiguas
Comenzamos por limpiar tipos de transmisión que no son de gran interés, como consultas de albaranes, solicitudes de tokens, duplicados ...
```javascript
// Establecemos la fecha a partir de la cual queremos mantener datos
let fechaInicio = ISODate("2022-01-01T00:00:00.000")
let fechaLimite = ISODate("2022-06-01T00:00:00.000")

// Vamos a borrar todo lo que no sea (10: CREAR PEDIDO), (20: CREAR DEVOLUCION) o (50: CREAR LOGISTICA)
let tiposQueSeQuedan = [ 10, 20, 50 ]

// Borramos !
db.tx.remove( {
  type: { $nin: tiposQueSeQuedan }, 
  createdAt: { $lte: fechaLimite, $gte: fechaInicio }
})

```

Segundo, purgaremos transmisiones antiguas que fueron rechazadas:
```javascript
// Establecemos la fecha a partir de la cual queremos mantener datos
let fechaInicio = ISODate("2021-11-01T00:00:00.000")
let fechaLimite = ISODate("2022-02-01T00:00:00.000")

// FALLOS DE AUTENTICACION/RECHAZOS
db.tx.remove({
  status: { $in: [3010, 3120] }, 
  createdAt: { $lt: fechaLimite, $gt: fechaInicio }
})

// DEVOLUVIONES EN ESTADOS INCOMPLETOS
db.tx.remove({
  status: { $in: [1010, 1020, 1030, 3110, 3130, 8100, 9000] }, 
  createdAt: { $lt: fechaLimite, $gt: fechaInicio }
})

// ALL IN ONE
db.tx.remove({
  status: { $in: [1010, 1020, 1030, 3010, 3020, 3110, 3120, 3130, 8100, 9000] }, 
  createdAt: { $lt: fechaLimite, $gt: fechaInicio }
})
```


Dentro de las transmisiones que se queda, hacemos limpieza de campos que ya no nos son relevantes, como las peticiones de SAP o cabeceras HTTP:
```javascript
// Establecemos la fecha a partir de la cual queremos mantener datos
let fechaInicio = ISODate("2021-11-01T00:00:00.000")
let fechaLimite = ISODate("2022-02-01T00:00:00.000")

// Borramos "sapRequest", "sapResponse", "clientResponse.headers"
// y almacenamos el ID del software de farmacia en una variable temporal
// llamada 'idSoftware'
db.tx.updateMany( 
{
  createdAt: { $lt: fechaLimite, $gt: fechaInicio }
},
{
	$unset: {
		"sapRequest": "",
		"sapResponse": ""
	},
	$set: {
		"clientResponse.headers": []
	},
	$rename: {
		"clientRequest.headers.software-id": "idSoftware"
	}
})

// Como ya hemos salvado el valor del ID del software de la farmacia, podemos
// borrar las cabeceras de la petición
db.tx.updateMany( 
  {
    createdAt: { $lt: fechaLimite, $gt: fechaInicio }
  },
	{
		$unset: {
			"clientRequest.headers": "",
	}
});

// Volvemos a dejar el ID del software de la farmacia que habíamos guardado
// temporalmente en 'idSoftware' en la cabecera.
db.tx.updateMany( {createdAt: { $lt: fechaLimite, $gt: fechaInicio }} ,
{
	$rename: {
		"idSoftware": "clientRequest.headers.software-id"
	}
});
```


Tras la purga de datos, algunas transmisiones no se muestran bien en el explorador, y esto se debe a que no tienen Software ID. Generalmente serán peticiones de la APP del empleado. Podemos arreglarlo poniendo a mano el valor de Software ID con el siguiente update:
```javascript
db.tx.updateMany({ 
	"clientRequest.headers": {$exists: false},
	"authenticatingUser" : "empleado"
}, {
	$set: {
		"clientRequest.headers.software-id": "9700"
	}
})
```
