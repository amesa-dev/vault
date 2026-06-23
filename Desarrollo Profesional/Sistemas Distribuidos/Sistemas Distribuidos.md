# 🌐 Sistemas Distribuidos y Mensajería

[[Desarrollo Profesional/Inicio Profesional|⬅️ Volver a Desarrollo Profesional]]

> [!abstract] Sobre esta sección
> Un sistema distribuido es un conjunto de procesos que se comunican por una red poco fiable para parecer un único sistema coherente. Toda la dificultad nace de ahí: la red falla, los mensajes se duplican o se pierden, los relojes no están sincronizados y no hay un "ahora" global. Esta sección cubre las leyes que gobiernan esos sistemas (CAP, consistencia) y la maquinaria práctica para construirlos: brokers de mensajería, el patrón Outbox, idempotencia y Sagas. Es la capa de infraestructura que hace reales los Domain Events, CQRS y Event Sourcing de [[Desarrollo Profesional/DDD/Páginas/03 - Patrones Avanzados|DDD]].

---

## 📚 Páginas de esta sección

1. [[Desarrollo Profesional/Sistemas Distribuidos/Páginas/01 - Fundamentos y CAP|01 — Fundamentos y CAP]] — falacias de la red, CAP, niveles de consistencia, relojes y orden
2. [[Desarrollo Profesional/Sistemas Distribuidos/Páginas/02 - Mensajería y Brokers|02 — Mensajería y Brokers]] — colas vs logs, Kafka/RabbitMQ/Pub/Sub, garantías de entrega, Outbox
3. [[Desarrollo Profesional/Sistemas Distribuidos/Páginas/03 - Transacciones Distribuidas|03 — Transacciones Distribuidas]] — el problema del 2PC, Sagas, idempotencia, exactly-once

---

## ¿Qué resuelve esta sección en una frase?

Te da el modelo mental para razonar sobre qué garantías puedes y no puedes tener cuando los datos viven en más de una máquina, y los patrones concretos para coordinar trabajo entre servicios sin corromper el estado. Conecta con [[Desarrollo Profesional/Patrones de Resiliencia/Patrones de Resiliencia|Patrones de Resiliencia]] (qué hacer cuando un nodo falla) y con [[Desarrollo Profesional/Diseño de Sistemas/Diseño de Sistemas|Diseño de Sistemas]] (cómo se combinan en un sistema real).

---
`#sistemas-distribuidos` `#mensajeria` `#arquitectura` `#indice`
