# ☸️ K8s 04 — Configuración y Operación

[[Desarrollo Profesional/Kubernetes/Kubernetes|⬅️ Volver a Kubernetes]] | [[Desarrollo Profesional/Kubernetes/Páginas/03 - Networking|← 03]]

> [!abstract] Introducción
> La configuración y la operación diaria de Kubernetes implica saber gestionar Secrets y ConfigMaps, entender el escalado automático (HPA), y tener los comandos de kubectl necesarios para diagnosticar y resolver problemas. Esta página cubre también RBAC — cómo controlar quién puede hacer qué en el cluster.

## ¿De qué vamos a hablar?

ConfigMap y Secret para gestión de configuración, HPA para escalado automático, RBAC para control de acceso, y los comandos de kubectl más útiles para el día a día.

---

## El Concepto

### ConfigMap — Configuración No Sensible

```yaml
# Crear ConfigMap con valores literales
apiVersion: v1
kind: ConfigMap
metadata:
  name: mi-app-config
  namespace: produccion
data:
  LOG_LEVEL: "INFO"
  MAX_CONNECTIONS: "100"
  FEATURE_FLAG_NUEVO_CHECKOUT: "true"
  app.properties: |
    servidor.timeout=30
    cache.ttl=3600
    api.version=v2

---
# Usar el ConfigMap en un Pod
spec:
  containers:
    - name: app
      image: mi-app:latest
      # Opción 1: variables de entorno individuales
      env:
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: mi-app-config
              key: LOG_LEVEL
      # Opción 2: todas las claves como variables de entorno
      envFrom:
        - configMapRef:
            name: mi-app-config
      # Opción 3: montar como fichero
      volumeMounts:
        - name: config-vol
          mountPath: /etc/config
  volumes:
    - name: config-vol
      configMap:
        name: mi-app-config
```

```bash
# Crear ConfigMap desde la línea de comandos
kubectl create configmap mi-config \
  --from-literal=LOG_LEVEL=INFO \
  --from-file=app.properties=./app.properties \
  -n produccion

# Editar en caliente (solo surte efecto en pods nuevos o si la app observa cambios)
kubectl edit configmap mi-app-config -n produccion
```

### Secrets — Configuración Sensible

Los Secrets almacenan datos sensibles codificados en base64. No están cifrados por defecto en etcd — en GKE puedes habilitar cifrado con CMEK (Customer-Managed Encryption Keys):

```yaml
# Secret declarativo — valores en base64
apiVersion: v1
kind: Secret
metadata:
  name: mi-app-secrets
  namespace: produccion
type: Opaque
data:
  DATABASE_URL: cG9zdGdyZXNxbDovL3VzZXI6cGFzc0Bob3N0L2Ri  # base64
  API_KEY: c2VjcmV0LWtleQ==

---
# Mejor: usar stringData (K8s hace el base64 automáticamente)
apiVersion: v1
kind: Secret
metadata:
  name: mi-app-secrets
  namespace: produccion
type: Opaque
stringData:
  DATABASE_URL: "postgresql://user:pass@host/db"
  API_KEY: "secret-key"
```

```bash
# Crear Secret desde la CLI
kubectl create secret generic mi-app-secrets \
  --from-literal=DATABASE_URL="postgresql://user:pass@host/db" \
  --from-literal=API_KEY="secret-key" \
  -n produccion

# En GKE — External Secrets Operator para sincronizar con Secret Manager
# Mejor práctica: no guardar secrets en el repositorio git
kubectl create secret generic mi-secret \
  --from-file=credentials.json=./service-account.json \
  -n produccion
```

### HPA — Horizontal Pod Autoscaler

El HPA escala el número de réplicas de un Deployment basándose en métricas:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: mi-api-hpa
  namespace: produccion
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: mi-api
  minReplicas: 2
  maxReplicas: 20
  metrics:
    # Escalar por uso de CPU
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70  # mantener CPU < 70% promedio
    # Escalar por uso de memoria
    - type: Resource
      resource:
        name: memory
        target:
          type: AverageValue
          averageValue: 400Mi
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60    # espera 60s antes de escalar arriba
      policies:
        - type: Pods
          value: 4                       # máximo 4 pods nuevos por ciclo
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300   # espera 5 minutos antes de escalar abajo
```

```bash
# Ver estado del HPA
kubectl get hpa -n produccion
kubectl describe hpa mi-api-hpa -n produccion
```

### RBAC — Control de Acceso

```yaml
# Role — permisos dentro de un namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer-role
  namespace: staging
rules:
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch", "create", "update"]
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["pods/exec"]
    verbs: ["create"]

---
# RoleBinding — asocia el Role a un usuario/SA
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-binding
  namespace: staging
subjects:
  - kind: User
    name: ana@empresa.com
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer-role
  apiGroup: rbac.authorization.k8s.io
```

### kubectl — Comandos del Día a Día

```bash
# ── INSPECCIÓN ──────────────────────────────────────────────────────────────
# Estado general del cluster
kubectl get nodes
kubectl top nodes          # uso de CPU/memoria por nodo (requiere metrics-server)
kubectl top pods -n produccion

# Ver todos los recursos de un namespace
kubectl get all -n produccion

# Detalle de un recurso
kubectl describe pod mi-api-7d8f9b-xyz -n produccion
kubectl describe node mi-node-1

# ── DEBUGGING ────────────────────────────────────────────────────────────────
# Logs de un pod
kubectl logs mi-api-7d8f9b-xyz -n produccion
kubectl logs -f mi-api-7d8f9b-xyz -c contenedor-name  # follow, contenedor específico
kubectl logs --previous mi-api-7d8f9b-xyz              # logs del contenedor anterior (si crasheó)

# Shell en un pod corriendo
kubectl exec -it mi-api-7d8f9b-xyz -n produccion -- /bin/bash

# Pod de debugging temporal
kubectl run debug --image=busybox --rm -it --restart=Never -- sh

# Port forwarding — acceder a un servicio local
kubectl port-forward service/mi-api 8080:80 -n produccion
kubectl port-forward pod/mi-api-7d8f9b-xyz 8080:8080 -n produccion

# ── GESTIÓN DE RECURSOS ──────────────────────────────────────────────────────
# Aplicar manifiestos
kubectl apply -f deployment.yaml
kubectl apply -f ./manifests/  # directorio completo
kubectl apply -k ./kustomize/  # con Kustomize

# Forzar recreación de pods (útil para recargar Secrets/ConfigMaps)
kubectl rollout restart deployment/mi-api -n produccion

# Escalar manualmente
kubectl scale deployment/mi-api --replicas=5 -n produccion

# Ver eventos del cluster (muy útil para debug)
kubectl get events -n produccion --sort-by='.lastTimestamp'

# ── CONTEXTOS ────────────────────────────────────────────────────────────────
kubectl config get-contexts              # ver todos los contextos
kubectl config use-context mi-cluster    # cambiar de cluster
kubectl config set-context --current --namespace=produccion  # namespace por defecto
```

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - **ConfigMap** para configuración no sensible, **Secret** para credenciales. Nunca guardes Secrets en el repositorio git — usa External Secrets Operator con Secret Manager de GCP.
> - El **HPA** requiere que los pods tengan `requests` definidos para funcionar. Sin `requests`, no hay baseline para calcular el porcentaje de uso.
> - **RBAC**: usa `Role` y `RoleBinding` para permisos a nivel de namespace. `ClusterRole` y `ClusterRoleBinding` para permisos globales.
> - `kubectl describe` y `kubectl get events` son tus mejores aliados para diagnosticar por qué un pod no arranca.

### Recursos
- 🌐 kubernetes.io/docs/concepts/configuration
- 🌐 kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters
- 🌐 kubernetes.io/docs/reference/kubectl/cheatsheet — cheatsheet oficial de kubectl

---
`#kubernetes` `#configmap` `#secrets` `#hpa` `#rbac` `#kubectl`
