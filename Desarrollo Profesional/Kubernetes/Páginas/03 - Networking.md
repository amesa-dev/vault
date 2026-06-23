# ☸️ K8s 03 — Networking

[[Desarrollo Profesional/Kubernetes/Kubernetes|⬅️ Volver a Kubernetes]] | [[Desarrollo Profesional/Kubernetes/Páginas/02 - Workloads|← 02]] | [[Desarrollo Profesional/Kubernetes/Páginas/04 - Configuración y Operación|04 →]]

> [!abstract] Introducción
> El networking de Kubernetes es famoso por su complejidad, pero la intuición base es simple: cada Pod tiene su propia IP, y Kubernetes garantiza que todos los pods pueden comunicarse entre sí directamente (flat network). Los Services abstraen la dirección IP efímera de los pods, y el Ingress expone servicios HTTP al exterior.

## ¿De qué vamos a hablar?

Services (ClusterIP, NodePort, LoadBalancer), Ingress, NetworkPolicy y cómo funciona el DNS interno de Kubernetes.

---

## El Concepto

### Services — Abstracciones de Red Estables

Un Pod tiene una IP que cambia cada vez que muere y se recrea. Un Service da una IP virtual estable (ClusterIP) o nombre DNS que enruta tráfico a los pods seleccionados por labels:

```yaml
# ClusterIP — acceso solo desde dentro del cluster (el tipo por defecto)
apiVersion: v1
kind: Service
metadata:
  name: mi-api
  namespace: produccion
spec:
  type: ClusterIP
  selector:
    app: mi-api  # selecciona los pods con este label
  ports:
    - port: 80        # puerto del Service
      targetPort: 8080  # puerto del contenedor
      protocol: TCP

---
# NodePort — expone el servicio en un puerto de cada nodo (30000-32767)
# Para desarrollo/testing, no recomendado en producción
apiVersion: v1
kind: Service
metadata:
  name: mi-api-nodeport
spec:
  type: NodePort
  selector:
    app: mi-api
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 31000  # opcional, K8s asigna uno si se omite

---
# LoadBalancer — crea un LB externo (en GKE, un GCP Load Balancer)
# Tiene coste — usa Ingress cuando puedas (un LB para múltiples servicios)
apiVersion: v1
kind: Service
metadata:
  name: mi-api-lb
spec:
  type: LoadBalancer
  selector:
    app: mi-api
  ports:
    - port: 443
      targetPort: 8080

---
# Headless Service — sin ClusterIP, el DNS devuelve las IPs de los pods directamente
# Usado por StatefulSets para que cada pod tenga su DNS único
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
spec:
  clusterIP: None  # headless
  selector:
    app: postgres
  ports:
    - port: 5432
      targetPort: 5432
# Los pods se acceden como: postgres-0.postgres-headless.default.svc.cluster.local
```

### DNS Interno

Kubernetes tiene un DNS interno (CoreDNS) que resuelve nombres de Services:

```
# Formato del nombre DNS de un Service:
<nombre-service>.<namespace>.svc.cluster.local

# Ejemplos:
mi-api.produccion.svc.cluster.local    # desde otro namespace
mi-api.svc.cluster.local               # desde el mismo namespace
mi-api                                  # desde el mismo namespace (nombre corto)

# Dentro de un pod, esto funcionará:
curl http://mi-api/health                   # mismo namespace
curl http://mi-api.produccion/health        # otro namespace
curl http://mi-api.produccion.svc.cluster.local/health  # nombre completo

# Para StatefulSets — pods accesibles por nombre:
# postgres-0.postgres-headless.default.svc.cluster.local
# postgres-1.postgres-headless.default.svc.cluster.local
```

### Ingress — HTTP Routing

El Ingress expone múltiples Services HTTP bajo un mismo Load Balancer externo. En GKE, el Ingress Controller de Google crea automáticamente un Global HTTP(S) Load Balancer:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mi-ingress
  namespace: produccion
  annotations:
    kubernetes.io/ingress.class: "gce"
    kubernetes.io/ingress.global-static-ip-name: "mi-ip-publica"
    networking.gke.io/managed-certificates: "mi-certificado-ssl"
spec:
  rules:
    - host: api.ejemplo.com
      http:
        paths:
          - path: /v1
            pathType: Prefix
            backend:
              service:
                name: api-v1
                port:
                  number: 80
          - path: /v2
            pathType: Prefix
            backend:
              service:
                name: api-v2
                port:
                  number: 80
    - host: admin.ejemplo.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: admin-ui
                port:
                  number: 80

---
# Certificado SSL gestionado por Google
apiVersion: networking.gke.io/v1
kind: ManagedCertificate
metadata:
  name: mi-certificado-ssl
spec:
  domains:
    - api.ejemplo.com
    - admin.ejemplo.com
```

### NetworkPolicy — Firewall a Nivel de Pod

Por defecto, todos los pods del cluster pueden comunicarse entre sí. NetworkPolicy restringe eso:

```yaml
# Política: los pods de "mi-api" solo pueden recibir tráfico
# de pods con label app=frontend, y solo en el puerto 8080
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: mi-api-ingress-policy
  namespace: produccion
spec:
  podSelector:
    matchLabels:
      app: mi-api
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 8080
  egress:
    # Permite salir solo a PostgreSQL
    - to:
        - podSelector:
            matchLabels:
              app: postgres
      ports:
        - protocol: TCP
          port: 5432
    # Permite DNS (siempre necesario)
    - ports:
        - protocol: UDP
          port: 53
```

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - **ClusterIP** es para comunicación interna. **LoadBalancer** crea un LB externo (con coste). Usa **Ingress** para exponer múltiples servicios HTTP bajo un solo LB.
> - El DNS interno de K8s resuelve `<servicio>.<namespace>.svc.cluster.local`. Dentro del mismo namespace puedes usar solo el nombre corto.
> - **NetworkPolicy** restringe la comunicación entre pods. Sin NetworkPolicy, todos los pods se pueden comunicar libremente — en producción deberías definir políticas.
> - El Ingress en GKE crea automáticamente un Global HTTP(S) Load Balancer — más barato que un LoadBalancer Service por servicio.

### Recursos
- 🌐 kubernetes.io/docs/concepts/services-networking
- 🌐 cilium.io/blog — artículos sobre networking de K8s con Cilium
- 🌐 networkpolicy.io — editor visual de NetworkPolicies

---
`#kubernetes` `#networking` `#service` `#ingress` `#networkpolicy`
