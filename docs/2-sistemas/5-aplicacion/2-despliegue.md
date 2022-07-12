---
sidebar_position: 2
title: Despliegue
---

# Despliegue de la aplicación

## Preparativos previos

### Usuarios y grupos

Solo necesitaremos crear el usuario `fedicom3`, que es quien ejecuta todas las aplicaciones del ecosistema fedicom3:

```
# useradd -u 1100 -U -m fedicom3
```

El usuario tendrá su directorio home por defecto `/home/fedicom3`.


### Directorios

Dentro del `$HOME` del usuario `fedicom3` la aplicación creará los siguientes directorios.

- **f3** Directorio del código de la aplicación Fedicom 3.
- **bin** Directorio con accesos directos a los ejecutables de la aplicación. Este directorio por defecto se incluye en el `$PATH` del usuario.
- **cache** Directorio donde se almacenan ficheros de caché o temporales de la aplicación:
	- ***.pid** Son los ficheros con los PIDs de los procesos lanzados.
	- **db** Directorio donde se almacenará la base de datos SQLite.
  - **logs** La salida de logs de los procesos.
  - **config** Una copia de la configuración del clúster (Esta configuración se almacena en MongoDB. No confundir con la configuración de la instancia).

:::tip
No es preciso crear manualmente estos directorios, ya que el directorio `bin` se incluye por defecto al crear el usuario, el directorio `f3` se crea automáticamente al instalar la aplicación, y el directorio `cache`, junto con todos sus subdirectorios, se crean al arrancar la aplicación por primera vez.
:::


## Instalación

Para la descarga del código fuente de la aplicación, nos logeamos con el usuario "fedicom3" y hacemos la primera descarga del código fuente desde el repositorio git.

```
su - fedicom3
cd $HOME
git clone https://github.com/AlejandroAbad/f3.git
```

Esto nos crea en la carpeta `f3` el codigo fuente de la aplicación. El siguiente paso que debemos seguir es instalar las dependencias:

```
cd $HOME/f3
git config --global url."https://".insteadOf ssh://
npm install
```

Una vez descargadas las fuentes de la aplicación, **es muy conveniente** crear un enlace simbólico al script de arranque parada en el directorio `bin` del usuario. Para esto:

```
chmod ug+x $HOME/f3/script/fedicom3.sh
ln -s $HOME/f3/script/fedicom3.sh $HOME/bin/f3
```

Finalmente, antes de arrancar la aplicación, copiaremos y adaptaremos el fichero de configuración de la misma.

```
cd $HOME/f3
cp config-sample.json config.json
```

## Siguientes pasos
- [Configuración de instancia Fedicom3](/docs/sistemas/aplicacion/configuracion)
- [Configuración del clúster Fedicom3](/docs/sistemas/aplicacion/cluster)
- [Scritps de manejo de los procesos](/docs/sistemas/aplicacion/scripts)
