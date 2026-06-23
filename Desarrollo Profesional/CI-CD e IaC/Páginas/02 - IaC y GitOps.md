# 🚀 CI/CD 02 — IaC y GitOps

[[Desarrollo Profesional/CI-CD e IaC/CI-CD e IaC|⬅️ Volver a CI/CD e IaC]] | [[Desarrollo Profesional/CI-CD e IaC/Páginas/01 - Pipelines CI-CD|← 01]]

> [!abstract] Introducción
> Durante décadas la infraestructura se montaba a mano: alguien entraba a una consola, creaba una máquina, configuraba una red, y ese conocimiento vivía en su cabeza. El problema: no es reproducible, no es auditable y nadie sabe exactamente qué hay desplegado (configuration drift). La Infraestructura como Código resuelve esto describiendo la infraestructura en ficheros versionados que se aplican de forma automática e idempotente. GitOps extiende la idea: git es la fuente de verdad y un agente mantiene el sistema sincronizado con lo que dice el repo.

## ¿De qué vamos a hablar?

Cómo gestionar infraestructura y despliegues como código versionado, declarativo y reproducible.

### Conceptos que vamos a cubrir
- IaC: declarativo vs imperativo, idempotencia
- Terraform: estado, plan/apply, drift
- Helm: plantillas para Kubernetes
- GitOps: el repo como fuente de verdad, ArgoCD
- Push vs Pull deployment

---

## El Concepto

### Infraestructura como Código: Declarativo vs Imperativo

Dos formas de automatizar infraestructura:

- **Imperativo**: describes *los pasos*. "Crea una VM, luego abre el puerto 443, luego instala nginx." Como un script bash. El problema: si lo ejecutas dos veces, falla o duplica; no sabe el estado actual.
- **Declarativo**: describes *el estado deseado*. "Quiero una VM con el puerto 443 abierto y nginx." La herramienta calcula la diferencia entre lo que hay y lo que quieres, y solo aplica lo necesario. Es **idempotente**: aplicarlo N veces deja el mismo resultado. Terraform, Kubernetes y la IaC moderna son declarativos.

La idempotencia es la propiedad clave (la misma idea que viste en [[Desarrollo Profesional/Sistemas Distribuidos/Páginas/03 - Transacciones Distribuidas|Sistemas Distribuidos]]): puedes aplicar la configuración una y otra vez sin miedo, porque solo cambia lo que difiere del estado deseado.

### Terraform

Terraform es la herramienta de IaC de facto para aprovisionar recursos en la nube (VMs, redes, buckets, bases de datos) de forma declarativa y agnóstica al proveedor.

```hcl
# main.tf — describe el estado deseado, no los pasos
resource "google_storage_bucket" "datos" {
  name          = "mi-empresa-datos-prod"
  location      = "EU"
  storage_class = "STANDARD"

  uniform_bucket_level_access = true   # seguridad por defecto

  lifecycle_rule {
    condition { age = 90 }
    action    { type = "Delete" }      # borra objetos de >90 días
  }
}

resource "google_cloud_run_v2_service" "api" {
  name     = "api"
  location = "europe-west1"
  template {
    containers {
      image = "europe-docker.pkg.dev/${var.proyecto}/app:${var.version}"
    }
  }
}
```

El ciclo de trabajo:
- `terraform plan`: muestra **qué cambiaría** sin aplicarlo (el diff entre el estado deseado y el real). Esto es lo que revisas en un PR.
- `terraform apply`: aplica el plan, creando/modificando/destruyendo recursos para alcanzar el estado deseado.

El concepto crítico es el **state file**: Terraform guarda un fichero (`terraform.tfstate`) con su visión del mundo —qué recursos gestiona y sus IDs reales. Este estado:
- Debe guardarse en un **backend remoto compartido** (GCS, S3) con **locking**, nunca en local ni en git, para que el equipo no lo corrompa con cambios concurrentes.
- Puede contener **secretos en claro** (contraseñas generadas) → trátalo como sensible, cífralo.
- Puede desincronizarse de la realidad si alguien toca la infra a mano (**drift**). `terraform plan` lo detecta; la disciplina es **no tocar a mano lo gestionado por Terraform**.

### Helm — Plantillas para Kubernetes

Los manifiestos YAML de Kubernetes ([[Desarrollo Profesional/Kubernetes/Páginas/02 - Workloads|Workloads]]) se repiten entre entornos con pequeñas diferencias (la imagen, las réplicas, los recursos). **Helm** es el gestor de paquetes de Kubernetes: empaqueta manifiestos en **charts** parametrizables con plantillas y un fichero de `values`.

```yaml
# templates/deployment.yaml — plantilla, no valores fijos
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
        - image: "{{ .Values.image.repo }}:{{ .Values.image.tag }}"
          resources:
            requests:
              cpu: {{ .Values.resources.cpu }}
---
# values-prod.yaml — valores para producción
replicaCount: 5
image:
  repo: europe-docker.pkg.dev/proyecto/app
  tag: "1.4.2"
resources:
  cpu: "500m"
```

Así, `helm install -f values-prod.yaml` y `-f values-staging.yaml` despliegan la misma plantilla con configuración distinta. Helm también gestiona **releases** (versiones desplegadas) y permite `helm rollback` a una versión anterior. Alternativa popular: **Kustomize** (parches en lugar de plantillas, integrado en `kubectl`).

### GitOps

GitOps es un modelo operativo con un principio simple: **el estado deseado de todo el sistema vive declarado en un repositorio git, y un agente automático reconcilia continuamente el sistema real para que coincida con el repo.**

Las cuatro reglas:
1. **Todo es declarativo** (manifiestos, charts, config).
2. **El estado deseado está versionado en git** (fuente única de verdad).
3. **Los cambios aprobados se aplican automáticamente** (un merge a main dispara el despliegue).
4. **Un agente reconcilia y corrige el drift** (si alguien cambia algo a mano en el clúster, el agente lo revierte al estado del repo).

La gran diferencia es **push vs pull**:

```
Despliegue PUSH (CI/CD clásico):
  Pipeline de CI ──(tiene credenciales del clúster)──→ kubectl apply → Clúster
  Problema: el CI necesita credenciales potentes de producción.

Despliegue PULL (GitOps):
  Repo git (estado deseado)
        ▲                 │
        │ observa cambios │
   Agente dentro del clúster (ArgoCD/Flux) ──reconcilia──→ Clúster
  El clúster se actualiza solo. CI nunca toca producción directamente.
```

**ArgoCD** (y Flux) es el agente que corre *dentro* del clúster, observa el repo y aplica los cambios. Ventajas del modelo pull:
- **Seguridad**: las credenciales del clúster no salen de él; el CI solo hace push a git.
- **Auditabilidad**: cada cambio a producción es un commit con autor, revisión y fecha.
- **Rollback trivial**: `git revert` y el agente vuelve al estado anterior.
- **Anti-drift**: el agente detecta y corrige cambios manuales, garantizando que producción siempre refleja el repo.

El flujo completo queda: **el pipeline de CI** ([página 01](01%20-%20Pipelines%20CI-CD)) construye y testea la imagen y actualiza el tag en el repo de config; **ArgoCD** detecta el cambio y despliega. CI construye, GitOps despliega.

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - **IaC declarativa** (describe el estado deseado, no los pasos) es **idempotente**: aplícala N veces, solo cambia lo que difiere. Imperativo describe pasos y no escala.
> - **Terraform** aprovisiona infra en la nube con `plan` (ver el diff, revisar en PR) y `apply`. Su **state file** debe ir en un backend remoto con locking, puede contener secretos, y detecta **drift** — no toques a mano lo que gestiona.
> - **Helm** empaqueta manifiestos de Kubernetes en charts parametrizables (plantillas + values por entorno) y gestiona releases y rollbacks. Kustomize es la alternativa basada en parches.
> - **GitOps**: git es la fuente de verdad y un agente reconcilia el sistema. Cambia el modelo **push** (CI con credenciales de prod) por **pull** (ArgoCD/Flux dentro del clúster), ganando seguridad, auditabilidad, rollback con `git revert` y corrección de drift.
> - División de responsabilidades: **CI construye y testea, GitOps despliega**.

### Para llevar a la práctica
- [ ] Identifica infraestructura tuya creada a mano. ¿Podrías describirla en Terraform e importarla?
- [ ] Comprueba dónde vive tu state de Terraform. Si está en local o en git, muévelo a un backend remoto con locking.
- [ ] Si despliegas con `kubectl apply` desde CI, evalúa GitOps (ArgoCD) para sacar las credenciales de prod del pipeline.
- [ ] Revisa si tienes configuration drift: ¿coincide lo que hay en el clúster con lo que está en el repo?

### Recursos
- 🌐 developer.hashicorp.com/terraform/intro — documentación oficial de Terraform
- 🌐 argo-cd.readthedocs.io — ArgoCD
- 🌐 opengitops.dev — los principios de GitOps
- 📖 *Terraform: Up & Running* — Yevgeniy Brikman
- 📖 *Infrastructure as Code* — Kief Morris

---
`#iac` `#terraform` `#gitops` `#argocd` `#helm`
