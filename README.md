Control de versiones y CI/CD – Parte II

1. Comprobación local

Primero se ejecuta la aplicación en local sin Docker para asegurarse de que funciona correctamente.

En este paso:

Se crea el entorno virtual de Python.

Se instalan las dependencias.

Se arranca la aplicación.

Se comprueba que responde correctamente.

Ejemplo básico:

python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
python src/app.py

2. Creación de la imagen Docker

Cuando la aplicación funciona en local, se crea una imagen Docker para poder ejecutarla en cualquier entorno.

En este paso:

Se define el Dockerfile.

Se construye la imagen.

Se prueba el contenedor.

Comando principal:

docker build -t usuario/galeria:latest .

3. Ejecución con Docker Compose

Después se utiliza Docker Compose para ejecutar el contenedor de forma más sencilla.

En este paso:

Se crea el archivo docker-compose.yml.

Se levanta el servicio con Docker Compose.

Se comprueba que funciona correctamente.

Comando principal:

docker compose up -d

4. Automatización con GitHub Actions

A continuación se configura un workflow en GitHub para automatizar la construcción de la imagen Docker.

El workflow hace lo siguiente:

Detecta cambios en el repositorio.

Construye la imagen Docker.

La sube automáticamente a DockerHub.

Esto permite no tener que construir ni subir la imagen manualmente.

El proceso se activa cada vez que se hace:

git push

5. Configuración de Secrets en GitHub

Para que GitHub pueda subir imágenes a DockerHub y conectarse al servidor, necesita credenciales.
Estas credenciales no se ponen en el código por seguridad, sino que se guardan como Secrets en el repositorio.

Los principales secrets son:

Para DockerHub

DOCKERHUB_USERNAME: nombre de usuario de DockerHub.

DOCKERHUB_TOKEN: token de acceso de DockerHub.

Estos permiten que GitHub pueda iniciar sesión y subir la imagen.

Para AWS

AWS_USERNAME: usuario de la máquina (por ejemplo, ubuntu).

AWS_HOSTNAME: IP pública o dominio del servidor.

AWS_PRIVATEKEY: clave privada SSH para conectarse.

Estos permiten que GitHub se conecte al servidor y actualice la aplicación.

6. Despliegue automático en AWS

Se añade un segundo paso al workflow para desplegar la aplicación en el servidor.

Cuando se hace push:

GitHub construye la imagen.

La sube a DockerHub.

Se conecta por SSH al servidor.

Descarga la nueva imagen.

Reinicia los contenedores con la versión actualizada.

El servidor queda actualizado automáticamente sin intervención manual.

Flujo completo del sistema

El funcionamiento general es:

Se modifica el código.

Se hace git push.

GitHub construye la imagen Docker.

La imagen se sube a DockerHub.

GitHub se conecta al servidor.

El servidor descarga la nueva imagen.

Se reinicia el contenedor.

La aplicación queda actualizada
