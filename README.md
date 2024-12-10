# Despliegue de MLFlow en GCP

Esta guía proporciona instrucciones paso a paso para configurar los servicios necesarios para desplegar MLFlow en Google Cloud Platform (GCP). Todos los nombres de marcador de posición (por ejemplo, `[NOMBRE]`) deben ser reemplazados con valores específicos de tu proyecto.

---

## 1. Configurar la Base de Datos

MLFlow utiliza una base de datos para almacenar los datos de experimentos. Comenzaremos creando una instancia de PostgreSQL en GCP.

### 1.1 Crear una instancia de Cloud SQL
```bash
gcloud sql instances create [NOMBRE_INSTANCIA] \
    --database-version=POSTGRES_15 \
    --region=us-central1 \
    --tier=db-f1-micro \
    --storage-type=HDD \
    --storage-size=10GB \
    --authorized-networks=0.0.0.0/0
```

### 1.2 Crear un usuario para la base de datos
```bash
gcloud sql users create [NOMBRE_USUARIO] \
    --instance=[NOMBRE_INSTANCIA] \
    --password=[CONTRASEÑA]
```

### 1.3 Crear la base de datos
```bash
gcloud sql databases create [NOMBRE_BASE_DATOS] --instance=[NOMBRE_INSTANCIA]
```

---

## 2. Configurar Cloud Storage

MLFlow requiere un repositorio remoto para los artefactos, el cual configuraremos usando Cloud Storage de GCP.

### 2.1 Crear un bucket de almacenamiento
```bash
gcloud storage buckets create gs://[NOMBRE_BUCKET]
```

### 2.2 Crear una carpeta dentro del bucket
En la consola de Cloud Storage, navega al bucket y crea una carpeta llamada `mlruns`.

---

## 3. Configurar Artifact Registry

Usaremos Artifact Registry para almacenar contenedores para Cloud Run.

### 3.1 Crear un repositorio Docker
```bash
gcloud artifacts repositories create [NOMBRE_REPOSITORIO] \
    --location=us-central1 \
    --repository-format=docker
```

---

## 4. Configurar una Cuenta de Servicio

Para permitir el acceso a Cloud SQL, Cloud Storage y otros servicios de GCP, crearemos una cuenta de servicio y le asignaremos los roles necesarios.

### 4.1 Crear la Cuenta de Servicio
```bash
gcloud iam service-accounts create [NOMBRE_CUENTA_SERVICIO]
```

### 4.2 Asignar Roles a la Cuenta de Servicio
Asigna los siguientes roles a la cuenta de servicio:

#### Editor de Cloud SQL
```bash
gcloud projects add-iam-policy-binding [ID_PROYECTO] \
    --member='serviceAccount:[NOMBRE_CUENTA_SERVICIO]@[ID_PROYECTO].iam.gserviceaccount.com' \
    --role='roles/cloudsql.editor'
```

#### Administrador de Cloud Storage
```bash
gcloud projects add-iam-policy-binding [ID_PROYECTO] \
    --member='serviceAccount:[NOMBRE_CUENTA_SERVICIO]@[ID_PROYECTO].iam.gserviceaccount.com' \
    --role='roles/storage.objectAdmin'
```

#### Acceso a Secret Manager
```bash
gcloud projects add-iam-policy-binding [ID_PROYECTO] \
    --member='serviceAccount:[NOMBRE_CUENTA_SERVICIO]@[ID_PROYECTO].iam.gserviceaccount.com' \
    --role='roles/secretmanager.secretAccessor'
```

#### Administrador de Artifact Registry
```bash
gcloud projects add-iam-policy-binding [ID_PROYECTO] \
    --member='serviceAccount:[NOMBRE_CUENTA_SERVICIO]@[ID_PROYECTO].iam.gserviceaccount.com' \
    --role='roles/artifactregistry.admin'
```

#### Administrador de Cloud Functions
```bash
gcloud projects add-iam-policy-binding [ID_PROYECTO] \
    --member='serviceAccount:[NOMBRE_CUENTA_SERVICIO]@[ID_PROYECTO].iam.gserviceaccount.com' \
    --role='roles/cloudfunctions.admin'
```

#### Agente de Cloud Deploy
```bash
gcloud projects add-iam-policy-binding [ID_PROYECTO] \
    --member='serviceAccount:[NOMBRE_CUENTA_SERVICIO]@[ID_PROYECTO].iam.gserviceaccount.com' \
    --role='roles/clouddeploy.serviceAgent'
```

### 4.3 Verificar las Cuentas de Servicio
```bash
gcloud config get-value project
gcloud iam service-accounts list
```

---

## 5. Habilitar la API de Cloud Run

```bash
gcloud services enable run.googleapis.com
```

---

## 6. Configurar Secret Manager

Para gestionar credenciales y configuraciones de forma segura, usaremos Secret Manager.

### 6.1 Crear una clave privada para la Cuenta de Servicio
```bash
gcloud iam service-accounts keys create sa-private-key.json \
    --iam-account=[NOMBRE_CUENTA_SERVICIO]@[ID_PROYECTO].iam.gserviceaccount.com
```

### 6.2 Crear un secreto para las claves de acceso
```bash
gcloud secrets create access_keys --data-file=sa-private-key.json
```

### 6.3 Agregar la URL de la base de datos PostgreSQL como un secreto
```bash
echo -n "postgresql://[NOMBRE_USUARIO]:[CONTRASEÑA]@[IP_INSTANCIA]/[NOMBRE_BASE_DATOS]" | \
    gcloud secrets versions add database_url --data-file=-
```

### 6.4 Agregar la URL del bucket como un secreto
```bash
echo -n "gs://[NOMBRE_BUCKET]/mlruns" | \
    gcloud secrets versions add bucket_url --data-file=-
```

---

## 7. Repositorio de GitHub

Creé este repositorio que contiene la estructura básica que necesitas para ejecutar la primera versión de este proyecto. Con esto desplegamos el servicio usando GitHub Actions.

### 7.1 GitHub Secret
Abre la configuración del repositorio y ve a **Secrets -> Actions** en la pestaña de “Seguridad”.

Crea un nuevo secreto llamado `GOOGLE_APPLICATION_CREDENTIALS` y copia el contenido del archivo `sa-private-key.json`.

Después de esto, puedes eliminar este archivo de tu máquina local de forma segura. Este secreto será utilizado por los flujos de trabajo de GitHub Actions para autenticarse con Google Cloud.

### 7.2 Dockerfile
El `Dockerfile` es muy simple. Definimos el secreto `access_keys` como una variable de entorno, que será pasada como un volumen montado en `secrets/credentials/` cuando despleguemos el contenedor en Cloud Run. Luego, instalamos los requisitos, exponemos el puerto 8080 y definimos el punto de entrada.

### 7.3 Archivo YML
Este archivo es el flujo de trabajo usado por GitHub Actions para construir nuestro contenedor y desplegarlo en Cloud Run.

Lo único que necesitas hacer aquí es reemplazar las variables dentro de `env` con las que has creado hasta ahora en este proyecto:

- `REGION`: La región que estás usando (ejemplo: `us-central1`).
- `PROJECT_ID`: Tu ID de proyecto en GCP.
- `REPOSITORY`: Tu repositorio creado en Artifact Registry.
- `SERVICE_ACCOUNT`: El correo completo de la cuenta de servicio.
- `SERVICE_NAME`: Un nombre adecuado para tu servicio en Cloud Run.
- `DEPLOY`: Cambia a `true` si deseas desplegar.

Este archivo utiliza el secreto `GOOGLE_APPLICATION_CREDENTIALS` que acabamos de definir para autenticarse con Google Cloud, construir el contenedor basado en el código del `Dockerfile`, subir el contenedor a Artifact Registry y desplegarlo en Cloud Run.

### 7.4 Push y Merge
Cuando tengas todo listo, haz un `push` y `merge` a la rama `main`. Deberías ver el progreso en la sección de **Actions** del repositorio.

---

## Notas

- Reemplaza todos los marcadores de posición (por ejemplo, `[ID_PROYECTO]`, `[NOMBRE_CUENTA_SERVICIO]`, `[NOMBRE_INSTANCIA]`) con los nombres específicos de tu proyecto.
- Asegúrate de tener permisos suficientes para realizar las acciones descritas.
- Por seguridad, evita usar `authorized-networks=0.0.0.0/0` en entornos de producción.









