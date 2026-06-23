# ☸️ K8s 01 — Arquitectura Core

[[Desarrollo Profesional/Kubernetes/Kubernetes|⬅️ Volver a Kubernetes]] | [[Desarrollo Profesional/Kubernetes/Páginas/02 - Workloads|02 →]]

> [!abstract] Introducción
> Kubernetes es un sistema distribuido complejo. Entender su arquitectura interna no es solo un ejercicio académico — es lo que te permite diagnosticar por qué un pod no arranca, por qué el scheduler no está distribuyendo carga, o por qué el cluster tarda en reflejarse los cambios. Hay dos tipos de nodos: el control plane (el cerebro) y los worker nodes (donde corre tu aplicación).

## ¿De qué vamos a hablar?

Los componentes del control plane y de los worker nodes, cómo interactúan entre sí, y el ciclo de vida de un Pod desde que haces `kubectl apply` hasta que el contenedor está corriendo.

### Conceptos que vamos a cubrir
- Control plane: API Server, etcd, Scheduler, Controller Manager
- Worker nodes: kubelet, kube-proxy, container runtime
- El ciclo de vida de un Pod
- Namespaces como unidad de aislamiento

---

## El Concepto

### La Arquitectura

```
┌─────────────────────────────────────────────────────────────┐
│                    CONTROL PLANE                            │
│                                                             │
│  ┌────────────┐  ┌────────────┐  ┌─────────────────────┐  │
│  │ API Server │  │   etcd     │  │ Controller Manager  │  │
│  │            │◄─┤ (estado)   │  │ (reconciliación)    │  │
│  │ :6443      │  └────────────┘  └─────────────────────┘  │
│  └─────┬──────┘                                            │
│        │              ┌─────────────────┐                  │
│        │              │   Scheduler     │                  │
│        │              │ (asignación)    │                  │
│        │              └────────────────-┘                  │
└────────┼────────────────────────────────────────────────────┘
         │  HTTPS
         ▼
┌─────────────────────────────────────────────────────────────┐
│                    WORKER NODE                              │
│                                                             │
│  ┌────────────────┐   ┌──────────────┐  ┌──────────────┐  │
│  │    kubelet     │   │  kube-proxy  │  │  container   │  │
│  │ (gestiona pods)│   │  (iptables)  │  │  runtime     │  │
│  └────────────────┘   └──────────────┘  │ (containerd) │  │
│                                         └──────────────┘  │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐                    │
│  │  Pod A  │  │  Pod B  │  │  Pod C  │                    │
│  └─────────┘  └─────────┘  └─────────┘                    │
└─────────────────────────────────────────────────────────────┘
```

### Control Plane — El Cerebro

**API Server (`kube-apiserver`):**
- El único componente del control plane con el que interactúan los clientes (kubectl, el kubelet, los controllers)
- Valida y persiste todos los objetos en etcd
- Expone la API REST de Kubernetes en el puerto 6443

**etcd:**
- Base de datos distribuida de clave-valor que almacena todo el estado del cluster
- Consistencia fuerte: Raft consensus
- Si etcd falla, el cluster no puede cambiar de estado (aunque los pods existentes siguen corriendo)
- **Nunca accedas a etcd directamente** — usa siempre el API Server

**Scheduler (`kube-scheduler`):**
- Observa pods sin nodo asignado (`spec.nodeName` vacío)
- Decide qué nodo es el mejor para cada pod basándose en recursos disponibles, affinity, taints/tolerations
- No inicia el pod — solo lo asigna. El kubelet del nodo lo inicia

**Controller Manager (`kube-controller-manager`):**
- Múltiples controllers en un solo binario: Deployment Controller, ReplicaSet Controller, Node Controller, etc.
- Implementa el **reconciliation loop**: observa el estado actual del cluster, lo compara con el estado deseado, y actúa para cerrar la brecha

### Worker Nodes

**kubelet:**
- Agente que corre en cada worker node
- Recibe del API Server la especificación de los pods que debe ejecutar en su nodo
- Instruye al container runtime para iniciar/parar contenedores
- Reporta el estado del pod de vuelta al API Server

**kube-proxy:**
- Mantiene las reglas de red (iptables/IPVS) en cada nodo para implementar los Services de Kubernetes
- Cuando un pod llama a un Service, kube-proxy redirige la petición a uno de los pods que hay detrás

**Container Runtime (containerd):**
- El componente que realmente ejecuta los contenedores
- Kubernetes habla con él a través de la Container Runtime Interface (CRI)
- GKE usa containerd desde 2023 (Docker fue deprecado como runtime)

### El Ciclo de Vida de un Pod

```
kubectl apply -f pod.yaml
       │
       ▼
API Server  ────── valida el manifiesto ────── lo guarda en etcd
       │
       ▼
Scheduler  ────── observa pod sin nodo ────── elige el nodo ────── actualiza etcd
       │
       ▼
kubelet (nodo elegido) ─── detecta el cambio ─── pide al container runtime
       │                   que inicie los contenedores
       ▼
containerd ─── descarga la imagen ─── crea el contenedor ─── lo arranca
       │
       ▼
kubelet ─── reporta status = Running al API Server ─── etcd actualizado
```

### Namespaces

Los namespaces dividen lógicamente el cluster. No son aislamiento de red real (para eso se necesita NetworkPolicy), pero sí limitan el scope de los recursos:

```bash
# Ver namespaces del sistema
kubectl get namespaces
# kube-system   — componentes del sistema (CoreDNS, kube-proxy)
# default       — namespace por defecto
# kube-public   — recursos públicamente accesibles
# kube-node-lease

# Crear namespace para un equipo/entorno
kubectl create namespace produccion
kubectl create namespace staging

# Operar en un namespace específico
kubectl get pods -n produccion
kubectl get all -n kube-system

# Configurar namespace por defecto en el contexto
kubectl config set-context --current --namespace=produccion
```

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - El **API Server** es el punto único de entrada al control plane. Todo (kubectl, kubelet, controllers) pasa por él.
> - **etcd** es la fuente de verdad del cluster. Su pérdida es catastrófica — en producción debe ser un cluster etcd de al menos 3 nodos.
> - El **Scheduler** asigna pods a nodos; el **kubelet** los inicia. Son componentes distintos con responsabilidades distintas.
> - Los controllers implementan el **reconciliation loop**: observa → compara estado deseado vs actual → actúa. Es la idea central de Kubernetes.
> - Los namespaces organizan recursos, pero **no son aislamiento de red**. Para eso necesitas NetworkPolicy.

### Recursos
- 🌐 kubernetes.io/docs/concepts/architecture
- 📖 *Kubernetes in Action* — Marko Luksa (el libro más completo sobre K8s internos)
- 🌐 iximiuz.com — artículos muy buenos sobre internos de Kubernetes

---
`#kubernetes` `#arquitectura` `#control-plane` `#etcd`
