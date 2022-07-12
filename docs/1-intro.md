---
sidebar_position: 1
---

# Motivación
<!-- RESUMIR EN UN PARRAFO MAS CORTO-->
En el año 2000, la Federación de Distribuidoras Farmacéuticas [FEDIFAR](http://fedifar.net/) lanzó el protocolo de comunicaciones Fedicom, con vocación de estandarizar el sistema de transmisiones telemáticas entre la Oficina de Farmacia y los Almacenes de Distribución farmacéutica. Hoy en día, la práctica totalidad de los pedidos que realizan las farmacias a los almacenes se transmiten mediante Fedicom.

En el año 2018, se forma un grupo de trabajo constituido por técnicos con el objetivo de dar forma a una nueva versión del protocolo, que incorpora importantes mejoras con respecto a la anterior, adecuándolo a los nuevos tiempos y formatos disponibles. Así pues nace Fedicom v3, basado en los principios de una API REST e intercambio de mensajes JSON, y que sustituye a las versiones anteriores del protocolo que aún usan transmisiones de mensajes de tamaño fijo a través de *sockets* "en crudo". 
<!-- FIN RESUMIR -->

Debido a que el nuevo protocolo es totalmente incompatible con versiones anteriores, es preciso el desarrollo de un nuevo concentrador para que [HEFAME](https://hefame.es) fuera capaz de recepcionar todas las peticiones de las farmacias. Aquí es donde yo, movido por las ganas de realizar un proyecto de esta de envergadura, paso a diseñar integralmente la arquitectura del servicio que se detalla en esta página. Y así es como consigo poder dedicarme al diseño, implementación, despligue y actual mantenimiento de la infraestructura Fedicom v3. El protocolo ve la luz durante el verano del año 2020.

# Objetivo y requerimientos

El objetivo del proyecto es el **diseño, implementación, despliegue y posterior mantenimiento de un sistema de recepción de transmisiones del protocolo Fedicom 3**, tal y como se define [en la documentación de Fedifar](http://fedifar.net/fedicomv3/) que cumpla los siguientes requisitos:

**1.** Debe **recepcionar transmisiones en formato Fedicom3** para procesarlos y **entregarlos al backend SAP**, que se encargará de crear el pedido. Con la información devuelta por SAP, **deberá dar una respuesta acorde al cliente**.

**2.** Debe implementarse un sistema en **alta disponibilidad** que garantice el funcionamiento constante del servicio.

**3.** El sistema debe poder ser **escalado de manera horizontal**, es decir, en caso de necesitar una mayor potencia de procesamiento, será posible desplegar nodos que de manera automática comenzarán a atender peticiones de los clientes. Asimismo, también debe ser posible eliminar nodos de procesamiento si estos ya no fueran necesarios.

**4.** Cuando una transmisión se recibe, **debe garantizarse la entrega** de la misma al backend SAP, debiendo **asumir que SAP no es un sistema fiable**. En caso de que este no estuviera disponible, las transmisiones quedarán almacenadas para su posterior procesamiento.

**5.** Debe proveerse los medios necesarios para la **monitorización y la administración** de todo el sistema.