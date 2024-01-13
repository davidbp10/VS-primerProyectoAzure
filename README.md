# Azure + GitLab CI/CD Documentation

Participantes de la pareja:
- David Brea Piñero.
- Jesús Alberto Mellado Montes.

## 1. Pipeline Azure Práctica 1.
Esta documentación proporciona una descripción general de cómo integrar Azure con GitLab CI/CD mediante archivos de configuración específicos. La integración implica configurar un entorno Docker para una aplicación Node.js, usar Docker Compose para definir y ejecutar una aplicación de múltiples contenedores y configurar un archivo YAML de Azure Pipelines para automatizar el proceso de implementación.

### Explicación Dockerfile

El Dockerfile proporcionado configura un entorno Node.js, instala dependencias y define el comando para ejecutar la aplicación.

```dockerfile
FROM node:14                 # Use the official Node.js 14 image as the base image
WORKDIR /app                # Set the working directory inside the container
COPY package*.json ./       # Copy package.json and package-lock.json to the working directory
RUN npm install             # Install the project dependencies defined in package.json
COPY . .                    # Copy the rest of the application's source code to the working directory
EXPOSE 3000                 # Make port 3000 available to the outside of the container
CMD ["npm", "start"]        # Define the command to run the application
```

### Explicación Docker-Compose

El archivo `docker-compose.yml` define los servicios que componen la aplicación. En este ejemplo, especifica un único servicio para la aplicación web.

```yaml
version: '3'               # Specify the version of the Compose file format
services:
  web:
    build: .               # Build the image using the Dockerfile in the current directory
    ports:
      - "3000:3000"        # Map port 3000 of the host to port 3000 in the container
```

### Explicación Pipeline de Azure

El archivo `azure-pipelines.yml` define la canalización de CI/CD para crear, enviar e implementar la imagen de Docker en Azure.

```yaml
trigger:
  branches:
    include:
      - main                # Trigger the pipeline on commits to the main branch

pool:
  vmImage: 'ubuntu-latest' # Use the latest Ubuntu VM image for the build

steps:
- script: echo 'Iniciando Docker Build y Compose'
  displayName: 'Iniciando Docker Build y Compose'

- task: Docker@2
  displayName: 'Construir Imagen Docker'
  inputs:
    command: 'build'
    Dockerfile: '**/Dockerfile'
    tags: 'latest'         # Tag the built image as 'latest'

- task: Docker@2
  displayName: 'Subir Imagen Docker al Registro'
  inputs:
    command: 'push'
    tags: 'latest'         # Push the 'latest' tag to the registry
    dockerRegistryServiceConnection: 'vsazure.azurecr.io' # Define the Azure Container Registry connection

- task: Docker@2
  displayName: 'Docker Compose Up'
  inputs:
    command: 'composeUp'
    dockerComposeFile: '**/docker-compose.yml'   # Use the docker-compose file for deployment
    removeContainersOnPull: true
    detachedService: true                        # Run services in the background
```

El pipeline automatiza los siguientes pasos:

- Hace eco de un mensaje de inicio para la compilación y redacción de Docker.
- Crea la imagen de Docker usando Dockerfile.
: envía la imagen creada al Azure Container Registry especificado.
- Utiliza Docker Compose para implementar la aplicación de múltiples contenedores en Azure Container Instances.


## 2. Pipeline Azure Práctica 2.

Esta documentación describe el proceso de configuración de un pipeline de integración e implementación continuas (CI/CD) mediante Azure Pipelines para trabajar con Terraform para la administración de infraestructura. El pipeline automatiza la implementación y administración de recursos en Azure mediante archivos de configuración de Terraform.

### Ficheros de configuración de Terraform (`*.tf`)

#### `main.tf`

```tf
terraform {
    required_providers {
        docker = {
            source = "kreuzwerker/docker"
            version = "~> 3.0.1"
        }
    }
}

// Definir proveedor de Docker
provider "docker" {}

resource "docker_image" "mariadb" {
    name = "mariadb:latest"
    keep_locally = false
}

resource "docker_image" "wordpress" {
    name = "wordpress:latest"
    keep_locally = false
}

// Definir la red de Docker
resource "docker_network" "my_network" {
    name = "my_network"
}

// Definir el contenedor de MariaDB
resource "docker_container" "mariadb" {
    image = docker_image.mariadb.image_id
    name  = "${var.db_container_name}"
    networks_advanced {
        name = "${docker_network.my_network.name}"
    }
    env = [
        "MYSQL_ROOT_PASSWORD=${var.MYSQL_ROOT_PASSWORD}",
        "MYSQL_DATABASE=${var.MYSQL_DATABASE}",
    ]
    volumes {
        container_path  = "/var/lib/mysql"
        volume_name     = docker_volume.my_volume.name
        read_only       = false
    }
}

// Definir el contenedor de Wordpress
resource "docker_container" "wordpress" {
    image = docker_image.wordpress.image_id
    name  = "${var.wp_container_name}"
    ports {
        internal = 80
        external = 8080
    }
    networks_advanced {
        name = "${docker_network.my_network.name}"
    }
    env = [
        "WORDPRESS_DB_HOST=${docker_container.mariadb.name}",
        "WORDPRESS_DB_USER=root",
        "WORDPRESS_DB_PASSWORD=${var.MYSQL_ROOT_PASSWORD}",
        "WORDPRESS_DB_NAME=${var.MYSQL_DATABASE}",
    ]
}

// Definir el volumen de Docker
resource "docker_volume" "my_volume" {
    name = "my_volume"
}

```

Este archivo define los proveedores necesarios y declara recursos para imágenes de Docker y redes de Docker.

- **Providers**: la configuración de Terraform especifica la versión del proveedor de Docker.
- **Docker Images**: Se definen los recursos para las imágenes Docker `mariadb` y `wordpress`, especificando los nombres de las imágenes y la decisión de no mantenerlas localmente.
- **Docker Network**: se crea un recurso de red Docker llamado `my_network`.
- **Docker Containers**: se definen dos recursos de contenedores Docker para `mariadb` y `wordpress`, utilizando las imágenes y la red de Docker definidas anteriormente. Están configurados con variables de entorno y volúmenes para el almacenamiento de datos persistente.

#### `variables.tf`

```tf
variable "MYSQL_ROOT_PASSWORD" {
    description = "La contraseña del usuario root de MySQL"
    type        = string
    default     = "password"
}

variable "MYSQL_DATABASE" {
    description = "El nombre de la base de datos de MySQL"
    type        = string
    default     = "wordpress"
}

variable "db_container_name" {
    description = "El nombre del contenedor de bd"
    type        = string
    default     = "mariadb"
}

variable "wp_container_name" {
    description = "El nombre del contenedor de wp"
    type        = string
    default     = "wordpress"
}
```

Este archivo declara variables utilizadas en la configuración de Terraform con valores predeterminados:

- **`MYSQL_ROOT_PASSWORD`**: La contraseña de root para la base de datos MariaDB.
- **`MYSQL_DATABASE`**: El nombre de la base de datos MySQL.
- **`db_container_name`**: El nombre del contenedor para MariaDB.
- **`wp_container_name`**: El nombre del contenedor para WordPress.

### Pipeline de configuración Azure (`azure-pipelines.yml`)

```yml
trigger:
- master

pool:
    vmImage: 'ubuntu-latest'

steps:
- task: UseDotNet@2
    inputs:
        packageType: 'sdk'
        version: '3.1.x'
        installationPath: $(Agent.ToolsDirectory)/dotnet

- script: |
        terraform init
        terraform validate
    displayName: 'Terraform Init and Validate'

- script: 'terraform plan -out=tfplan'
    displayName: 'Terraform Plan'

- script: 'terraform apply -auto-approve tfplan'
    displayName: 'Terraform Apply'
```

El archivo `azure-pipelines.yml` define el pipeline de CI/CD en Azure DevOps:

- **Trigger**: el pipeline se activa al confirmarse en la rama `master`.
- **Pool**: especifica el uso de la última imagen de VM de Ubuntu.
- **Pasos**:
   - **NET Core SDK**: Instala una versión específica del SDK de .NET Core necesaria para la canalización.
   - **Terraform Tool Installer**: Instala una versión de Terraform específica.
   - **Terraform Init**: Inicializa un directorio de trabajo de Terraform nuevo o existente realizando varios pasos de inicialización.
   - **Terraform Validate**: valida la corrección de los archivos de Terraform.
   - **Terraform Plan**: Crea un plan de ejecución, determinando qué acciones son necesarias para lograr el estado deseado especificado en los archivos de configuración.
   - **Terraform Apply**: Aplica los cambios descritos por el plan para alcanzar el estado deseado de la configuración.

Durante la ejecución del pipeline, se ejecutan comandos de Terraform para planificar y aplicar cambios en la infraestructura. Se recomienda almacenar el archivo de estado de Terraform en un backend remoto como Azure Blob Storage para compartirlo y bloquearlo durante las operaciones para evitar modificaciones de estado simultáneas.

## 3. Pipeline Gitlab
Esta documentación proporciona una descripción general de como configurar un proceso de Integración Continua (CI) y Despliegue Continuo (CD) con GitLab. Este proyecto implica la transformación de datos, generación de gráficos, documentación y publicación en GitHub Pages.

### Explicación script_grafica.py
El script proporcionado a continuación realiza una gráfica simple de los datos que encontramos en el archivo proporcionado a través del campus virtual `SensorData.csv`. Tanto el fichero `SensorData.csv` como el script `script_grafica.py` deben estar en nuestro repositorio de Gitlab y en el mismo directorio.

```py
import matplotlib.pyplot as plt
import pandas as pd

# Leer los datos del archivo CSV
df = pd.read_csv('SensorData.csv')

# Especificar las columnas que deseas graficar, por ejemplo 'timestamp' y 'temperatureSHT31'
# Asegúrate de reemplazar estas columnas con los nombres de columna correctos de tu archivo CSV
x = df['timestamp']
y = df['temperatureSHT31']

# Crear la gráfica
plt.plot(x, y)
plt.title('Gráfico de SensorData')
plt.xlabel('Timestamp')
plt.ylabel('Temperatura SHT31')

# Guardar la gráfica como una imagen
plt.savefig('grafica_ejemplo.png')

# Mostrar la gráfica (opcional)
plt.show()
```


### Explicación del Pipeline de GitLab
El archivo gitlab-pipelines.yml es un archivo de configuración de GitLab CI/CD  en formato YAML. Este archivo genera la gráfica de los datos, la documentación a partir del archivo Markdown README. del repositorio, incrusta en ella la gráfica de los datos y por último genera un artefacto en formato HTML a partir del `README.md`.
```yaml
image: python:3.9

stages:
  - plot
  - deploy

before_script:
  - pip install --upgrade pip
  - pip install matplotlib numpy pandas

plot:
  stage: plot
  script:
    - python script_grafica.py  # Este script genera grafica_ejemplo.png
  artifacts:
    paths:
      - grafica_ejemplo.png  # Esto asegura que grafica_ejemplo.png esté disponible para la siguiente etapa

deploy:
  stage: deploy
  script:
    # Instala pandoc
    - apt-get update && apt-get install -y pandoc
    # Genera el HTML del Markdown
    - pandoc README.md --self-contained -o documentation.html #Para que se incruste la grafica, debemos añadir <![](grafica_ejemplo.png)> en el README.md
  artifacts:
    paths:
      - documentation.html  # The HTML file now includes the embedded image
```

Antes de realizar cada trabajo en el pipeline se ejecutará la configuración previa (`before_script`) necesaria para ejecutar el script de generación de gráficos.
El pipeline automatiza los siguientes pasos:
- Crea la gráfica usando el script de python `script_grafica.py` y asegura que el fichero `grafica_ejemplo.png` esté disponible para las siguientes etapas.
- Genera el artefaco Html a partir del Markdown y de la imagen.