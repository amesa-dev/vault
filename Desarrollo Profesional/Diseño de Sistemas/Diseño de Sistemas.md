# 🏛️ Diseño de Sistemas

[[Desarrollo Profesional/Inicio Profesional|⬅️ Volver a Desarrollo Profesional]]

> [!abstract] Sobre esta sección
> Esta es la sección que integra todo lo demás. Diseñar un sistema es el arte de combinar bases de datos, cachés, colas, balanceadores y servicios para cumplir unos requisitos de escala, latencia y disponibilidad — tomando decisiones explícitas sobre los compromisos (recuerda [[Desarrollo Profesional/Sistemas Distribuidos/Páginas/01 - Fundamentos y CAP|CAP/PACELC]]: no puedes tenerlo todo). Está construida al estilo de los libros *System Design Interview* de Alex Xu: primero el método y los bloques de construcción, luego una batería de **casos de estudio** desarrollados de principio a fin —requisitos, estimaciones, diseño de alto nivel, profundización y cuellos de botella—. Útil tanto para diseñar sistemas reales como para entrevistas.

---

## 📚 Páginas de esta sección

### 📋 Plantilla
- [[Desarrollo Profesional/Diseño de Sistemas/Plantilla de Entrevista|Plantilla de Diseño en Entrevista]] — worksheet de relleno, guion de 6 pasos y checklist de building blocks

### Fundamentos
1. [[Desarrollo Profesional/Diseño de Sistemas/Páginas/01 - El Método y Estimaciones|01 — El Método y Estimaciones]] — el framework de 4 pasos, back-of-the-envelope, los números que hay que saber
2. [[Desarrollo Profesional/Diseño de Sistemas/Páginas/02 - Escalar de Cero a Millones|02 — Escalar de Cero a Millones]] — la evolución de un servidor único a una arquitectura para millones
3. [[Desarrollo Profesional/Diseño de Sistemas/Páginas/03 - Almacenamiento Replicación y Sharding|03 — Almacenamiento, Replicación y Sharding]] — SQL/NoSQL, réplicas, particionado, consistent hashing
4. [[Desarrollo Profesional/Diseño de Sistemas/Páginas/04 - Caché CDN Balanceo y Colas|04 — Caché, CDN, Balanceo y Colas]] — los building blocks de rendimiento y desacoplo

### Casos de estudio
5. [[Desarrollo Profesional/Diseño de Sistemas/Páginas/05 - Caso Rate Limiter|05 — Caso: Rate Limiter]] — algoritmos, dónde colocarlo, en distribuido
6. [[Desarrollo Profesional/Diseño de Sistemas/Páginas/06 - Caso URL Shortener|06 — Caso: Acortador de URLs]] — generación de IDs, redirección, escala de lectura
7. [[Desarrollo Profesional/Diseño de Sistemas/Páginas/07 - Caso News Feed|07 — Caso: News Feed / Timeline]] — fan-out on write vs on read, el problema de las celebridades
8. [[Desarrollo Profesional/Diseño de Sistemas/Páginas/08 - Caso Chat en Tiempo Real|08 — Caso: Chat en Tiempo Real]] — WebSockets, presencia, entrega y orden de mensajes
9. [[Desarrollo Profesional/Diseño de Sistemas/Páginas/09 - Caso Sistema de Notificaciones|09 — Caso: Sistema de Notificaciones]] — multicanal, colas, integración con terceros, fiabilidad
10. [[Desarrollo Profesional/Diseño de Sistemas/Páginas/10 - Caso Autocompletado|10 — Caso: Autocompletado / Typeahead]] — trie, top-k precalculadas, offline vs online
11. [[Desarrollo Profesional/Diseño de Sistemas/Páginas/11 - Caso Generador de IDs|11 — Caso: Generador de IDs Únicos]] — Snowflake, IDs distribuidos y ordenables
12. [[Desarrollo Profesional/Diseño de Sistemas/Páginas/12 - Caso Web Crawler|12 — Caso: Web Crawler]] — BFS distribuido, frontera de URLs, cortesía, dedup
13. [[Desarrollo Profesional/Diseño de Sistemas/Páginas/13 - Caso YouTube y Streaming|13 — Caso: YouTube / Streaming]] — transcodificación, CDN, streaming adaptativo
14. [[Desarrollo Profesional/Diseño de Sistemas/Páginas/14 - Caso Google Drive|14 — Caso: Google Drive]] — bloques, delta sync, metadatos vs contenido, conflictos
15. [[Desarrollo Profesional/Diseño de Sistemas/Páginas/15 - Caso Proximidad Geoespacial|15 — Caso: Proximidad Geoespacial]] — geohash, quadtree, objetos en movimiento
16. [[Desarrollo Profesional/Diseño de Sistemas/Páginas/16 - Caso Sistema de Pagos|16 — Caso: Sistema de Pagos]] — idempotencia, ledger de doble entrada, reconciliación

---

## ¿Qué resuelve esta sección en una frase?

Te entrena para pasar de un requisito difuso ("quiero un Twitter") a una arquitectura justificada, razonando sobre escala con números y eligiendo cada pieza por sus compromisos. Es la culminación de [[Desarrollo Profesional/Sistemas Distribuidos/Sistemas Distribuidos|Sistemas Distribuidos]], [[Desarrollo Profesional/Redis/Redis|Redis]], [[Desarrollo Profesional/PostgreSQL/PostgreSQL|PostgreSQL]] y [[Desarrollo Profesional/Patrones de Resiliencia/Patrones de Resiliencia|Resiliencia]].

---
`#diseño-de-sistemas` `#system-design` `#arquitectura` `#escalabilidad` `#indice`
