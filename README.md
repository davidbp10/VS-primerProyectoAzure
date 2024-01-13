# Azure + GitLab CI/CD Documentation

## Pipeline Azure Práctica 1.
Esta documentación proporciona una descripción general de cómo integrar Azure con GitLab CI/CD mediante archivos de configuración específicos. La integración implica configurar un entorno Docker para una aplicación Node.js, usar Docker Compose para definir y ejecutar una aplicación de múltiples contenedores y configurar un archivo YAML de Azure Pipelines para automatizar el proceso de implementación.

### Dockerfile Explanation

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

### Docker-Compose Explanation

El archivo `docker-compose.yml` define los servicios que componen la aplicación. En este ejemplo, especifica un único servicio para la aplicación web.

```yaml
version: '3'               # Specify the version of the Compose file format
services:
  web:
    build: .               # Build the image using the Dockerfile in the current directory
    ports:
      - "3000:3000"        # Map port 3000 of the host to port 3000 in the container
```

### Azure Pipelines YAML Explanation

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


## Pipeline Azure Práctica 2.

Esta documentación describe el proceso de configuración de un pipeline de integración e implementación continuas (CI/CD) mediante Azure Pipelines para trabajar con Terraform para la administración de infraestructura. El pipeline automatiza la implementación y administración de recursos en Azure mediante archivos de configuración de Terraform.

### Ficheros de configuración de Terraform (`*.tf`)

#### `main.tf`

Este archivo define los proveedores necesarios y declara recursos para imágenes de Docker y redes de Docker.

- **Providers**: la configuración de Terraform especifica la versión del proveedor de Docker.
- **Docker Images**: Se definen los recursos para las imágenes Docker `mariadb` y `wordpress`, especificando los nombres de las imágenes y la decisión de no mantenerlas localmente.
- **Docker Network**: se crea un recurso de red Docker llamado `my_network`.
- **Docker Containers**: se definen dos recursos de contenedores Docker para `mariadb` y `wordpress`, utilizando las imágenes y la red de Docker definidas anteriormente. Están configurados con variables de entorno y volúmenes para el almacenamiento de datos persistente.

#### `variables.tf`

Este archivo declara variables utilizadas en la configuración de Terraform con valores predeterminados:

- **`MYSQL_ROOT_PASSWORD`**: La contraseña de root para la base de datos MariaDB.
- **`MYSQL_DATABASE`**: El nombre de la base de datos MySQL.
- **`db_container_name`**: El nombre del contenedor para MariaDB.
- **`wp_container_name`**: El nombre del contenedor para WordPress.

## Pipeline de configuración Azure (`azure-pipelines.yml`)

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