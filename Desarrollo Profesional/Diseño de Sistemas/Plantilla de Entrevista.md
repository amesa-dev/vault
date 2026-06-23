# 📋 Plantilla de Diseño en Entrevista

[[Desarrollo Profesional/Diseño de Sistemas/Diseño de Sistemas|⬅️ Volver a Diseño de Sistemas]]

> [!abstract] Cómo usar esta plantilla
> Esta es una **worksheet de relleno**: un guion paso a paso para abordar cualquier pregunta de diseño de sistemas en una entrevista (o un diseño real) sin quedarte en blanco ni olvidar nada. Copia el bloque de abajo, pégalo y rellénalo para el problema concreto. Se apoya en el método de la [[Desarrollo Profesional/Diseño de Sistemas/Páginas/01 - El Método y Estimaciones|página 01]] y enlaza con los casos resueltos. La idea es que tengas siempre **una estructura que seguir y una checklist de cosas que mencionar**.

---

## El Guion de 6 Pasos (con tiempos para ~45 min)

### Paso 1 — Aclarar requisitos y alcance (~5 min)
**No asumas. Pregunta.** Demuestra que acotas antes de diseñar.
- ¿Qué **features** exactas entran en el alcance? (acota a 2-3 núcleo)
- ¿Cuántos **usuarios** (DAU/MAU)? ¿Crecimiento esperado?
- ¿**Lectura** o **escritura** intensiva? ¿Qué ratio?
- ¿Qué es lo **crítico**: latencia, consistencia, disponibilidad, coste?
- ¿Hay restricciones especiales (tiempo real, geografía, regulación)?

> Frase útil: *"Antes de diseñar, déjame confirmar el alcance y la escala…"*

### Paso 2 — Requisitos funcionales y no funcionales (escríbelos)
- **Funcionales** (qué hace): lista de operaciones núcleo.
- **No funcionales** (cómo de bien): escala, latencia objetivo (p99), disponibilidad (¿99,9%? ¿99,99%?), modelo de consistencia, durabilidad. *Aquí se juega el diseño.*

### Paso 3 — Estimaciones back-of-the-envelope (~5 min)
Calcula solo lo que cambie una decisión:
- **QPS**: usuarios × actividad / 86.400 (≈10⁵). Y el **pico** (×2-10).
- **Almacenamiento**: volumen/día × tamaño × retención.
- **Ancho de banda**: QPS × tamaño de payload.
- **Memoria de caché**: ~20% del working set caliente (regla 80/20).
> Cada número debe llevar a una conclusión ("230k QPS lectura → necesito caché + réplicas").

### Paso 4 — Diseño de alto nivel (~10-15 min)
- Define la **API** (los endpoints/operaciones principales, con sus params).
- Esboza el **modelo de datos** (entidades y relaciones clave).
- Dibuja los **componentes** y el **flujo de datos**: cliente → CDN/LB → servicios → caché → BD → colas.
- Acuerda este esqueleto antes de profundizar.

### Paso 5 — Profundizar (~10-15 min)
Elige los 1-2 componentes más interesantes/críticos y entra a fondo:
- El **algoritmo clave** (rate limiting, fan-out, geohash, transcodificación…).
- El **esquema de datos** y cómo se **shardea** (shard key, consistent hashing).
- Cómo se **cachea** y se **invalida**.
- El manejo de **concurrencia/consistencia** donde haya estado compartido.

### Paso 6 — Cuellos de botella, fallos y cierre (~5 min)
- ¿Dónde se rompe esto a **10× la escala**?
- **Single points of failure** y cómo los eliminas (redundancia).
- **Resiliencia**: timeouts, reintentos, circuit breakers, degradación.
- **Observabilidad**: qué métricas/alertas (latencia, errores, saturación).
- Resume los **trade-offs** que tomaste y por qué.

---

## Checklist de Building Blocks (¿lo he considerado?)

Repasa mentalmente estas piezas en todo diseño — no todas aplican, pero pregúntate por cada una:

- [ ] **Load balancer** (L4/L7) y servidores **stateless**
- [ ] **Caché** (niveles: cliente, CDN, app/Redis, BD) y su **invalidación**
- [ ] **CDN** para contenido estático/geográfico
- [ ] **Base de datos**: ¿SQL o NoSQL? ¿por qué?
- [ ] **Réplicas** de lectura (escala lectura) y **sharding** (escala escritura)
- [ ] **Consistent hashing** si reparto datos/caché dinámicamente
- [ ] **Cola de mensajes** para desacoplar y procesar asíncrono lo lento
- [ ] **Modelo de consistencia**: ¿fuerte o eventual? ([[Desarrollo Profesional/Sistemas Distribuidos/Páginas/01 - Fundamentos y CAP|CAP/PACELC]])
- [ ] **Idempotencia** donde haya reintentos/escrituras críticas
- [ ] **Rate limiting** y protección de la API
- [ ] **Generación de IDs** (¿necesito únicos/ordenables? → Snowflake)
- [ ] **Resiliencia**: timeouts, reintentos+backoff, circuit breaker, bulkhead
- [ ] **Observabilidad**: métricas, logs, trazas, alertas
- [ ] **Seguridad**: authn/authz, secretos, datos sensibles

---

## Errores Frecuentes a Evitar

- ❌ **Lanzarte a dibujar** sin aclarar requisitos ni estimar.
- ❌ **Sobre-ingeniería**: meter todas las piezas "porque sí". Justifica cada una por un requisito real.
- ❌ **Ignorar los no funcionales**: diseñar para 100 usuarios cuando piden 100M (o al revés).
- ❌ **No mencionar trade-offs**: toda decisión tiene un coste; nómbralo (consistencia vs latencia, etc.).
- ❌ **Olvidar los fallos**: no decir qué pasa cuando un componente cae.
- ❌ **Silencio**: piensa en voz alta. El entrevistador evalúa tu razonamiento, no solo el resultado.

---

## Índice rápido de casos resueltos

| Patrón que enseña | Caso |
|-------------------|------|
| Algoritmos + distribuido | [[Desarrollo Profesional/Diseño de Sistemas/Páginas/05 - Caso Rate Limiter\|Rate Limiter]] |
| Generación de IDs, lectura masiva | [[Desarrollo Profesional/Diseño de Sistemas/Páginas/06 - Caso URL Shortener\|URL Shortener]] |
| Fan-out write vs read | [[Desarrollo Profesional/Diseño de Sistemas/Páginas/07 - Caso News Feed\|News Feed]] |
| Tiempo real, WebSockets, estado | [[Desarrollo Profesional/Diseño de Sistemas/Páginas/08 - Caso Chat en Tiempo Real\|Chat]] |
| Desacoplo, terceros poco fiables | [[Desarrollo Profesional/Diseño de Sistemas/Páginas/09 - Caso Sistema de Notificaciones\|Notificaciones]] |
| Precálculo offline, trie | [[Desarrollo Profesional/Diseño de Sistemas/Páginas/10 - Caso Autocompletado\|Autocompletado]] |
| IDs únicos ordenables | [[Desarrollo Profesional/Diseño de Sistemas/Páginas/11 - Caso Generador de IDs\|Generador de IDs]] |
| BFS distribuido, cortesía | [[Desarrollo Profesional/Diseño de Sistemas/Páginas/12 - Caso Web Crawler\|Web Crawler]] |
| CDN, procesamiento pesado | [[Desarrollo Profesional/Diseño de Sistemas/Páginas/13 - Caso YouTube y Streaming\|YouTube]] |
| Sincronización, bloques | [[Desarrollo Profesional/Diseño de Sistemas/Páginas/14 - Caso Google Drive\|Google Drive]] |
| Índices geoespaciales | [[Desarrollo Profesional/Diseño de Sistemas/Páginas/15 - Caso Proximidad Geoespacial\|Proximidad]] |
| Corrección, idempotencia, ledger | [[Desarrollo Profesional/Diseño de Sistemas/Páginas/16 - Caso Sistema de Pagos\|Pagos]] |

---

## Bloque para copiar y rellenar

```markdown
## Diseño: <NOMBRE DEL SISTEMA>

### 1. Requisitos
Funcionales:
-
No funcionales (escala, latencia, disponibilidad, consistencia):
-

### 2. Estimaciones
- DAU: ____   QPS lectura: ____   QPS escritura: ____   Pico: ____
- Almacenamiento (X años): ____
- Conclusiones de diseño: ____

### 3. API
-

### 4. Modelo de datos
-

### 5. Diagrama de alto nivel
(cliente → LB → servicios → caché → BD / colas)

### 6. Profundización
- Componente crítico 1: ____
- Sharding / caché / consistencia: ____

### 7. Cuellos de botella y trade-offs
- A 10× escala se rompe en: ____
- SPOFs y mitigación: ____
- Trade-offs asumidos: ____
```

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - Sigue **6 pasos**: aclarar requisitos → funcionales/no funcionales → estimar → diseño de alto nivel → profundizar → cuellos de botella y cierre. Nunca empieces por dibujar.
> - Repasa la **checklist de building blocks** en cada diseño (LB, caché, CDN, SQL/NoSQL, réplicas/sharding, colas, consistencia, idempotencia, resiliencia, observabilidad, seguridad) — pero justifica cada pieza por un requisito.
> - Evita los errores típicos: dibujar sin aclarar, sobre-ingeniería, ignorar no funcionales, no nombrar trade-offs, olvidar los fallos y quedarte en silencio.
> - Usa el **bloque de relleno** para estructurar la respuesta y los **casos resueltos** como referencia de patrones.

### Recursos
- 📖 *System Design Interview, Vol. 1 y 2* — Alex Xu (el framework del capítulo 3)
- 🌐 github.com/donnemartin/system-design-primer
- 🌐 bytebytego.com — material y diagramas de Alex Xu

---
`#diseño-de-sistemas` `#entrevistas` `#plantilla` `#checklist`
