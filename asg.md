## Un acercamiento teórico al diseño de NSGs y ASGs en Azure

Ayudando a un cliente que está a punto de subir múltiples VMs a sus suscripciones de Azure, comenzaron a sugir varias preguntas respecto a la seguridad en redes.

¿cuantos NSG? ¿utilizo ASG? ¿qué puede ser mas facil? ¿qué puede ser mas "administrable"?

Primero necesitamos entender los requerimientos:

- Salida libre a Internet estará bloqueda
- Si se utilizarán servicios de Azure (AzureMonitor, AzureBackup, etc)
- Se pretende proteger el trafico entrante/saliente de la subnet (north/south) como así también el tráfico dentro de la subnett (east/west)
- Se desea reducir la cantidad de reglas de NSGs y la complejidad en general.

Imaginemos el siguiente escenario:

Tenemos una aplicación IaaS, muy simple que se compone de dos capas: web y base de datos. Adicionalmente estos servidores son Windows y están unidos a un dominio de Active Directory.

Cada capa de la aplicación está en su respectiva subnet (SN_Web y SN_DB), como así también los controladores de dominio se encuentran en su propia subnet que corresponde a los servicios centrales (SN_CIT).

![](https://github.com/paratz/paratz.github.io/blob/master/scenario_noasg.jpg)

