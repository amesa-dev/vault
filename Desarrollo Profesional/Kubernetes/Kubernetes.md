# ☸️ Kubernetes (K8s)

[[Desarrollo Profesional/Inicio Profesional|⬅️ Volver a Desarrollo Profesional]]

> [!abstract] Kubernetes
> Plataforma de código abierto para automatizar la implementación, el escalado y la administración de aplicaciones en contenedores.

---

## 🏗️ Conceptos Clave
- **Pod:** La unidad mínima de despliegue que comparte red y almacenamiento.
- **Service:** Abstracción para exponer un conjunto de Pods como un servicio de red.
- **Deployment:** Define el estado deseado para tus Pods y réplicas.
- **Namespace:** División física/lógica del clúster para organizar recursos.

---

## ⚡ Comandos Esenciales de `kubectl`

### Inspección de Recursos
```bash
# Listar recursos básicos
kubectl get pods
kubectl get services
kubectl get deployments -n <namespace>

# Obtener información detallada de un recurso
kubectl describe pod <nombre-pod>
```

### Depuración y Logs
```bash
# Ver los logs de un contenedor
kubectl logs <nombre-pod>

# Seguir los logs en tiempo real
kubectl logs -f <nombre-pod> -c <nombre-contenedor>

# Entrar a la consola interactiva de un Pod
kubectl exec -it <nombre-pod> -- /bin/sh
```

### Aplicar Configuraciones
```bash
# Crear o actualizar recursos definidos en un archivo YAML
kubectl apply -f manifest.yaml

# Eliminar recursos definidos en un YAML
kubectl delete -f manifest.yaml
```

---
`#kubernetes` `#k8s` `#devops` `#apuntes`
