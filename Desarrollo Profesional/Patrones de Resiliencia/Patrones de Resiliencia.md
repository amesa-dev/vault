# 🛡️ Patrones de Resiliencia

[[Desarrollo Profesional/Inicio Profesional|⬅️ Volver a Desarrollo Profesional]]

> [!abstract] Sobre esta sección
> En un sistema distribuido los fallos no son la excepción: son el modo de operación normal. Una dependencia se cae, una llamada tarda 30 segundos, un servicio recibe el triple de tráfico del esperado. La resiliencia es la propiedad de un sistema que **sigue dando un servicio aceptable cuando sus partes fallan**, en lugar de derrumbarse en cascada. Esta sección recoge los patrones tácticos —timeouts, reintentos, circuit breakers, bulkheads, rate limiting— que convierten fallos catastróficos en degradaciones controladas. Es el complemento natural de [[Desarrollo Profesional/Sistemas Distribuidos/Sistemas Distribuidos|Sistemas Distribuidos]].

---

## 📚 Páginas de esta sección

1. [[Desarrollo Profesional/Patrones de Resiliencia/Páginas/01 - Timeouts Reintentos y Backoff|01 — Timeouts, Reintentos y Backoff]] — el primer perímetro de defensa frente a fallos transitorios
2. [[Desarrollo Profesional/Patrones de Resiliencia/Páginas/02 - Circuit Breaker y Aislamiento|02 — Circuit Breaker y Aislamiento]] — circuit breaker, bulkhead, rate limiting, degradación elegante

---

## ¿Qué resuelve esta sección en una frase?

Te da las herramientas para que un fallo en una parte del sistema **no se propague** y tumbe el resto — para que tu servicio degrade en lugar de morir. Estos patrones asumen las verdades de [[Desarrollo Profesional/Sistemas Distribuidos/Páginas/01 - Fundamentos y CAP|las falacias de la red]] y se apoyan en la idempotencia vista en [[Desarrollo Profesional/Sistemas Distribuidos/Páginas/03 - Transacciones Distribuidas|Transacciones Distribuidas]].

---
`#resiliencia` `#fiabilidad` `#arquitectura` `#indice`
