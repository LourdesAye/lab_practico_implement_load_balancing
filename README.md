# Laboratorio Práctico de Google Cloud Platform (GCP) : "Implement Load Balancing on Compute Engine"
Se ha realizado un laboratorio práctico en Google Cloud Platform (GCP) en el que se implementó un balanceador de cargas.

## Pasos Realizados
1. Definir variables de entorno (accesibles dentro de la misma sesión de terminal y se usan para evitar repetir valores en los comandos.)
```
export INSTANCE=nucleus-jumphost-624 
export FIREWALL=accept-tcp-rule-112
export REGION=us-east1
export ZONE=us-east1-c
```
* INSTANCE almacena el nombre de la instancia de máquina virtual (VM).
* FIREWALL almacena el nombre de la regla de firewall.
* REGION y ZONE definen la ubicación donde se ejecutarán los recursos en GCP.

> [!NOTE]
> ¿Por qué definir zona y región?   
> GCP divide su infraestructura en regiones (conjuntos de centros de datos) y dentro de cada región hay zonas.  
> Por ejemplo: la región “us-central1” corresponde a una región en el centro de Estados Unidos que tiene las zonas us-central1-a, us-central1-b, us-central1-c y us-central1-f.
> La zona determina en qué centro de datos específico se aloja la VM.    
> Un disco persistente y una instancia de máquina virtual deben ubicarse en la misma zona para poder conectarlos.   
> Esto es importante por:  
> Latencia y disponibilidad: Si tu app tiene usuarios en EE.UU., conviene elegir una zona cercana.  
> Redundancia: Si un centro de datos falla, otras zonas pueden seguir funcionando.  
> Información sobre las regiones y zonas: [Regiones](https://cloud.google.com/docs/geography-and-regions?hl=es-419) y [Zonas](https://cloud.google.com/compute/docs/regions-zones?hl=es-419)  


> [!NOTE]
> Los valores de las variables de entorno son proporcionadas por el laboratorio al momento de su realización.

2. Creación de una Máquina Virtual (VM) en Google Cloud: 
```
gcloud compute instances create $INSTANCE \
    --zone=$ZONE \
    --machine-type=e2-micro
```
* ```gcloud compute instances create``` crea una máquina virtual (VM) en Google Cloud.
* $INSTANCE es el nombre de la VM, tomado de la variable de entorno.
* --zone=$ZONE define en qué zona geográfica se creará la VM.
* --machine-type=e2-micro define el tipo de máquina.

> [!NOTE]
> ¿Qué es e2-micro?  
> Es un tipo de máquina con: 2 vCPUs compartidas y 1 GB de RAM. Adecuada para pruebas y pequeños servidores.

3. Generar un script para crear y ejecutar un archivo de inicio llamado startup.sh, que va a permitir la instalación y configuración de Nginx en una máquina virtual.
```
cat << EOF > startup.sh
#! /bin/bash
apt-get update
apt-get install -y nginx
service nginx start
sed -i -- 's/nginx/Google Cloud Platform - '"\$HOSTNAME"'/' /var/www/html/index.nginx-debian.html
EOF
```

* cat es para escribir un bloque de texto en un archivo startup.sh.
* EOF es un delimitador que indica dónde comienza y termina el contenido a copiar en el archivo startup.sh.
* ">" redirige la salida al archivo startup.sh, sobrescribiéndolo (si existía).
* #! : Especifica que el archivo es un script e indica qué programa debe usarse para ejecutarlo (intérprete)
  * Ejemplo 1: #!/bin/sh → Usa el shell sh (menos funciones que Bash, pero más rápido).
  * Ejemplo 2: #!/usr/bin/python3 → Usa Python 3 como intérprete.
  * Ejemplo 3: #!/usr/bin/env node → Usa Node.js como intérprete.
* ```apt-get update```: Actualiza la lista de paquetes disponibles en el sistema.
* ```apt-get install -y nginx```: Instala el servidor web Nginx. ```-y``` evita que el sistema pida confirmación ("¿Quieres instalar nginx? [y/n]"), aceptando automáticamente.
* ```service nginx start```: Inicia el servidor Nginx, haciendo que la página web sea accesible.
* sed es un editor de texto en línea de comandos.
* -i edita el archivo directamente (sin -i, solo mostraría el cambio sin aplicarlo).
* 's/nginx/Google Cloud Platform - '"$HOSTNAME"'/': reemplaza la palabra "nginx" en el archivo con "Google Cloud Platform - $HOSTNAME", recordando que $HOSTNAME es una variable que contiene el nombre de la máquina virtual.
* /var/www/html/index.nginx-debian.html: Es la página principal que se muestra cuando accedes a la IP de la VM.

> [!NOTE]
> ¿Qué es Nginx y para qué se usa?  
> Nginx es un servidor web y proxy inverso.  
> Servidor web: Es un software que escucha solicitudes HTTP y las responde enviando páginas web a los clientes (en sus navegadores).  
> Proxy inverso: Es un intermediario que recibe peticiones y las reenvía a servidores internos. Se usa para balanceo de carga, seguridad y caché.    
>
>  Ejemplo:  
> Si visitas www.ejemplo.com, el servidor web te envía la página.
> Si hay muchas visitas, un proxy inverso reparte las solicitudes entre varios servidores para evitar sobrecargas.  
>
> En este caso:
> * Nginx es el software del servidor web.
> * Se instala en una máquina virtual (VM).
> * Recibe peticiones HTTP de los usuarios y envía la página web solicitada.


4. Crear un Template de Instancia
> [!NOTE]
> Un template de instancia es una plantilla que define la configuración base que van a tener las máquinas virtuales (VMs) a crear.
> No crea VMs, solo guarda la configuración para usarse después.
> Permite crear múltiples VMs con la misma configuración de forma automática. Es decir, si se necesita escalar el número de servidores, se puede usar este template sin definir los parámetros una y otra vez.
> 

```
gcloud compute instance-templates create web-server-template \
        --metadata-from-file startup-script=startup.sh \
        --machine-type e2-medium \
        --region $REGION
```

* web-server-template: Es el nombre del template (puede tener cualquier nombre).
* --metadata-from-file startup-script=startup.sh: permite incluir un script de inicio (startup.sh, el cual fue creado en el paso 3, el paso anterior) que se ejecutará cuando una VM arranque. Este script permite instalar y configurar Nginx en la máquina virtual (MV).
* --machine-type e2-medium: Define el tipo de máquina virtual (2 vCPUs y 4GB de RAM).
* --region $REGION: Especifica la región donde se creará la instancia cuando se use el template.

5. Crear de un Grupo de Instancias
```
gcloud compute instance-groups managed create web-server-group \
        --base-instance-name web-server \
        --size 2 \
        --template web-server-template \
        --region $REGION
```

Aquí se están creando un grupo de instancias gestionadas (Managed Instance Group, MIG). Se trata de un conjunto de máquinas virtuales que trabajan juntas como si fueran un solo sistema.  
Se usa para escalar automáticamente ( si el tráfico aumenta, puede crear más VMs ;  si el tráfico baja, puede eliminar VMs para ahorrar costos).  
Las VMs dentro del grupo se crean a partir del template que se definio en el paso anterior (Paso 4).  

* web-server-group: Es el nombre del grupo de instancias.
* --base-instance-name web-server: Define el prefijo de los nombres de las VMs. Si tiene varias instancias quedaría, por ejemplo, web-server-abc123 y web-server-def456
* --size 2: Crea 2 instancias dentro del grupo.
* --template web-server-template: Usa el template que creado antes (en el Paso 4) (web-server-template). Aquí es cuando se crean realmente las VMs, siguiendo la configuración del template.
--region $REGION: Especifica en qué región se van a desplegar estas VMs.
  
Resumen:
* Antes (Paso 4) se creó una plantilla (pero no VMs).
* Ahora (Paso 5) se usa la plantilla para crear 2 VMs dentro de un grupo de instancias.

6. Crear regla de firewall que permita tráfico HTTP en el puerto 80.
```
gcloud compute firewall-rules create $FIREWALL \
        --allow tcp:80 \
        --network default
```  
Un firewall es un sistema de seguridad que filtra y controla el tráfico de red basado en reglas.  
En Google Cloud hay un firewall predeterminado que bloquea casi todo el tráfico excepto SSH, RDP y ICMP.    
El comando expresado permite crear una nueva regla para permitir tráfico HTTP en el puerto 80.  

En Google Cloud Platform, cada cuenta tiene una red predeterminada llamada "default". Es la red en la que se crean todos los recursos (VMs, balanceadores de cargas, etcétera) en caso de no especificar otra.  
Para ver las redes disponibles en GCP se puede usar:   
```gcloud compute networks list```.  

7. Crear un Health Check  
```gcloud compute http-health-checks create http-basic-check```      
Un Health Check es una prueba automática que verifica si las VMs están funcionando (revisa si Nginx está respondiendo en HTTP). El health check envía solicitudes HTTP a cada VM. Si una VM responde correctamente, el balanceador sigue enviándole tráfico. Pero si una VM no responde o falla, el balanceador la deja de usar.  
Ejemplo:  
El health check revisa http://web-server-1/ cada 10 segundos.  
Si devuelve 200 → ✅  → La VM está sana.  
Si no responde  → ❌ →  Se saca del balanceador hasta que vuelva a responder.

8. Configurar los puertos en las máquinas virtuales
```
gcloud compute instance-groups managed \
        set-named-ports web-server-group \
        --named-ports http:80 \
        --region $REGION
```
Este comando asigna un puerto lógico ("http") al puerto físico (80) en todas las máquinas virtuales del grupo de instancias (conjunto de VM que trabajan juntas como si fueran un solo sistema).
Es necesario para que el balanceador de carga sepa a qué puerto enviar el tráfico.
No cambia nada dentro de las VMs, solo las hace "descubribles" por el balanceador.

* ```set-named-ports```: Asigna un nombre lógico ("http") a un puerto real (80).
* ```web-server-group```: El grupo de instancias ((conjunto de máquinas virtuales que operan juntas como si se tratara de un solo sistema) donde se aplicará esta configuración.
* ```--named-ports http:80 ```: Le dice a GCP que el servicio llamado "http" está en el puerto 80 de todas las instancias (de cada máquina virtual).
* ```--region $REGION```: Se aplica a todas las VMs dentro de ese grupo en la región especificada.

Este comando es importante porque:
* le permite al balanceador de carga encontrar el puerto correcto en cada VM.
* Hace que las VMs dentro del grupo sean accesibles mediante HTTP (puerto 80).
* Facilita la administración de tráfico sin necesidad de configurar cada VM manualmente.

9. Crear un servicio de backend

```
gcloud compute backend-services create web-server-backend \
        --protocol HTTP \
        --http-health-checks http-basic-check \
        --global
```
Un Backend Service NO es un servidor físico ni una VM, sino una configuración que le dice al balanceador de carga cómo distribuir el tráfico entre las VMs. 
* --protocol HTTP : utiliza HTTP como protocolo.
* --http-health-checks http-basic-check : utiliza el health checks para saber qué VMs están funcionando correctamente.
* --global: servicio de backend está disponible en todas las regiones.

10. Agregar el grupo de instancias (VMs que trabajan juntas) al backend. 
Recordatorio: Un grupo de instancias es un conjunto de VMs gestionadas juntas, permitiendo escalabilidad y alta disponibilidad * Si el tráfico aumenta, puede crear más VMs ; Si el tráfico baja, puede eliminar VMs para ahorrar costos) 
```
gcloud compute backend-services add-backend web-server-backend \
        --instance-group web-server-group \
        --instance-group-region $REGION \
        --global
```
🔹 El web-server-backend es la configuración lógica que maneja el tráfico.
🔹 El web-server-group es el grupo de instancias (las VMs con NGINX).
🔹 --instance-group-region $REGION indica la región donde están las VMs.
🔹 --global confirma que el backend service es global.

Antes, creamos el Backend Service (web-server-backend), pero estaba vacío. 
Con este comando, le estamos diciendo qué VMs usar para responder tráfico.

 Le decimos al backend que su "backend real" (quien va a recibir las solicitudes HTTP) es el grupo de instancias.
📌 ¿Qué hace?
Añade el grupo de instancias (web-server-group) al backend para que las VMs reciban tráfico.

📌 Situación:
Un usuario escribe en su navegador: http://mi-app.com 
Esto envía una solicitud HTTP al balanceador de carga.
El balanceador de carga recibe la solicitud y su trabajo es decidir a qué servidor enviar la solicitud.
Pero el balanceador no se conecta directamente a las VMs, sino que primero consulta el Backend Service.
El Backend Service decide a qué VM enviar la solicitud. Es un intermediario entre el balanceador de carga y las VMs. Sabe qué VMs están sanas (gracias al Health Check). Distribuye la carga entre las VMs activas.
Finalmente, la solicitud llega a una VM. Cada VM tiene NGINX instalado (gracias al script de inicio). NGINX responde con la página web que el usuario quiere ver.

[ Usuario en navegador ]  
        │  
        ▼  
[ Balanceador de carga ] 
        │  
        ▼  
[ Backend Service ] 
        │  
        ▼  
[ Grupo de VMs ]  
    ├── VM 1 (con NGINX) ✅  
    ├── VM 2 (con NGINX) ✅  
    ├── VM 3 (con NGINX) ❌ (falló el health check)  
        │  
        ▼  
[ Respuesta enviada al usuario ]  


✅ NGINX es un servidor web en cada VM.
✅ El backend es el grupo de VMs que atienden tráfico HTTP.
✅ El balanceador de carga no habla directamente con las VMs, lo hace a través del Backend Service.
✅ El Backend Service decide cuál VM responde la solicitud.
✅ El Health Check mantiene el sistema funcionando solo con VMs activas.







