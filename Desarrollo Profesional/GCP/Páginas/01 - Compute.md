# ☁️ GCP 01 — Compute

[[Desarrollo Profesional/GCP/GCP|⬅️ Volver a GCP]] | [[Desarrollo Profesional/GCP/Páginas/02 - Storage y Bases de Datos|02 →]]

> [!abstract] Introducción
> Google Cloud ofrece un espectro completo de opciones de cómputo: desde máquinas virtuales de bajo nivel hasta plataformas serverless donde no gestionas absolutamente nada. La decisión de cuál usar no es trivial — afecta al coste, al control, a la operabilidad y a la velocidad de desarrollo.

## ¿De qué vamos a hablar?

Los cuatro servicios de cómputo principales de GCP: Compute Engine, GKE, Cloud Run y Cloud Functions — cuándo usar cada uno y cómo se configuran.

### Conceptos que vamos a cubrir
- Compute Engine: VMs y cuándo todavía tiene sentido
- Google Kubernetes Engine (GKE): Kubernetes gestionado
- Cloud Run: contenedores serverless
- Cloud Functions: funciones serverless basadas en eventos
- Árbol de decisión: qué usar en cada escenario

---

## El Concepto

### Árbol de Decisión

```
¿Qué tipo de workload?
│
├── Quiero control total del SO, configs especiales de red, GPUs
│   └── Compute Engine (VMs)
│
├── Tengo contenedores y quiero orquestación, estado, microservicios complejos
│   └── GKE (Kubernetes gestionado)
│
├── Tengo un servicio HTTP/gRPC en contenedor, quiero escalar a cero
│   └── Cloud Run (serverless containers)
│
└── Tengo lógica pequeña que responde a eventos (Pub/Sub, HTTP, Storage)
    └── Cloud Functions (serverless functions)
```

### Compute Engine — Máquinas Virtuales (IaaS)

Cuando necesitas control total: sistemas operativo, networking, librerías de sistema, drivers de GPU.

```bash
# Crear una VM
gcloud compute instances create mi-vm \
  --zone=europe-west1-b \
  --machine-type=e2-medium \
  --image-family=debian-12 \
  --image-project=debian-cloud \
  --boot-disk-size=50GB \
  --tags=http-server,https-server

# Conectar por SSH
gcloud compute ssh mi-vm --zone=europe-west1-b

# Tipos de máquina:
# e2-*        : propósito general, más baratas
# n2-*        : mejor precio/rendimiento para cargas de trabajo mixtas
# c2-*        : CPU-optimized para cómputo intensivo
# m3-*        : Memory-optimized para bases de datos en memoria
# a2-*        : GPU machines (A100) para IA/ML

# Preemptible/Spot VMs — hasta 90% de descuento, pueden ser interrumpidas
gcloud compute instances create mi-vm-spot \
  --provisioning-model=SPOT \
  --machine-type=n2-standard-4
```

**Cuándo usar Compute Engine:**
- Cargas de trabajo que requieren configuración específica del SO
- Aplicaciones legacy que no están en contenedores
- Workloads de ML con GPUs
- Bases de datos autogestionadas

### GKE — Kubernetes Gestionado

GKE gestiona el control plane de Kubernetes (no pagas por él, Google lo administra). Tú gestionas los worker nodes.

```bash
# Crear un cluster
gcloud container clusters create mi-cluster \
  --zone=europe-west1-b \
  --num-nodes=3 \
  --machine-type=n2-standard-4 \
  --enable-autoscaling \
  --min-nodes=1 \
  --max-nodes=10

# Autopilot — GKE donde Google gestiona también los nodes (recomendado)
gcloud container clusters create-auto mi-cluster-autopilot \
  --region=europe-west1

# Obtener credenciales para kubectl
gcloud container clusters get-credentials mi-cluster --zone=europe-west1-b

# Modos de GKE:
# Standard: tú gestionas los node pools, más control y más coste operativo
# Autopilot: Google gestiona nodes, pagas solo por los pods — RECOMENDADO para nuevos proyectos
```

### Cloud Run — Contenedores Serverless

El sweet spot entre "quiero contenedores" y "no quiero gestionar Kubernetes". Escala a cero, pagas solo por peticiones.

```bash
# Desplegar una imagen de contenedor
gcloud run deploy mi-servicio \
  --image=gcr.io/mi-proyecto/mi-app:latest \
  --region=europe-west1 \
  --platform=managed \
  --allow-unauthenticated \
  --memory=512Mi \
  --cpu=1 \
  --concurrency=100 \
  --max-instances=10 \
  --set-env-vars="DATABASE_URL=postgresql://..."

# Cloud Run Jobs — para tareas batch que no son HTTP
gcloud run jobs create mi-job \
  --image=gcr.io/mi-proyecto/mi-job:latest \
  --region=europe-west1 \
  --tasks=10 \
  --parallelism=5

# Actualizar tráfico — despliegues graduales
gcloud run services update-traffic mi-servicio \
  --to-revisions=LATEST=10  # 10% al nuevo, 90% al anterior
```

**Cuándo usar Cloud Run:**
- Servicios HTTP/gRPC sin estado
- APIs que tienen tráfico irregular (muchos picos y valles)
- Servicios que quieres desplegar rápido sin gestionar infraestructura

### Cloud Functions — Funciones Serverless

Para lógica pequeña y event-driven. Escala automáticamente, pagas por invocación.

```python
# Función HTTP — responde a peticiones HTTP
import functions_framework

@functions_framework.http
def mi_funcion(request):
    nombre = request.args.get("nombre", "mundo")
    return f"Hola, {nombre}!"

# Función CloudEvent — responde a eventos de GCP
@functions_framework.cloud_event
def procesar_archivo(cloud_event):
    data = cloud_event.data
    nombre_archivo = data["name"]
    bucket = data["bucket"]
    print(f"Nuevo archivo: gs://{bucket}/{nombre_archivo}")
```

```bash
# Desplegar
gcloud functions deploy mi-funcion \
  --runtime=python312 \
  --trigger-http \
  --allow-unauthenticated \
  --region=europe-west1 \
  --memory=256MB

# Disparada por eventos de Cloud Storage
gcloud functions deploy procesar-archivo \
  --runtime=python312 \
  --trigger-event=google.storage.object.finalize \
  --trigger-resource=mi-bucket \
  --region=europe-west1
```

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - **Compute Engine**: IaaS, control total, para casos que no encajan en contenedores o necesitan GPUs.
> - **GKE Autopilot**: para microservicios con estado, cargas de trabajo complejas que necesitan Kubernetes. Autopilot en lugar de Standard para nuevos proyectos.
> - **Cloud Run**: la elección por defecto para servicios HTTP. Serverless, escala a cero, sin gestión de nodes.
> - **Cloud Functions**: para lógica pequeña event-driven. Ideal para integraciones entre servicios de GCP.

### Recursos
- 🌐 cloud.google.com/run/docs — documentación de Cloud Run
- 🌐 cloud.google.com/kubernetes-engine/docs/concepts/autopilot-overview
- 🌐 cloud.google.com/functions/docs

---
`#gcp` `#compute` `#cloud-run` `#gke` `#kubernetes`
