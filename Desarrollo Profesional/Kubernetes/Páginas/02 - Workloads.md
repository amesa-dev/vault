# ☸️ K8s 02 — Workloads

[[Desarrollo Profesional/Kubernetes/Kubernetes|⬅️ Volver a Kubernetes]] | [[Desarrollo Profesional/Kubernetes/Páginas/01 - Arquitectura Core|← 01]] | [[Desarrollo Profesional/Kubernetes/Páginas/03 - Networking|03 →]]

> [!abstract] Introducción
> Un workload es una aplicación corriendo en Kubernetes. Los objetos de workload de Kubernetes gestionan los Pods — la unidad mínima de despliegue. Para la mayoría de aplicaciones sin estado usarás Deployment; para bases de datos y aplicaciones con estado, StatefulSet; para tareas por lotes, Job/CronJob. Entender cuándo usar cada uno es fundamental.

## ¿De qué vamos a hablar?

Los objetos de workload de Kubernetes: Pod, Deployment, StatefulSet, DaemonSet y Job/CronJob — sus diferencias, cuándo usarlos y cómo escribir sus manifiestos correctamente.

### Conceptos que vamos a cubrir
- Pod: la unidad de despliegue
- Deployment: aplicaciones sin estado
- StatefulSet: aplicaciones con estado
- DaemonSet: un pod por nodo
- Job y CronJob: tareas por lotes

---

## El Concepto

### Pod — La Unidad Mínima

Un Pod es un grupo de uno o más contenedores que comparten red (mismo localhost) y volúmenes. Son efímeros — si un pod muere, Kubernetes puede crear uno nuevo, pero no lo resucita automáticamente (eso lo hacen los controladores).

```yaml
# pod.yaml — casi nunca se crea un Pod directamente
apiVersion: v1
kind: Pod
metadata:
  name: mi-app
  labels:
    app: mi-app
spec:
  containers:
    - name: app
      image: gcr.io/mi-proyecto/mi-app:v1.2.3
      ports:
        - containerPort: 8080
      resources:
        requests:      # lo que el scheduler reserva
          memory: "128Mi"
          cpu: "100m"  # 100 millicores = 0.1 CPU
        limits:        # el máximo — si lo supera, OOMKilled (memoria) o throttling (CPU)
          memory: "256Mi"
          cpu: "500m"
      env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: mi-secret
              key: database-url
      livenessProbe:
        httpGet:
          path: /health
          port: 8080
        initialDelaySeconds: 30
        periodSeconds: 10
      readinessProbe:
        httpGet:
          path: /ready
          port: 8080
        initialDelaySeconds: 5
        periodSeconds: 5
```

### Deployment — Aplicaciones Sin Estado

El objeto que usarás el 90% del tiempo. Gestiona ReplicaSets, que gestionan Pods. Permite rollouts y rollbacks:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mi-api
  namespace: produccion
spec:
  replicas: 3
  selector:
    matchLabels:
      app: mi-api
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1   # máximo 1 pod caído durante el update
      maxSurge: 1         # máximo 1 pod extra durante el update
  template:
    metadata:
      labels:
        app: mi-api
        version: "1.2.3"
    spec:
      containers:
        - name: api
          image: gcr.io/mi-proyecto/mi-api:1.2.3
          ports:
            - containerPort: 8080
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "1000m"
          envFrom:
            - secretRef:
                name: mi-api-secrets
            - configMapRef:
                name: mi-api-config
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 30
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            initialDelaySeconds: 10
      # Distribuir pods entre nodos — evitar SPOF
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: mi-api
```

```bash
# Operaciones de Deployment
kubectl apply -f deployment.yaml

# Ver estado del rollout
kubectl rollout status deployment/mi-api -n produccion

# Historial de versiones
kubectl rollout history deployment/mi-api -n produccion

# Rollback a la versión anterior
kubectl rollout undo deployment/mi-api -n produccion

# Rollback a una versión específica
kubectl rollout undo deployment/mi-api --to-revision=2 -n produccion

# Escalar manualmente
kubectl scale deployment/mi-api --replicas=5 -n produccion
```

### StatefulSet — Aplicaciones con Estado

Para aplicaciones donde cada pod necesita identidad estable y almacenamiento persistente: bases de datos, Kafka, Elasticsearch. A diferencia de Deployment, los pods se crean y destruyen en orden y mantienen nombres estables (`pod-0`, `pod-1`...):

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres-headless
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:15
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: password
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
          resources:
            requests:
              memory: "512Mi"
              cpu: "500m"
            limits:
              memory: "2Gi"
              cpu: "2000m"
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: standard-rwo
        resources:
          requests:
            storage: 50Gi
```

### DaemonSet — Un Pod por Nodo

Para agentes que deben correr en cada nodo: colectores de logs (Fluentd), agentes de monitorización (Datadog, Prometheus node-exporter), agentes de seguridad.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: kube-system
spec:
  selector:
    matchLabels:
      name: fluentd
  template:
    metadata:
      labels:
        name: fluentd
    spec:
      tolerations:
        - key: node-role.kubernetes.io/control-plane
          effect: NoSchedule
      containers:
        - name: fluentd
          image: fluent/fluentd-kubernetes-daemonset:v1
          resources:
            limits:
              memory: "200Mi"
            requests:
              cpu: "100m"
              memory: "200Mi"
          volumeMounts:
            - name: varlog
              mountPath: /var/log
      volumes:
        - name: varlog
          hostPath:
            path: /var/log
```

### Job y CronJob — Tareas por Lotes

```yaml
# Job — ejecuta una tarea hasta que termina con éxito
apiVersion: batch/v1
kind: Job
metadata:
  name: migrar-bd
spec:
  completions: 1
  parallelism: 1
  backoffLimit: 3  # reintentos máximos
  template:
    spec:
      restartPolicy: OnFailure
      containers:
        - name: migrar
          image: gcr.io/mi-proyecto/migrations:latest
          command: ["python", "-m", "alembic", "upgrade", "head"]
          envFrom:
            - secretRef:
                name: mi-app-secrets

---
# CronJob — ejecuta un Job según un schedule cron
apiVersion: batch/v1
kind: CronJob
metadata:
  name: limpiar-sesiones
spec:
  schedule: "0 2 * * *"  # cada día a las 2:00 AM
  concurrencyPolicy: Forbid  # no ejecutar si el anterior aún corre
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: limpiar
              image: gcr.io/mi-proyecto/cleaner:latest
              command: ["python", "clean_sessions.py"]
```

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - **Pod**: unidad mínima, efímero. Casi nunca se crea directamente — usa un controller.
> - **Deployment**: para aplicaciones sin estado. Gestiona rolling updates y rollbacks. La opción por defecto para servicios HTTP.
> - **StatefulSet**: para aplicaciones con estado (BD, Kafka). Los pods tienen nombres estables y volúmenes persistentes.
> - **DaemonSet**: para agentes que deben correr en cada nodo del cluster.
> - **Requests y Limits**: siempre defínelos. Sin `requests`, el scheduler no puede planificar bien. Sin `limits`, un pod puede consumir todos los recursos del nodo.

### Recursos
- 🌐 kubernetes.io/docs/concepts/workloads
- 📖 *Kubernetes in Action* — Marko Luksa, Capítulos 4-9
- 🌐 learnk8s.io — guías prácticas sobre K8s

---
`#kubernetes` `#workloads` `#deployment` `#statefulset` `#pod`
