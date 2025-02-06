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

