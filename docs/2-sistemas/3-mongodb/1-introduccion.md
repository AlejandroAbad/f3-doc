---
sidebar_position: 1
title: Introducción a MongoDB
---

# Introducción a MongoDB
La aplicación Fedicom3 utiliza una base de datos MongoDB para almacenar todos los datos necesarios para garantizar que estos entran en SAP.
Esta base de datos es el sustituyente de las librerías ISAM de Fedicom2, obteniendo varias ventajas en el cambio:

1. Soporte nativo de JSON, lo cual es conveniente teniendo en cuenta que el protocolo Fedicom3 se basa en JSON.
2. Replicación automática de la base de datos, consiguiendo que cada nodo tenga exactamente la misma visión de los datos en cualquier momento, y obteniendo HA en los mismos.
3. Permite el balanceo de la carga, lo que se conoce como `Sharding`. No vamos a utilizarlo por el momento ya que no se prevee una carga que lo exija e instalar un sistema con `Sharding` implica aumentar el número de nodos de Mongo DB. Esta característica es lo que permite el **escalado horizontal** del clúster, permitiendo aumentar el _throughput_ de la base de datos considerablemente.

:::tip
**Resumiendo**:
Salto a un sistema gestor de BBDD moderno, mantenido, con replicación de datos y que es capaz de escalar horizontalmente.
:::


Para la aplicación, vamos a montar una única base de datos replicada, mediante lo que se conoce como un _ReplicaSet_. En un _ReplicaSet_, hay un nodo PRIMARIO que es quien recibe las ordenes de escritura y las replica al resto de nodos SECUNDARIOS. Adicionalmente puede existir un nodo ARBITRO que no tiene copia de los datos, sino que sirve para evitar split-brain en caso de caída de un nodo.

Los clientes, a la hora de conectarse, deben conocer previamente una lista de servidores a los que conectarse, esto es, no existe una IP virtual ni nada similar. 

En la operativa normal, un cliente se conecta a todos los nodos disponibles del clúster y tiene consciencia de la topología del mismo (no te agobies, esto lo hace el driver automáticamente). Gracias a esto, y dependiendo de la configuración del propio cliente, puede elegir sabiamente a donde realizar las peticiones de lectura/escritura. (Spoiler: por defecto, todas las operaciones se hacen en el nodo primario)

En un ReplicaSet, si el nodo primario cae, la elección de que nodo va a asumir el rol de primario se hace mediante votación entre el resto de nodos que queden vivos. ¿Qué se considera un nodo vivo? Un nodo está vivo si es capaz de comunicarse con **estrictamente más de la mitad de los nodos del clúster que tienen voto**. El hecho de que un nodo tenga voto o no, es algo que se configura individualmente en cada nodo según nos convenga.

Una vez que hay consenso entre los nodos vivos, para determinar cual debe asumir el rol de primario, se utiliza el nodo con mayor peso. El peso es un valor que se configura individualmente para cada nodo.

## Despligue para Fedicom 3
Para nuestra aplicación, montaremos el siguiente escenario:

![Diagrama de servidores MongoDB](./img/diagrama.svg)


En este diseño se han tenido en cuenta las siguientes directrices:

- Las instancias `F3SAN1`, y `F3MAD1` son las que están pensadas para la alta disponibilidad activa/pasiva del clúster, siendo la instancia de Santomera la de mayor prioridad.
- La instancia `F3BCN` es un mero árbitro, por lo que solo está para romper el empate de votos que podría haber entre Santomera y Madrid en caso de que no pudieran comunicarse.
- La instancia `F3SAN2` no tiene voto, por lo tanto, no puede tener peso. Esto es así para mantener el mismo número de votos en cada sede. Esta instancia tendrá copia de los datos y podrá accederse a la misma en modo de solo lectura. En cualquier momento, como veremos en secciones posteriores, podremos variar estos parámetros.
