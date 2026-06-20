# ☁️ GCP (Google Cloud Platform)

[[Desarrollo Profesional/Inicio Profesional|⬅️ Volver a Desarrollo Profesional]]

> [!abstract] Google Cloud Platform (GCP)
> Plataforma de servicios en la nube de Google que ofrece computación, almacenamiento, análisis de datos y aprendizaje automático.

---

## 🛠️ Servicios Principales

### 1. Cómputo (Compute)
- **Compute Engine:** Máquinas virtuales administradas (IaaS).
- **Google Kubernetes Engine (GKE):** Orquestación y ejecución de contenedores administrados por Google.
- **Cloud Run:** Plataforma serverless para ejecutar contenedores HTTP sin preocuparse por la infraestructura.
- **Cloud Functions:** Funciones serverless basadas en eventos.

### 2. Almacenamiento (Storage)
- **Cloud Storage (GCS):** Almacenamiento de objetos altamente disponible e ilimitado (similar a AWS S3).
- **Cloud SQL:** Motores relacionales administrados (PostgreSQL, MySQL, SQL Server).
- **Cloud Spanner:** Base de datos relacional de escalabilidad horizontal global.
- **Firestore:** Base de datos NoSQL basada en documentos.

### 3. Redes y Seguridad
- **VPC (Virtual Private Cloud):** Red privada virtual para tus recursos en la nube.
- **IAM (Identity and Access Management):** Gestión de accesos, políticas y cuentas de servicio.

---

## ⚡ Comandos Útiles (`gcloud CLI`)

| Comando | Descripción |
| --- | --- |
| `gcloud auth login` | Autenticarse en GCP |
| `gcloud config set project [PROJECT_ID]` | Seleccionar un proyecto de trabajo |
| `gcloud compute instances list` | Listar máquinas virtuales |
| `gcloud container clusters get-credentials [CLUSTER]` | Configurar acceso local a un clúster de GKE |

---
`#gcp` `#cloud` `#google` `#apuntes`
