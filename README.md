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