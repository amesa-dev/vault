# 🔀 CQRS — Command Query Responsibility Segregation

[[Desarrollo Profesional/Inicio Profesional|⬅️ Volver a Desarrollo Profesional]]

> [!abstract] Sobre esta sección
> CQRS es un patrón engañosamente simple: separa las operaciones que **cambian** el estado (commands) de las que solo lo **leen** (queries), y deja que cada lado evolucione por su cuenta. La idea nació de Bertrand Meyer (CQS) y la popularizó Greg Young para sistemas. Bien aplicado, te da modelos de lectura ultrarrápidos y un modelo de escritura limpio y rico en reglas; mal aplicado, es complejidad gratuita. Esta sección cubre el porqué, el cómo en Python y cuándo conviene (y cuándo no).

---

## 📚 Páginas de esta sección

1. [[Desarrollo Profesional/CQRS/Páginas/01 - Concepto y Motivación|01 — Concepto y Motivación]] — CQS vs CQRS, commands vs queries, por qué separar y los niveles de adopción
2. [[Desarrollo Profesional/CQRS/Páginas/02 - Implementación en Python|02 — Implementación en Python]] — buses, handlers, el mediator y cómo cablearlo todo
3. [[Desarrollo Profesional/CQRS/Páginas/03 - Read Models Eventos y Escalado|03 — Read Models, Eventos y Escalado]] — proyecciones, consistencia eventual, relación con Event Sourcing y cuándo NO usarlo

---

## ¿Qué es CQRS en una frase?

CQRS es decidir que preguntar y ordenar son responsabilidades tan distintas que merecen modelos, caminos de código y, a veces, bases de datos diferentes. Es el desarrollo natural del [[Desarrollo Profesional/DDD/Páginas/03 - Patrones Avanzados|patrón CQRS que vimos en DDD]], llevado a su propia sección.

---

## Ruta recomendada

Si vienes de [[Desarrollo Profesional/DDD/DDD|DDD]], la página 01 te sonará: aquí la profundizamos. Si ya tienes claro el concepto, salta a la 02 (código) y la 03 (escalado). CQRS encaja con [[Desarrollo Profesional/Arquitectura Hexagonal/Arquitectura Hexagonal|Arquitectura Hexagonal]] y con la mensajería de [[Desarrollo Profesional/Sistemas Distribuidos/Sistemas Distribuidos|Sistemas Distribuidos]].

---
`#cqrs` `#arquitectura` `#patrones` `#indice`
