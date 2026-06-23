# ☁️ GCP 03 — Networking e IAM

[[Desarrollo Profesional/GCP/GCP|⬅️ Volver a GCP]] | [[Desarrollo Profesional/GCP/Páginas/02 - Storage y Bases de Datos|← 02]] | [[Desarrollo Profesional/GCP/Páginas/04 - Observabilidad|04 →]]

> [!abstract] Introducción
> La red y el control de acceso son las dos capas de seguridad más importantes en GCP. Una VPC mal diseñada expone recursos que deberían ser privados. Un IAM mal configurado concede más permisos de los necesarios. Entender VPC, Cloud Load Balancing, y las Service Accounts es fundamental para desplegar de forma segura.

## ¿De qué vamos a hablar?

VPC, subnets, reglas de firewall, Cloud Load Balancing, IAM roles y Service Accounts.

---

## El Concepto

### VPC — Virtual Private Cloud

Una VPC es tu red privada virtual en GCP. Cada proyecto tiene una VPC `default`, pero en producción deberías crear la tuya:

```bash
# Crear VPC
gcloud compute networks create mi-vpc \
  --subnet-mode=custom \
  --bgp-routing-mode=regional

# Crear subnet
gcloud compute networks subnets create mi-subnet-eu \
  --network=mi-vpc \
  --region=europe-west1 \
  --range=10.0.0.0/24 \
  --secondary-range=pods=10.1.0.0/16,services=10.2.0.0/20  # para GKE

# Reglas de firewall
# Permite tráfico HTTP/HTTPS desde internet
gcloud compute firewall-rules create allow-http \
  --network=mi-vpc \
  --allow=tcp:80,tcp:443 \
  --source-ranges=0.0.0.0/0 \
  --target-tags=http-server

# Permite SSH solo desde Cloud IAP (Identity-Aware Proxy) — no exponer SSH al mundo
gcloud compute firewall-rules create allow-ssh-iap \
  --network=mi-vpc \
  --allow=tcp:22 \
  --source-ranges=35.235.240.0/20  # IP range de Cloud IAP

# Denegar todo tráfico entrante excepto lo explícitamente permitido (por defecto en VPCs custom)
gcloud compute firewall-rules create deny-all-ingress \
  --network=mi-vpc \
  --action=DENY \
  --rules=all \
  --direction=INGRESS \
  --priority=65534
```

### Cloud Load Balancing

GCP ofrece distintos tipos de load balancer según el caso de uso:

```
Tipo de tráfico y alcance:
├── Global HTTP(S) Load Balancer — HTTP/HTTPS, global (CDN integrado)
├── Regional HTTP(S) Load Balancer — HTTP/HTTPS, una región
├── TCP/SSL Proxy Load Balancer — TCP, global
└── Network Load Balancer — cualquier protocolo, regional, alta performance

Para la mayoría de aplicaciones web: Global HTTP(S) Load Balancer
Para GKE: usar Ingress de Kubernetes (crea automáticamente un HTTP LB)
Para Cloud Run: tiene su propio LB gestionado automáticamente
```

```bash
# Para GKE — Ingress crea automáticamente un HTTP LB
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mi-ingress
  annotations:
    kubernetes.io/ingress.class: "gce"
    kubernetes.io/ingress.global-static-ip-name: "mi-ip-estatica"
spec:
  rules:
    - host: api.ejemplo.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: mi-api
                port:
                  number: 80
EOF
```

### IAM — Identity and Access Management

IAM controla quién puede hacer qué sobre qué recursos. El principio de mínimo privilegio: dar exactamente los permisos necesarios, ni más ni menos.

```
Entidades (WHO)      → Roles (WHAT)        → Recursos (ON WHAT)
├── User accounts    → Basic roles          ├── Project
├── Groups           │  (Owner, Editor,     ├── Folder
├── Service Accounts │   Viewer) — EVITAR   ├── Organization
└── Workload         → Predefined roles      └── Resource específico
   Identity          │  (roles/storage.objectViewer)
                     └── Custom roles
```

```bash
# Listar miembros y roles de un proyecto
gcloud projects get-iam-policy mi-proyecto

# Dar un rol a un usuario
gcloud projects add-iam-policy-binding mi-proyecto \
  --member=user:ana@empresa.com \
  --role=roles/storage.objectViewer  # acceso de solo lectura a GCS

# Revocar un rol
gcloud projects remove-iam-policy-binding mi-proyecto \
  --member=user:ana@empresa.com \
  --role=roles/storage.objectViewer

# Roles más usados:
# roles/viewer                    — lectura de todos los recursos
# roles/editor                    — lectura y escritura (EVITAR en producción)
# roles/storage.objectAdmin       — control total de objetos en GCS
# roles/storage.objectViewer      — solo lectura de objetos
# roles/cloudsql.client           — conectar a Cloud SQL
# roles/logging.logWriter         — escribir logs
# roles/monitoring.metricWriter   — escribir métricas
```

### Service Accounts — Identidad para Aplicaciones

Las Service Accounts son identidades para aplicaciones (no para humanos). Tu servicio en Cloud Run, GKE o GCE debe tener su propia Service Account con exactamente los permisos que necesita:

```bash
# Crear Service Account para tu aplicación
gcloud iam service-accounts create mi-app-sa \
  --display-name="Service Account para mi-app" \
  --description="Identidad de mi-app en producción"

# Dar permisos mínimos necesarios
# Solo puede leer de GCS (no escribir, no borrar)
gcloud projects add-iam-policy-binding mi-proyecto \
  --member=serviceAccount:mi-app-sa@mi-proyecto.iam.gserviceaccount.com \
  --role=roles/storage.objectViewer

# Solo puede escribir en Cloud SQL
gcloud projects add-iam-policy-binding mi-proyecto \
  --member=serviceAccount:mi-app-sa@mi-proyecto.iam.gserviceaccount.com \
  --role=roles/cloudsql.client

# Asignar la SA a un Deployment de GKE con Workload Identity
# (sin claves JSON, la autenticación es automática)
gcloud iam service-accounts add-iam-policy-binding \
  mi-app-sa@mi-proyecto.iam.gserviceaccount.com \
  --role=roles/iam.workloadIdentityUser \
  --member="serviceAccount:mi-proyecto.svc.id.goog[default/mi-app-ksa]"
```

```python
# En el código: las librerías de GCP detectan automáticamente las credenciales
# En GKE con Workload Identity, Cloud Run o GCE, no necesitas configurar nada
from google.cloud import storage

# Esto usa automáticamente la Service Account del entorno
client = storage.Client()
bucket = client.bucket("mi-bucket")
blob = bucket.blob("archivo.txt")
blob.upload_from_string("contenido")

# En desarrollo local: usa Application Default Credentials
# gcloud auth application-default login
```

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - Crea una **VPC custom** en producción — nunca uses la VPC `default`. Define subnets con rangos apropiados y reglas de firewall explícitas.
> - Usa **Cloud IAP** para acceso SSH a VMs en lugar de exponer el puerto 22 a internet.
> - En IAM, sigue el **principio de mínimo privilegio**: da el rol predefinido más restrictivo que cubra el caso de uso. Evita `roles/editor` y `roles/owner` en producción.
> - Cada servicio debe tener su propia **Service Account** con permisos mínimos. Usa Workload Identity en GKE para autenticación sin claves JSON.

### Recursos
- 🌐 cloud.google.com/iam/docs/understanding-roles
- 🌐 cloud.google.com/vpc/docs/vpc
- 🌐 cloud.google.com/kubernetes-engine/docs/how-to/workload-identity

---
`#gcp` `#networking` `#iam` `#vpc` `#service-accounts`
