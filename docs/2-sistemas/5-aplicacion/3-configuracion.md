---
sidebar_position: 3
title: Configuración
---

# Configuración de la aplicación
El código fuente de la aplicación trae consigo un fichero de configuración de ejemplo. 
Solamente tendremos que copiar el fichero de configuración que viene de ejemplo y modificarlo según las necesidades.

```bash
su - fedicom3
cp ~/f3/config-sample.json ~/f3/config.json
```

Este es el aspecto del fichero de ejemplo:


```json
{
	"produccion": false,
	"numeroWorkers": 1,
	"mongodb": {
		"servidores": [
			"f3dev2.hefame.es:27017",
			"f3dev1.hefame.es:27017"
		],
		"usuario": "fedicom3",
		"password": "password",
		"database": "fedicom3",
		"replicaSet": "fedicom3-dev",
		"intervaloReconexion": 5000
	},
	"directorioCache": "/home/fedicom3/cache",
	"log": {
		"consola": false
	},
	"http": {
		"puertoConcentrador": 5000,
		"puertoMonitor": 5001
	}
}
```

# Configuración general

| Parámetro                     | Obligatorio   | Defecto | Descripción        |
| ------------------            |:-------------:| ------- | ------------------ |
| `produccion` | si | | Indica si la instancia es de producción o de test. Las instancias de producción no admiten transmisiones desde el simulador y pueden llevar a cabo otras comprobaciones para evitar que algún despistado la líe. 🤡 |
| `sinWatchdogPedidos` | no | `false` | Indica si no se debe lanzar el proceso Watchdog de pedidos. |
| `sinWatchdogSqlite` | no | `false` | Indica si no se debe lanzar el proceso Watchdog de SQLite. |
| `sinMonitor` | no | `false` | Indica si no se debe lanzar el proceso monitor. |
| `numeroWorkers` | no | `1` | El número de procesos `worker` que la aplicación desplegará para atender peticiones. |
| `directorioCache` | si | | La ruta al directorio donde se alamacenarán los datos de caché y temporales de la instancia. |


# Configuración de bases de datos

| Parámetro                     | Obligatorio   | Defecto   | Descripción        |
| ------------------            |:-------------:| --------- | ------------------ |
| `mongodb`                     | si            |           | *Un objeto con la configuración de MongoDB.* |
| `mongodb`.`servidores`             | si            |           | Una lista con los nombres de host de los servidores del ReplicaSet de MongoDB. |
| `mongodb`.`usuario`          | si            |           | El usuario a utilizar para la conexión a MongoDB. |
| `mongodb`.`password`               | si            |           | La contraseña del usuario para la conexión a MongoDB. |
| `mongodb`.`database`          | si            |           | El nombre de la base de datos. |
| `mongodb`.`replicaSet`        | si            |           | El nombre del replica set. |
| `mongodb`.`intervaloReconexion`      | no            | `5000`       | El tiempo, en **milisegundos**, que la aplicación esperará entre intentos de reconexión a la base de datos.  |


# Configuración de HTTP

| Parámetro                     | Obligatorio   | Defecto   | Descripción        |
| ------------------            |:-------------:| --------- | ------------------ |
| `http`                     | no            |  `{}`         | *Un objeto con la configuración de HTTP.* |
| `http`.`puertoConcentrador`             | no            | `5000`          | El puerto en el que la instancia escuchará peticiones HTTP para transmisiones Fedicom3. Estas se envían a los procesos `worker`. |
| `http`.`puertoMonitor`             | no            | `5001`          | El puerto en el que la instancia escuchará peticiones HTTP para consultas de monitorización. Estas se envían al proceso `monitor`. |





# Configuración de log

| Parámetro                     | Obligatorio   | Defecto   | Descripción        |
| ------------------            |:-------------:| --------- | ------------------ |
| `log`                     | no            |  `{}`         | *Un objeto con la configuración de los logs.* |
| `log`.`consola`             | no            | `false`          | Indica al logger si debe escribir la salida del log por consola. |


