# 🟥 Redis

[[Desarrollo Profesional/Inicio Profesional|⬅️ Volver a Desarrollo Profesional]]

> [!abstract] Sobre esta sección
> Redis es un almacén de estructuras de datos en memoria que se usa como caché, broker de mensajes, contador, sistema de locks distribuidos y mucho más. Su superpoder es la latencia: al vivir en RAM y ser de un solo hilo para los comandos, ofrece operaciones en microsegundos con semántica atómica. Pero esa misma simplicidad esconde decisiones importantes: ¿qué pasa si se reinicia? ¿cómo invalido la caché? ¿es seguro mi lock distribuido? Esta sección cubre tanto las estructuras como los patrones de uso que aparecen una y otra vez en sistemas a escala.

---

## 📚 Páginas de esta sección

1. [[Desarrollo Profesional/Redis/Páginas/01 - Estructuras y Modelo|01 — Estructuras y Modelo]] — tipos de datos, atomicidad, persistencia, expiración, eviction
2. [[Desarrollo Profesional/Redis/Páginas/02 - Patrones de Uso|02 — Patrones de Uso]] — caching, locks distribuidos, pub/sub, streams, rate limiting

---

## ¿Qué resuelve esta sección en una frase?

Te da las herramientas para quitar carga de tu base de datos, coordinar procesos distribuidos y reaccionar a eventos con latencia de microsegundos — sabiendo exactamente qué garantías tienes y cuáles no. Es una pieza recurrente en [[Desarrollo Profesional/Diseño de Sistemas/Diseño de Sistemas|Diseño de Sistemas]].

---
`#redis` `#cache` `#in-memory` `#indice`
