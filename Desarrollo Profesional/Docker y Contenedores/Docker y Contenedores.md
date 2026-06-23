# 🐳 Docker y Contenedores

[[Desarrollo Profesional/Inicio Profesional|⬅️ Volver a Desarrollo Profesional]]

> [!abstract] Sobre esta sección
> [[Desarrollo Profesional/Kubernetes/Kubernetes|Kubernetes]] orquesta contenedores, pero da por sentado que entiendes qué *es* un contenedor. Esta sección llena ese hueco fundamental: un contenedor no es una máquina virtual ligera, es un proceso normal de Linux aislado mediante dos primitivas del kernel (namespaces y cgroups). Entender esto —y cómo se construyen las imágenes por capas— cambia la forma en que escribes Dockerfiles, depuras problemas y razonas sobre seguridad y rendimiento.

---

## 📚 Páginas de esta sección

1. [[Desarrollo Profesional/Docker y Contenedores/Páginas/01 - Fundamentos e Imágenes|01 — Fundamentos e Imágenes]] — qué es un contenedor de verdad, namespaces, cgroups, capas
2. [[Desarrollo Profesional/Docker y Contenedores/Páginas/02 - Dockerfiles y Buenas Prácticas|02 — Dockerfiles y Buenas Prácticas]] — multi-stage, caché de capas, imágenes mínimas, seguridad, compose

---

## ¿Qué resuelve esta sección en una frase?

Te da el modelo mental correcto de qué es un contenedor (un proceso aislado, no una VM) y las técnicas para construir imágenes pequeñas, rápidas de construir y seguras — la base sobre la que se apoyan [[Desarrollo Profesional/Kubernetes/Kubernetes|Kubernetes]] y [[Desarrollo Profesional/CI-CD e IaC/CI-CD e IaC|CI/CD]].

---
`#docker` `#contenedores` `#devops` `#indice`
