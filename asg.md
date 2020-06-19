# Un acercamiento teórico al diseño de NSGs y ASGs en Azure

## La situación:
Ayudando a un cliente que está a punto de subir múltiples VMs a sus suscripciones de Azure, comenzaron a surgir varias preguntas respecto a la seguridad en redes.

¿cuantos NSG? ¿utilizo ASG? ¿qué puede ser mas facil? ¿qué puede ser mas "administrable"?

Primero necesitamos entender los requerimientos:

- Se utilizarán servicios de Azure (AzureMonitor, AzureBackup, etc)
- Se pretende proteger el trafico entrante/saliente de la subnet (north/south) como así también el tráfico dentro de la subnett (east/west), por ejemplo bloquear la salida a internet. Solo se permite el tráfico explícitamente definido.
- Se desea reducir la cantidad de reglas de NSGs y la complejidad en general.

## El Escenario:

Tenemos una aplicación IaaS -app1-, muy simple en su arquitectura que se compone de dos capas: web y base de datos. Adicionalmente estos servidores son Windows y están unidos a un dominio de Active Directory.

Cada capa de la aplicación está en su respectiva subnet (SN_Web y SN_DB), como así también los controladores de dominio se encuentran en su propia subnet que corresponde a los servicios centrales (SN_CIT).

![](https://github.com/paratz/paratz.github.io/blob/master/scenario_noasg.jpg)

## Una posible forma:

En primer instancia trabajaremos con 3 Network Security Groups en Azure, los cuales estarán asignados cada uno a las subredes que tenemos:


| NSG | Asignado a |
|---|---|
| nsg_sn_cit | Subnet Central IT |
| nsg_sn_web | Subnet Web |
| nsg_sn_db | Subnet Database |


Luego, uno de los requerimientos es minimizar la complejidad. Cuando trabajamos con NSGs, la respuesta a este requisito son los Application Security Groups. Vamos a crear los siguientes ASG:


| ASG | Contiene |
|---|---|
|asg_domainmembers | todos los servidores windows en azure unidos al dominio |
|asg_dcs | domain controllers en azure |
|asg_app1_web | todos los servidores web de la app1 |
|asg_app1_db | todos los servidores base de datos de la app1 |
|asg_paws | todos los servidores jump de administración |


Aqui es importante mencionar que una VM puede pertenecer a más de un ASG, por ejemplo cualquier servidor web es miembro del asg_app1_web, cómo así también del asg_domainmembers.

Creados los NSG y creados los ASG, tenemos que comenzar a crear las reglas en los NSGs. Vamos a empezar con los requerimientos más simples, que es la funcionalidad de la aplicación web. Pensemos que cualquiera debería poder alcanzar la capa web en el puerto TCP 443 (HTTPS), por lo tanto crearemos una regla que permita esto, como así también, necesitamos que la capa web pueda comunicarse con la base de datos a través del puerto TCP 1433. Aqui vienen las primeras reglas:


|NSG|Aplicado a |Prioridad|Dirección|Regla|Desde|Hacia|Puertos|
|---|---|---|---|---|---|---|---|
|NSG_SN_Web|Subnet Web|450|Incoming|**app1 - Allow Web Traffic n/s**|*|asg_app1_web|HTTPS|
|NSG_SN_Web|Subnet Web|440|Outgoing|**app1 - Allow DB Traffic n/s**|asg_app1_web|asg_app1_db|SQL|
| | | | | | | | |
|NSG_SN_DB|Subnet Database|460|Incoming|**app1 - Allow DB Traffic n/s**|asg_app1_web|asg_app1_db|SQL|


Es importante destacar, que hicimos una regla saliente desde la subnet web, para luego hacer una regla entrante en la subnet de base de datos.

Ahora, tenemos que asegurarnos que estos miembros del dominio, tienen la conectividad necesaria con los controladores de dominio. Para esto, simplemente permitimos la comunicación utilizando el asg_domainmembers, ya que no solo la app1, sino todo servidor miembro del dominio necesita esta conectividad. Esta es nuestra nueva configuración de reglas:


|NSG|Aplicado a |Prioridad|Dirección|Regla|Desde|Hacia|Puertos|
|---|---|---|---|---|---|---|---|
|NSG_SN_Web|Subnet Web|450|Incoming|app1 - Allow Web Traffic n/s|*|asg_app1_web|HTTPS|
|NSG_SN_Web|Subnet Web|440|Outgoing|app1 - Allow DB Traffic n/s|asg_app1_web|asg_app1_db|SQL|
|NSG_SN_Web|Subnet Web|470|Outgoing|**inf - Allow Domain Traffic n/s**|asg_domainmembers|asg_dcs|Kerberos, DNS, RPC|
| | | | | | | | |
|NSG_SN_DB|Subnet Database|460|Incoming|app1 - Allow DB Traffic n/s|asg_app1_web|asg_app1_db|SQL|
|NSG_SN_DB|Subnet DB|470|Outgoing|**inf - Allow Domain Traffic n/s**|asg_domainmembers|asg_dcs|Kerberos, DNS, RPC|
| | | | | | | | |
|NSG_SN_CIT|Subnet Central IT|470|Incoming|**inf - Allow Domain Traffic n/s**|asg_domainmembers|asg_dcs|Kerberos, DNS, RPC|


Aquí vemos que las reglas que permiten la conectividad a los controladores de dominio, son reglas que deberían estar presentes en todos los NSG que apliquen a subredes donde haya miembros del dominio. Es decir, esas son las primeras reglas a nivel infrastructura que debemos tener, de ahi el prefijo INF-.

Otro de los requisitos es tener conectividad a los servicios de Azure como Azure Monitor o Azure Backup, por lo cual, tenemos reglas de Infrastructura adicionales, entendiendo que cualquier servidor del dominio puede utilizar estos servicios:


|NSG|Aplicado a |Prioridad|Dirección|Regla|Desde|Hacia|Puertos|
|---|---|---|---|---|---|---|---|
|NSG_SN_Web|Subnet Web|450|Incoming|app1 - Allow Web Traffic n/s|*|asg_app1_web|HTTPS|
|NSG_SN_Web|Subnet Web|440|Outgoing|app1 - Allow DB Traffic n/s|asg_app1_web|asg_app1_db|SQL|
|NSG_SN_Web|Subnet Web|470|Outgoing|inf - Allow Domain Traffic n/s|asg_domainmembers|asg_dcs|Kerberos, DNS, RPC|
|NSG_SN_Web|Subnet Web|480|Outgoing|**inf - Allow Azure Mgmt Taffic n/s**|asg_domainmembers|Service Tag: AzureMonitor, Backup|HTTPS|
| | | | | | | | |
|NSG_SN_DB|Subnet Database|460|Incoming|app1 - Allow DB Traffic n/s|asg_app1_web|asg_app1_db|SQL|
|NSG_SN_DB|Subnet DB|470|Outgoing|inf - Allow Domain Traffic n/s|asg_domainmembers|asg_dcs|Kerberos, DNS, RPC|
|NSG_SN_DB|Subnet DB|480|Outgoing|**inf - Allow Azure Mgmt Taffic n/s**|asg_domainmembers|Service Tag: AzureMonitor, Backup|HTTPS|
| | | | | | | | |
|NSG_SN_CIT|Subnet Central IT|470|Incoming|inf - Allow Domain Traffic n/s|asg_domainmembers|asg_dcs|Kerberos, DNS, RPC|
|NSG_SN_CIT|Subnet Central IT|480|Outgoing|**inf - Allow Azure Mgmt Taffic n/s**|asg_domainmembers|Service Tag: AzureMonitor, Backup|HTTPS|


Otro requerimiento, asumido, pero no listado es la posibilidad de administrar los equipos desde estaciones de trabajo seguras (PAWs -Privileged Access Workstations), que se encuentran en Central IT, por lo cual se creó un asg que contiene estos servidores de administración para permitir la conectividad, a través de nuevas reglas:


|NSG|Aplicado a |Prioridad|Dirección|Regla|Desde|Hacia|Puertos|
|---|---|---|---|---|---|---|---|
|NSG_SN_Web|Subnet Web|450|Incoming|app1 - Allow Web Traffic n/s|*|asg_app1_web|HTTPS|
|NSG_SN_Web|Subnet Web|480|Incoming|**inf Allow PAW Management Traffic n/s**|asg_paws|asg_domainmembers|RDP, RM, RPS|
|NSG_SN_Web|Subnet Web|440|Outgoing|app1 - Allow DB Traffic n/s|asg_app1_web|asg_app1_db|SQL|
|NSG_SN_Web|Subnet Web|470|Outgoing|inf - Allow Domain Traffic n/s|asg_domainmembers|asg_dcs|Kerberos, DNS, RPC|
|NSG_SN_Web|Subnet Web|480|Outgoing|inf - Allow Azure Mgmt Taffic n/s|asg_domainmembers|Service Tag: AzureMonitor, Backup|HTTPS|
| | | | | | | | |
|NSG_SN_DB|Subnet Database|460|Incoming|app1 - Allow DB Traffic n/s|asg_app1_web|asg_app1_db|SQL|
|NSG_SN_DB|Subnet DB|480|Incoming|**inf Allow PAW Management Traffic n/s**|asg_paws|asg_domainmembers|RDP, RM, RPS|
|NSG_SN_DB|Subnet DB|470|Outgoing|inf - Allow Domain Traffic n/s|asg_domainmembers|asg_dcs|Kerberos, DNS, RPC|
|NSG_SN_DB|Subnet DB|480|Outgoing|inf - Allow Azure Mgmt Taffic n/s|asg_domainmembers|Service Tag: AzureMonitor, Backup|HTTPS|
| | | | | | | | |
|NSG_SN_CIT|Subnet Central IT|470|Incoming|inf - Allow Domain Traffic n/s|asg_domainmembers|asg_dcs|Kerberos, DNS, RPC|
|NSG_SN_CIT|Subnet Central IT|480|Outgoing|inf - Allow Azure Mgmt Taffic n/s|asg_domainmembers|Service Tag: AzureMonitor, Backup|HTTPS|
|NSG_SN_CIT|Subnet Central IT|490|Outgoing|**inf Allow PAW Management Traffic n/s**|asg_paws|asg_domainmembers|RDP, RM, RPS|


Con todas las reglas agregadas a este punto ya cumplimos los requisitos de la conectividad que debemos permitir, sin embargo los NSG tienen reglas default que permiten el resto del tráfico (IntraVnet o saliente a internet), por lo cual agregaremos nuevas reglas que bloqueen todo el tráfico que no coincida con el definido de manera explícita.


|NSG|Aplicado a |Prioridad|Dirección|Regla|Desde|Hacia|Puertos|
|---|---|---|---|---|---|---|---|
|NSG_SN_Web|Subnet Web|450|Incoming|app1 - Allow Web Traffic n/s|*|asg_app1_web|HTTPS|
|NSG_SN_Web|Subnet Web|480|Incoming|inf Allow PAW Management Traffic n/s|asg_paws|asg_domainmembers|RDP, RM, RPS|
|NSG_SN_Web|Subnet Web|500|Incoming|**inf - Block north/west east/west Unmatched Traffic**|*|*|*|
|NSG_SN_Web|Subnet Web|440|Outgoing|app1 - Allow DB Traffic n/s|asg_app1_web|asg_app1_db|SQL|
|NSG_SN_Web|Subnet Web|470|Outgoing|inf - Allow Domain Traffic n/s|asg_domainmembers|asg_dcs|Kerberos, DNS, RPC|
|NSG_SN_Web|Subnet Web|480|Outgoing|inf - Allow Azure Mgmt Taffic n/s|asg_domainmembers|Service Tag: AzureMonitor, Backup|HTTPS|
|NSG_SN_Web|Subnet Web|500|Outgoing|**inf - Block north/west east/west Unmatched Traffic**|*|*|*|
| | | | | | | | |
|NSG_SN_DB|Subnet Database|460|Incoming|app1 - Allow DB Traffic n/s|asg_app1_web|asg_app1_db|SQL|
|NSG_SN_DB|Subnet DB|480|Incoming|inf Allow PAW Management Traffic n/s|asg_paws|asg_domainmembers|RDP, RM, RPS|
|NSG_SN_DB|Subnet DB|500|Incoming|**inf - Block north/west east/west Unmatched Traffic**|*|*|*|
|NSG_SN_DB|Subnet DB|470|Outgoing|inf - Allow Domain Traffic n/s|asg_domainmembers|asg_dcs|Kerberos, DNS, RPC|
|NSG_SN_DB|Subnet DB|480|Outgoing|inf - Allow Azure Mgmt Taffic n/s|asg_domainmembers|Service Tag: AzureMonitor, Backup|HTTPS|
|NSG_SN_DB|Subnet DB|500|Outgoing|**inf - Block north/west east/west Unmatched Traffic**|*|*|*|
| | | | | | | | |
|NSG_SN_CIT|Subnet Central IT|470|Incoming|inf - Allow Domain Traffic n/s|asg_domainmembers|asg_dcs|Kerberos, DNS, RPC|
|NSG_SN_Web|Subnet Web|500|Incoming|**inf - Block north/west east/west Unmatched Traffic**|*|*|*|
|NSG_SN_CIT|Subnet Central IT|480|Outgoing|inf - Allow Azure Mgmt Taffic n/s|asg_domainmembers|Service Tag: AzureMonitor, Backup|HTTPS|
|NSG_SN_CIT|Subnet Central IT|490|Outgoing|inf Allow PAW Management Traffic n/s|asg_paws|asg_domainmembers|RDP, RM, RPS|
|NSG_SN_CIT|Subnet Central IT|500|Outgoing|**inf - Block north/west east/west Unmatched Traffic**|*|*|*|

## Palabras Finales:

- Esto es un modelo teórico, si se implementa tal cual, lo mas probable es que los controladores de dominio no funcionen por que no podrán replicar entre si o que no sea posible conectarse a las PAWs de administración. Simplemente intenta mostrar una forma de segurizar una red con componentes IaaS en Azure y requiere que su previa implementación sea evaluada en detalle para cada sistema y componente involucrado.
- La creación de las reglas de infrastructura junto con ASG "globales" ayuda a definir el tráfico que debe existir, aquí utilicé algo bien simple para la conectividad del dominio y de PAWs, pero claramente puede haber otras integraciones, como la conectividad a on-prem o sistemas de monitoreo que deben tenerse en cuenta.
- El límite de reglas por NSG es de 1000, en este caso se utilizaron 3 reglas para permitir el tráfico explícito de la app1. Pensando en un escenario real, donde haya 50 reglas de infrastructura, podríamos alojar como unas 200 aplicaciones con reglas similares... en cada subnet.
- Este escenario es un escenario de lock-down, es decir que solo el tráfico explicitamente permitido puede pasar. En muchos casos la primer opción para esto es desplegar un NVA que centralice estas operaciones, pero no es lo que se intenta demostrar aquí.


## Links Adicionales:

[Networking limits - Azure Resource Manager](https://docs.microsoft.com/en-us/azure/azure-resource-manager/management/azure-subscription-service-limits?toc=/azure/virtual-network/toc.json#azure-resource-manager-virtual-networking-limits)

[Network security groups](https://docs.microsoft.com/en-us/azure/virtual-network/security-overview)

[Secure VNets](https://docs.microsoft.com/en-us/azure/cloud-adoption-framework/migrate/azure-best-practices/migrate-best-practices-networking#secure-vnets)

[Azure best practices for network security](https://docs.microsoft.com/en-us/azure/security/fundamentals/network-best-practices#:~:text=When%20you%20use%20network%20security,to%20ensure%20simplicity%20and%20flexibility.)