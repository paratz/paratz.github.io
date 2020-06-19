## Un acercamiento teórico al diseño de NSGs y ASGs en Azure

Ayudando a un cliente que está a punto de subir múltiples VMs a sus suscripciones de Azure, comenzaron a sugir varias preguntas respecto a la seguridad en redes.

¿cuantos NSG? ¿utilizo ASG? ¿qué puede ser mas facil? ¿qué puede ser mas "administrable"?

Primero necesitamos entender los requerimientos:

- Salida libre a Internet estará bloqueda
- Si se utilizarán servicios de Azure (AzureMonitor, AzureBackup, etc)
- Se pretende proteger el trafico entrante/saliente de la subnet (north/south) como así también el tráfico dentro de la subnett (east/west)
- Se desea reducir la cantidad de reglas de NSGs y la complejidad en general.

Imaginemos el siguiente escenario:

