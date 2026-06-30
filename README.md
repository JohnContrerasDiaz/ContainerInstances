# Bienvenidos a Oracle Training Labs
## Laboratorio de Container Instances 
## Despliegue de Wordpress
#### El laboratorio consiste en el despliegue de Wordpress en una subred privada utilizando una Base de Datos MySQL creada en otro container dentro de la misma instancia. La seguridad lo mas importante !!! 

#### Para acceder a nuestro sistema Wordpress vamos a crear un Load Balancer publico protegido por un WAF (Web Appplication Firewall) como proteccion a posibles ciberataques

![](images/Arquitectura.jpg)

#### Prerrequisitos para la realizacion del Laboratorio
* Creacion de VCN y subredes, una publica y una privada
* Creacion de Internet Gateway y entrada en la tabla de rutas para la subred publica
* Creacion de NAT Gateway y entrada en la tabla de rutas para la subred privada
+ Configuracion de Security Lists:
  + Para la subred publica permitir el trafico HTTP por el puerto 80. No exponga MySQL (3306) a Internet.
  + Para la subred privada permitir todo el trafico desde la subred publica
  
# 1. Creación de Container Instance

### Menu principal >Developer Services > Container instances

![](images/create_container_instance.jpg)

## Configuración de la instancia
Debemos ingresar la informacion del nombre de la instancia, AD, Shape y capacidades de computo(OCPU y Memoria RAM)

![](images/create_container_instance1.jpg)

En la parte de Networking seleccionamos la VCN y la subred privada

![](images/create_ci_10.jpg)

### Configuración de los contenedores
En esta parte vamos a asignar los nombres de los contenedores, para ello seleccionamos las imagenes a utilizar y creamos las variables de ambiente que necesita el contenedor para funcionar adecuadamente. Para el laboratorio vamos a utilizar las imagenes publicas del Docker Hub

### El primer container a crear es el de MySQL
Asignamos un nombre al container y seleccionamos la imagen a descargar desde el Docker Hub

![](images/create_container_2.jpg)

Configuracion de las variables de ambiente necesarias para el despliegue del container MySQL
Click en crear another container

![](images/create_container_4.jpg)

### El segundo container a crear es el de Wordpress
Asignamos un nombre al container y seleccionamos la imagen a descargar desde el Docker Hub

![](images/create_container_5.jpg)

Configuracion de las variables de ambiente necesarias para el despliegue del container Wordpress
El valor de la variable WORDPRESS_DB_HOST corresponde a la IP seleccionada durante la creacion del Container Instance en la parte de Networking

![](images/create_ci_11.jpg)

Click en Create

![](images/create_container_7.jpg)

# 2. Creación del Load Balancer

#### Para acceder de forma publica al servicio de Wordpress es necesario configurar un Load Balancer para recibir el trafico desde internet

### Menu principal > Networking > Load Balancers

![](images/create_lb_1.jpg)

Asignamos el nombre del LB y seleccionamos las opciones de visibilidad publica e IP

![](images/create_lb_2.jpg)

Seleccionamos la VCN y la subred publica ceeada durante los prerrequisitos

![](images/create_lb_3.jpg)

Luego seleccionamos el algoritmo de balanceo del trafico y el protocolo y puerto para el  Check. En la imagen se muestra el  check utilizando el protocolo TCP/80 con codigo de respuesta 200.
Tambien se podria utilizar HTTP/80 con codigo de respuesta 302.

![](images/create_lb_4.jpg)

Configuramos el listener de tipo HTTP (las peticiones ingresan por el puerto 80)

![](images/create_lb_5.jpg)

Activamos los logs del LB

![](images/create_lb_6.jpg)

Adicionamos el Backend, este corresponde a la Container Instance

![](images/create_lb_7.jpg)

El backend corresponde a la IP privada del Container Instance

![](images/create_lb_8.jpg)


# Practica avanzada - de Monolito a Contenedor

# WordPress on Oracle Cloud Container Instances

![OCI](https://img.shields.io/badge/Oracle%20Cloud-OCI-red)
![Docker](https://img.shields.io/badge/Container-Docker-blue)
![WordPress](https://img.shields.io/badge/App-WordPress-blue)

Este lab muestra cómo **convertir una aplicación monolítica (WordPress + MySQL) en contenedores** y desplegarla en **Oracle Cloud Infrastructure (OCI)** utilizando:

- OCI Container Registry (OCIR)
- OCI Container Instances
- OCI Load Balancer
- Virtual Cloud Network (VCN)

---

# Arquitectura

```mermaid
flowchart TB

    User[Internet User]

    LB[OCI Load Balancer]

    subgraph VCN[Virtual Cloud Network]

        subgraph Subnet[Public Subnet]

            CI[Container Instance]

            WP[WordPress Container]
            DB[MySQL Container]

        end

    end

    User --> LB
    LB --> CI

    CI --> WP
    CI --> DB

    WP --> DB
```

Flujo de tráfico:

```
Internet
   │
OCI Load Balancer
   │
Container Instance
   ├── WordPress
   └── MySQL
```

---

# Prerrequisitos

Antes de comenzar necesitas:

- Docker instalado (recomendado utilizar Cloud Shell de OCI)
- Cuenta en docker hub
- Auth Token de OCI
- OCI CLI (opcional)

Servicios OCI utilizados:

- Container Registry
- Container Instances
- Load Balancer
- VCN

---

# Estructura del repositorio

```
oci-wordpress-demo/
│
├ Dockerfile.wordpress
├ Dockerfile.mysql
├ health.html
├ k8s/
│   └ wordpress-oke.yaml.template
│
└ README.md
```

---

# Health Check Endpoint

Archivo:

```
health.html
```

Contenido:

```html
OK
```

Este endpoint será utilizado por el **Load Balancer** para verificar la salud del contenedor.

---

# Dockerfile WordPress

Archivo:

```
Dockerfile.wordpress
```

Contenido:

```dockerfile
FROM docker.io/library/wordpress:php8.2-apache

COPY health.html /var/www/html/health.html

ENV WORDPRESS_DB_HOST=localhost
ENV WORDPRESS_DB_USER=wpuser
ENV WORDPRESS_DB_NAME=wpdb

EXPOSE 80
```

Explicación:

- Usa la imagen oficial de WordPress
- Agrega un endpoint de health check
- Expone el puerto 80

---

# Dockerfile MySQL

Archivo:

```
Dockerfile.mysql
```

Contenido:

```dockerfile
FROM docker.io/library/mysql:8.0

ENV MYSQL_DATABASE=wpdb
ENV MYSQL_USER=wpuser

EXPOSE 3306
```

Las contraseñas se configurarán en runtime mediante variables de entorno.

---

# Construir las imágenes

Hacer login en docker hub con los usuarios creados previamente:
- docker login docker.io

Desde el directorio raíz del proyecto ejecutar:

```bash
docker build -t wordpress-demo -f Dockerfile.wordpress .
docker build -t mysql-demo -f Dockerfile.mysql .
```

Verificar:

```bash
docker images
```

---

# Login en OCI Container Registry (OCIR)

Autenticarse contra el registry de OCI.

Formato:

```
docker login <region>.ocir.io
```

Ejemplo:

```bash
docker login iad.ocir.io
```

Usuario:

```
Obtener tenancy-namespace desde: Profile --> Tenancy --> Object storage namespace
<tenancy-namespace>/<domain>/<username>
ejemplo
axlcwp0sg7zw/user@mail.com
```

Password:

```
OCI Auth Token
```

El Auth Token se genera desde:

```
OCI Console
→ User Settings
→ Auth Tokens
```

---

# Taggear imágenes para OCIR

- Obtener tenancy-namespace desde: Profile --> Tenancy --> Object storage namespace

- Formato requerido por OCI:

```
<region>.ocir.io/<tenancy-namespace>/<repository>/<image>:<tag>
```

Ejemplo:

```bash
docker tag wordpress-demo iad.ocir.io/<tenancy-namespace>/demo/wordpress:1.0
docker tag mysql-demo iad.ocir.io/<tenancy-namespace>/demo/mysql:1.0
```

---

# Subir imágenes a OCIR


- Subir imágenes al registry:

```bash
docker push iad.ocir.io/<tenancy-namespace>/demo/wordpress:1.0
docker push iad.ocir.io/<tenancy-namespace>/demo/mysql:1.0
```

Las imágenes quedarán disponibles en:

```
OCI Console
→ Developer Services
→ Container Registry
```

---

# Creacion de Dynamic Group y Policy

Ir a:

```
Identity & Security
Domains (Compartment root) -> Default
→ Dynamic Groups
→ Crear DG con la siguiente clausula: ALL {resource.type = 'computecontainerinstance'}

Luego
Identity & Security
Policies -> crear policy con el texto: Allow dynamic-group NOMBRE-DG to read repos in compartment PoC-FGN

```


# Crear Container Instance

Ir a:

```
Developer Services
→ Container Instances
```

Crear una instancia con **dos contenedores**.

---

## MySQL Container

Imagen:

```
<region>.ocir.io/mytenancy/demo/mysql:1.0
```

Variables de entorno:

```
MYSQL_ROOT_PASSWORD=rootpassword
MYSQL_PASSWORD=wppassword
MYSQL_DATABASE=wpdb
MYSQL_USER=wpuser
```

Puerto:

```
3306
```

---

## WordPress Container

Imagen:

```
<region>.ocir.io/mytenancy/demo/wordpress:1.0
```

Variables:

```
WORDPRESS_DB_HOST=localhost
WORDPRESS_DB_USER=wpuser
WORDPRESS_DB_PASSWORD=wppassword
WORDPRESS_DB_NAME=wpdb
```

Puerto:

```
80
```

---

# Comunicación entre contenedores

Los contenedores dentro del mismo **Container Instance** comparten red.

WordPress puede conectarse a MySQL usando:

```
localhost:3306
```

---

# Crear Load Balancer

Ir a:

```
Networking
→ Load Balancers
```

Crear **Public Load Balancer**.

Configuración:

Listener:

```
Protocol: HTTP
Port: 80
```

Backend:

```
Target: Container Instance
Port: 80
```

---

# Configurar Health Check

Path:

```
/health.html
```

Expected response:

```
HTTP 200
```

Esto permite al Load Balancer verificar que WordPress está funcionando correctamente.

---

# Acceder a WordPress

Una vez desplegado el Load Balancer:

```
http://<public-ip>
```

Aparecerá el instalador de WordPress.

---

# Resultado final

Arquitectura desplegada:

```
Internet
   │
OCI Load Balancer
   │
Container Instance
   ├ WordPress
   └ MySQL
```

---

# Mejoras para producción

Para un entorno productivo se recomienda:

- Base de datos gestionada (OCI MySQL DB System)
- Persistencia de almacenamiento
- HTTPS con certificados TLS
- Gestión segura de secretos
- Observabilidad y logging
- Escalamiento horizontal

---

# Parte final: WordPress como microservicios en OKE

En esta parte se reutilizan las imagenes de WordPress y MySQL publicadas en OCIR y se despliegan como cargas separadas en Oracle Kubernetes Engine (OKE). MySQL queda accesible solamente dentro del cluster y WordPress se publica mediante un `Service` de tipo `LoadBalancer`.

> Este laboratorio usa dos `Deployments` con una replica cada uno y volumenes persistentes de bloque. Es adecuado para aprendizaje, no representa por si solo una arquitectura WordPress altamente disponible.

## 1. Crear el cluster con el wizard Quick Create

1. En OCI Console abra **Developer Services > Kubernetes Clusters (OKE)**.
2. Seleccione **Create cluster**.
3. En el wizard seleccione **Quick Create** y luego **Proceed**.
4. Defina el nombre `oke-wordpress`, el compartimento del laboratorio y una version soportada de Kubernetes que no sea *preview*.
5. Para ejecutar el laboratorio desde OCI Cloud Shell, seleccione un **Kubernetes API endpoint publico**. Para un endpoint privado se requiere conectividad privada o Bastion.
6. Seleccione **Managed nodes**, una shape disponible y al menos un nodo. Revise el costo estimado antes de crear.
7. Seleccione **Next**, revise los recursos y seleccione **Create cluster**.
8. Espere hasta que el cluster y el node pool aparezcan en estado **Active**. Quick Create crea tambien la VCN, gateways, subred de workers y subred de load balancers.

Flujo didactico del laboratorio:

![Flujo didactico del laboratorio WordPress en OKE](images/flujo-laboratorio-wordpress-oke.png)

## 2. Conectar OCI Cloud Shell al cluster

En la pagina del cluster seleccione **Access cluster > Cloud Shell Access > Launch Cloud Shell**. Copie y ejecute el comando presentado por OCI, similar a:

```bash
oci ce cluster create-kubeconfig \
  --cluster-id <cluster-ocid> \
  --file "$HOME/.kube/config" \
  --region <region> \
  --token-version 2.0.0 \
  --kube-endpoint PUBLIC_ENDPOINT

kubectl get nodes
kubectl get storageclass oci-bv
```

Todos los nodos deben estar `Ready` y la clase de almacenamiento `oci-bv` debe existir antes de continuar.

## 3. Preparar las variables del despliegue

Clone este repositorio en Cloud Shell y ejecute desde su raiz:

```bash
cp .env.oke.example .env.oke
chmod 600 .env.oke
vi .env.oke
```

Complete los valores reales de OCIR. `OCIR_USERNAME` usa el formato mostrado en el perfil de OCI; si utiliza un dominio de identidad, normalmente es `dominio/usuario`. En los comandos siguientes se antepone automaticamente `<tenancy-namespace>/` al crear el secreto de OCIR. Para evitar errores durante la practica, conserve `MYSQL_ROOT_PASSWORD=rootpassword` y `WORDPRESS_DB_PASSWORD=wppassword`.

Importe las variables en la sesion:

```bash
set -a
source .env.oke
set +a
```

No comparta ni confirme `.env.oke`; contiene el Auth Token de OCI y contrasenas de base de datos.

Valide que no queden marcadores y que las imagenes existan antes de crear recursos:

```bash
if grep -Eq '^[A-Z_]+=.*<[^>]+>' .env.oke; then
  echo "ERROR: complete todos los valores <...> de .env.oke"
  return 1 2>/dev/null || exit 1
fi

printf '%s' "$OCIR_AUTH_TOKEN" | docker login "${OCIR_REGION_KEY}.ocir.io" \
  --username "${OCIR_TENANCY_NAMESPACE}/${OCIR_USERNAME}" \
  --password-stdin

docker pull "$WORDPRESS_IMAGE"
docker pull "$MYSQL_IMAGE"
```

## 4. Revisar y desplegar

Primero renderice el manifiesto reemplazando las rutas de las imagenes de OCIR:

```bash
sed \
  -e "s|__WORDPRESS_IMAGE__|${WORDPRESS_IMAGE}|g" \
  -e "s|__MYSQL_IMAGE__|${MYSQL_IMAGE}|g" \
  k8s/wordpress-oke.yaml.template \
  > k8s/wordpress-oke.rendered.yaml

kubectl apply --dry-run=client -f k8s/wordpress-oke.rendered.yaml
```

Revise las imagenes renderizadas y confirme que Cloud Shell apunta al cluster correcto:

```bash
grep 'image:' k8s/wordpress-oke.rendered.yaml
kubectl config current-context
kubectl get nodes
```

Cree el namespace del laboratorio:

```bash
kubectl create namespace "$OKE_NAMESPACE" \
  --dry-run=client -o yaml | kubectl apply -f -
```

Cree el secreto para descargar las imagenes privadas desde OCIR:

```bash
kubectl -n "$OKE_NAMESPACE" create secret docker-registry ocir-secret \
  --docker-server="${OCIR_REGION_KEY}.ocir.io" \
  --docker-username="${OCIR_TENANCY_NAMESPACE}/${OCIR_USERNAME}" \
  --docker-password="$OCIR_AUTH_TOKEN" \
  --docker-email="$OCIR_EMAIL" \
  --dry-run=client -o yaml | kubectl apply -f -
```

Cree el secreto con las variables educativas de WordPress y MySQL:

```bash
kubectl -n "$OKE_NAMESPACE" create secret generic wordpress-db \
  --from-literal=WORDPRESS_DB_NAME="$WORDPRESS_DB_NAME" \
  --from-literal=WORDPRESS_DB_USER="$WORDPRESS_DB_USER" \
  --from-literal=WORDPRESS_DB_PASSWORD="$WORDPRESS_DB_PASSWORD" \
  --from-literal=MYSQL_ROOT_PASSWORD="$MYSQL_ROOT_PASSWORD" \
  --dry-run=client -o yaml | kubectl apply -f -
```

Los secretos se crean directamente en Kubernetes y no se escriben en el manifiesto. Despliegue los recursos:

```bash
kubectl -n "$OKE_NAMESPACE" apply --dry-run=server \
  -f k8s/wordpress-oke.rendered.yaml

kubectl -n "$OKE_NAMESPACE" apply \
  -f k8s/wordpress-oke.rendered.yaml

kubectl -n "$OKE_NAMESPACE" rollout status deployment/mysql --timeout=10m
kubectl -n "$OKE_NAMESPACE" rollout status deployment/wordpress --timeout=10m
```

## 5. Verificar WordPress

```bash
kubectl -n "$OKE_NAMESPACE" get pods,svc,pvc
kubectl -n "$OKE_NAMESPACE" get events --sort-by=.lastTimestamp
kubectl -n "$OKE_NAMESPACE" get svc wordpress -w
```

Cuando `EXTERNAL-IP` tenga un valor, abra:

```text
http://<EXTERNAL-IP>
```

Si aparece `ImagePullBackOff`, valide la ruta de las imagenes, el region key de OCIR y el Auth Token. Si un PVC permanece `Pending`, valide que el cluster tenga la clase `oci-bv` y capacidad/cuota de Block Volume.

## 6. Arquitectura desplegada

![Arquitectura WordPress en Oracle Kubernetes Engine](images/arquitectura-wordpress-oke.png)

Los archivos fuente SVG tambien estan disponibles en `images/` para reutilizarlos sin perdida de calidad.

## 7. Limpieza

Para eliminar las cargas y los load balancers creados por el manifiesto:

```bash
kubectl delete namespace "$OKE_NAMESPACE"
```

Revise en OCI Console que el Load Balancer y los Block Volumes hayan sido eliminados. El cluster OKE y la red creados por Quick Create se eliminan por separado desde OCI Console cuando termine el laboratorio.

## Referencias oficiales

- [Crear un cluster OKE con Quick Create](https://docs.oracle.com/en-us/iaas/Content/ContEng/Tasks/contengcreatingclusterusingoke_topic-Using_the_Console_to_create_a_Quick_Cluster_with_Default_Settings.htm)
- [Configurar acceso al cluster](https://docs.oracle.com/en-us/iaas/Content/ContEng/Tasks/contengdownloadkubeconfigfile.htm)
- [Consumir imagenes de OCIR desde OKE](https://docs.oracle.com/en-us/iaas/Content/ContEng/Tasks/contengpullingimagesfromocir.htm)
- [Provisionar PVC con OCI Block Volume](https://docs.oracle.com/en-us/iaas/Content/ContEng/Tasks/contengcreatingpersistentvolumeclaim_topic-Provisioning_PVCs_on_BV.htm)
