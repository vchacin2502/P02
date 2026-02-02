# Control de versiones y CI/CD - Parte II

## 1. Preparación del Entorno

Antes de comenzar, asegúrate de reiniciar los servicios necesarios en tu máquina virtual o entorno local:

sudo systemctl restart docker.service 
sudo systemctl restart networking.service

## 2. Configuración y Prueba Local de la Aplicación

### 2.1. Entorno Python

Primero, verificamos que la aplicación funciona(sin Docker).

1. **Crear y activar el entorno virtual:**
   
    python3 -m venv venv
    source venv/bin/activate

3. **Instalar dependencias:**
   
    pip install -r requirements.txt

5. **Iniciar la aplicación:**

        python src/app.py

6. **Verificar funcionamiento:**
   
    Abre tu navegador o usa curl:
    curl http://localhost:5000/status

### 2.2. Creación de Imagen Docker

Dockerizamos la aplicación para asegurar que corra igual en cualquier entorno.

1. **Crear el `Dockerfile`:**
   
    Asegúrate de tener el archivo con el siguiente contenido:

    ```dockerfile
    # start by pulling the python image
    FROM python:3-slim

    # switch working directory
    WORKDIR /app

    # copy the requirements file into the image
    COPY ./requirements.txt ./

    # install the dependencies and packages in the requirements file
    RUN pip install -r requirements.txt

    # copy every content from the local file to the image
    COPY src .

    # configure the container to run in an executed manner
    CMD gunicorn --bind 0.0.0.0:5000 app:app
    ```

3. **Construir la imagen:**
   
    Reemplaza `<usuariodockerhub>` con tu usuario real.

    docker build -t <usuariodockerhub>/galeria:latest .

5. **Probar el contenedor:**

    docker run -d -p 80:5000 --name testgaleria <usuariodockerhub>/galeria

### 2.3. Orquestación con Docker Compose

Usaremos `docker-compose` para simplificar la ejecución.

1. **Crear/Modificar `docker-compose.yml`:**
    
    ```yaml
    services:
      galeria:
        container_name: galeria
        image: <usuariodockerhub>/galeria:latest
        ports:
          - 80:5000
        restart: always
    ```

2. **Ejecutar:**

    docker compose up -d

3. **Subida manual a DockerHub (Prueba):**

    docker push <usuariodockerhub>/galeria:latest

## 3. Automatización con GitHub Actions (Build & Push)

Configuraremos GitHub para que construya y suba la imagen automáticamente al detectar cambios.

### 3.1. Configuración de Secretos

Para que GitHub tenga permiso de subir imágenes a tu DockerHub, necesitas configurar **Secrets** en el repositorio (`Settings` -> `Security` -> `Secrets and Variables` -> `Actions`).

* `DOCKERHUB_USERNAME`: Tu nombre de usuario.
* `DOCKERHUB_TOKEN`: Un Access Token generado en DockerHub 

### 3.2. Crear el Workflow

Crea un archivo en `.github/workflows/docker-ci.yml` con el siguiente contenido. Nota que se activa al haber cambios en la carpeta `src`.

```yaml
name: Docker CI/CD

on:
  push:
    branches: [ main ]
    paths:
      - src/**
  pull_request:
    branches: [ main ]
    paths:
      - src/**

jobs:
  build:
    name: Build and Push Docker image
    runs-on: ubuntu-latest
    steps:
      # 1. Descargar el código
      - uses: actions/checkout@v4
      
      # 2. Loguearse en DockerHub
      - name: Login to DockerHub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
          
      # 3. Construir y subir imagen
      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ secrets.DOCKERHUB_USERNAME }}/galeria:latest
```

## 4. Despliegue Automático en AWS

Añadiremos una etapa final para desplegar la nueva imagen en una instancia EC2.

### 4.1. Requisitos y Secretos AWS

Necesitas una instancia EC2 con Docker instalado. Añade estos secretos en GitHub:

* `AWS_USERNAME`: Generalmente `admin` (Debian) o `ubuntu` (Ubuntu).
* `AWS_HOSTNAME`: La IP pública o DNS de tu instancia.
* `AWS_PRIVATEKEY`: El contenido completo de tu archivo `.pem` (clave privada SSH).

### 4.2. Actualizar el Workflow

```yaml
  # ... (código anterior del job build) ...

  aws:
    name: Deploy image to AWS
    needs: build  # Espera a que termine el build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      # 1. Copiar el fichero docker-compose al servidor
      - name: Copy docker compose via SCP
        uses: appleboy/scp-action@v0.1.7
        with:
          host: ${{ secrets.AWS_HOSTNAME }}
          username: ${{ secrets.AWS_USERNAME }}
          port: 22
          key: ${{ secrets.AWS_PRIVATEKEY }}
          source: "docker-compose.yml"
          target: "/home/${{ secrets.AWS_USERNAME }}"

      # 2. Conectarse por SSH y desplegar
      - name: Deploy Docker services via SSH
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.AWS_HOSTNAME }}
          username: ${{ secrets.AWS_USERNAME }}
          key: ${{ secrets.AWS_PRIVATEKEY }}
          port: 22
          script: |
            # Pausa breve para asegurar que DockerHub procesó la subida
            sleep 60 
            # Detener y borrar contenedores e imágenes viejas
            docker compose down --rmi all
            # Levantar el servicio con la imagen nueva
            docker compose up -d
```

### 4.3. Prueba Final

1.  Modifica el archivo `src/index.html` (ej. pon tu nombre en el título).
2.  Haz `git add`, `git commit` y `git push`.
3.  Observa en GitHub Actions cómo se ejecuta `build` y luego `aws`.
4.  Entra a la IP de tu AWS y verifica que el cambio está publicado.
