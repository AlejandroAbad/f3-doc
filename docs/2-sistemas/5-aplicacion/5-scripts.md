---
sidebar_position: 5
title: Scripts
---

# Scripts de Arranque/Parada
Las operaciones de arranque y parada del servicio Fedicom3 se realizan mediante el script `f3` que habremos linkeado en el directorio `bin` del usuario `fedicom3` durante el [despliegue de la aplicación en el servidor](/docs/sistemas/aplicacion/despliegue).

# Documentación del script **f3**

```bash
Uso: /home/fedicom3/bin/f3 (<accion> [<opciones>]) | -h

    -h  Muestra esta ayuda

ACCIONES:
    start
        Arranca los procesos de la aplicación Fedicom 3.

    stop
        Detiene los procesos de la aplicación Fedicom 3.

    restart
        Reinicia los procesos de la aplicación Fedicom 3.
        Equivale a ejecutar 'f3 stop' y 'f3 start' en ese orden.

    status
        Muestra el estado de los procesos Fedicom 3 ejecuntandose en el servidor.

OPCIONES:
    --actualizar-git -u
        Actualiza la aplicacion desde el repositorio GIT.

    --actualizar-npm -n
        Realiza una instalación límpia de las dependencias NodeJS.

    --limpiar-log -l
        Elimina los logs del directorio de logs. Esto NO elimina los ficheros de DUMP.

    --limpiar-sqlite -s
        Purga la base de datos auxiliar SQLite.
```


