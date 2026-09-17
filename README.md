---

# Guía de Despliegue y Configuración: Entorno de Validación SAP

Este documento describe el procedimiento completo para inicializar desde cero el entorno de ejecución del validador de datos SAP en un servidor Ubuntu Linux, configurando la sincronización con Google Drive, autenticación GCP/BigQuery y el contenedor Docker de procesamiento.

---

## Requisitos Previos

* Servidor con **Ubuntu Server 22.04 LTS** o superior.
* Acceso con privilegios de `sudo`.
* Cuenta de Google Workspace/GCP con permisos en el proyecto de BigQuery e insumos en Google Drive.

---

## 1. Instalación de Dependencias Base del Sistema

Actualizar el sistema e instalar utilerías generales:

```bash
sudo apt-get update && sudo apt-get upgrade -y
sudo apt-get install -y curl wget git unzip ca-certificates gnupg apt-transport-https fuse3

```

---

## 2. Instalación y Configuración de `rclone` (Google Drive)

### 2.1. Instalación de rclone

```bash
sudo curl https://rclone.org/install.sh | sudo bash

```

### 2.2. Configuración del remoto gdrive

Ejecutar el asistente interactivo:

```bash
rclone config

```

1. Presionar `n` para un nuevo remoto.
2. Nombre del remoto: `gdrive`
3. Tipo de almacenamiento: Seleccionar `drive` (Google Drive).
4. Dejar `client_id` y `client_secret` en blanco (presionar Enter).
5. Scope: Seleccionar opción `1` (`drive` - acceso completo).
6. Seguir las instrucciones para autenticar mediante el navegador.

### 2.3. Crear punto de montaje y ejecutar

```bash
# Crear directorio local de montado
mkdir -p ~/google_drive

# Habilitar 'allow_other' en la configuración de FUSE
sudo sed -i 's/#user_allow_other/user_allow_other/g' /etc/fuse.conf

# Montar Google Drive en segundo plano
rclone mount gdrive: ~/google_drive \
  --vfs-cache-mode full \
  --vfs-cache-max-age 1h \
  --allow-other \
  --daemon

```

---

## 3. Instalación y Configuración de Google Cloud CLI (`gcloud`)

### 3.1. Instalación del SDK de Google Cloud

```bash
# Agregar la clave pública de GCP y el repositorio
curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo gpg --dearmor -o /usr/share/keyrings/cloud.google.gpg
echo "deb [signed-by=/usr/share/keyrings/cloud.google.gpg] https://packages.cloud.google.com/apt cloud-sdk main" | sudo tee -a /etc/apt/sources.list.d/google-cloud-sdk.list

# Instalar gcloud CLI
sudo apt-get update && sudo apt-get install -y google-cloud-cli

```

### 3.2. Generar Application Default Credentials (ADC)

```bash
# Autenticar usuario para librerías de Python/BigQuery
gcloud auth application-default login --no-launch-browser

# Configurar el proyecto de cuota por defecto (Ejemplo para QAS)
gcloud auth application-default set-quota-project crp-qas-dominio-logistica

```

*Copiar la URL mostrada, abrirla en el navegador con la cuenta institucional de GCP y pegar el código de verificación resultante.*

---

## 4. Instalación de Docker Engine

```bash
# Descargar e instalar Docker automáticamente
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Agregar el usuario actual al grupo de docker (evita usar 'sudo docker')
sudo usermod -aG docker $USER

# Aplicar cambios de grupo (o reiniciar la sesión SSH)
newgrp docker

```

---

## 5. Construcción y Ejecución del Contenedor

### 5.1. Construir la Imagen Docker

Desde la raíz del proyecto (`~/validator`):

```bash
docker build -t data_validator:v1 .

```

### 5.2. Despliegue del Contenedor de Desarrollo/Producción

Para correr el contenedor mapeando el usuario local, el código fuente, la carpeta de Google Drive y las credenciales de BigQuery:

```bash
docker run -it --name dev_container \
  --user $(id -u):$(id -g) \
  -v $(pwd):/workspace \
  -v ~/google_drive:/workspace/drive \
  -v ~/.config/gcloud:/tmp/gcloud \
  -e GOOGLE_APPLICATION_CREDENTIALS=/tmp/gcloud/application_default_credentials.json \
  data_validator:v1

```

---

## 6. Mantenimiento Diarios y Uso Posterior

### Reingresar al contenedor existente (sin volver a crearlo):

```bash
docker start -ai dev_container

```

### Ejecutar prueba rápida de salud (BigQuery + Drive):

Dentro del contenedor:

```bash
python -c "
from src.config import obtener_cliente_bigquery
bq_client, cfg = obtener_cliente_bigquery('QAS')
print('[HEALTH OK] BigQuery Conectado a:', cfg['pid'])
"

```

---
