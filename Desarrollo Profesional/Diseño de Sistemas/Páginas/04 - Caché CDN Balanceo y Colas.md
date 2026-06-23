# 🏛️ Diseño de Sistemas 04 — Caché, CDN, Balanceo y Colas

[[Desarrollo Profesional/Diseño de Sistemas/Diseño de Sistemas|⬅️ Volver a Diseño de Sistemas]] | [[Desarrollo Profesional/Diseño de Sistemas/Páginas/03 - Almacenamiento Replicación y Sharding|← 03]] | [[Desarrollo Profesional/Diseño de Sistemas/Páginas/05 - Caso Rate Limiter|05 →]]

> [!abstract] Introducción
> Si la página 03 cubrió dónde viven los datos, esta cubre las piezas que hacen el sistema rápido y robusto: las cachés (en sus distintos niveles), las CDN, los balanceadores de carga (con sus algoritmos) y las colas de mensajes. Son los building blocks que aparecen una y otra vez en los casos de estudio. Ya tocamos varios desde su ángulo específico ([[Desarrollo Profesional/Redis/Redis|Redis]], [[Desarrollo Profesional/Sistemas Distribuidos/Páginas/02 - Mensajería y Brokers|Mensajería]]); aquí los miramos desde la perspectiva del diseñador que decide dónde colocarlos y por qué.

## ¿De qué vamos a hablar?

Los bloques de construcción de rendimiento y desacoplo, y los criterios para colocarlos bien en una arquitectura.

### Conceptos que vamos a cubrir
- Los niveles de caché: del navegador a la base de datos
- CDN: cómo funciona y qué cachear
- Load balancing: algoritmos y capas (L4 vs L7)
- Colas de mensajes en el diseño
- Cómo encajan: una petición de principio a fin

---

## El Concepto

### Los Niveles de Caché

"Caché" no es un sitio, son muchos. Una petición atraviesa varias capas, cada una una oportunidad de no ir más lejos. De más cerca del usuario a más cerca de los datos:

```
[Navegador] → [CDN] → [Load Balancer] → [Caché de app: Redis] → [BD: buffer pool]
   cache         cache       (cache de       cache aplicación      cache interna
  local HTTP    de borde      respuestas)     (cache-aside)         de la BD
```

1. **Caché de cliente/navegador**: cabeceras HTTP (`Cache-Control`, `ETag`) hacen que el navegador no vuelva a pedir lo que no cambió. Gratis y potentísimo para estáticos.
2. **CDN**: caché geográficamente distribuida (siguiente sección).
3. **Caché de aplicación** ([[Desarrollo Profesional/Redis/Páginas/02 - Patrones de Uso|Redis, cache-aside]]): el grueso del trabajo, evita golpear la BD.
4. **Caché de base de datos**: la propia BD cachea en RAM las páginas calientes (buffer pool) y resultados.

El principio: **cachea lo más cerca posible del consumidor.** Cuanto antes corte una petición, menos recursos consume. La regla 80/20 ([[Desarrollo Profesional/Diseño de Sistemas/Páginas/01 - El Método y Estimaciones|estimaciones]]) justifica cachear el 20% caliente que sirve el 80% del tráfico.

### CDN

Una **CDN** es una red global de servidores *edge* que cachean contenido cerca del usuario. Funciona así: el usuario pide `imagen.jpg`; el DNS lo enruta al edge más cercano; si el edge la tiene (*cache hit*), la sirve al instante; si no (*miss*), la pide al **origen** (tu servidor), la cachea y la sirve.

```
[Usuario Tokio] → [Edge Tokio] ──hit──> sirve (≈5ms)
                       └──miss──> [Origen Europa] (≈250ms, solo la 1ª vez)
```

Qué cachear en CDN: **contenido estático** (imágenes, vídeo, JS, CSS, descargas) y, con cuidado, respuestas dinámicas cacheables. Decisiones clave: el **TTL** (cuánto vive antes de revalidar), la **invalidación** (purgar cuando el contenido cambia — o usar URLs versionadas `app.v2.js` para que un cambio sea una URL nueva), y qué *no* cachear (contenido personalizado o privado). Beneficios: latencia baja (cerca del usuario), descarga del origen, absorción de picos y mitigación de DDoS. Las CDN modernas (Cloudflare, Fastly, CloudFront) también ejecutan lógica en el edge.

### Load Balancing

El **balanceador de carga** reparte el tráfico entre servidores y es la base de la alta disponibilidad. Dos dimensiones:

**Capa (OSI):**
- **L4 (transporte)**: enruta por IP/puerto, sin mirar el contenido. Rapidísimo, pero "ciego" al HTTP.
- **L7 (aplicación)**: entiende HTTP; puede enrutar por path, host o cabeceras (`/api` a un pool, `/static` a otro), terminar TLS, etc. Más flexible, algo más de overhead. Lo habitual hoy.

**Algoritmos de reparto:**
- **Round robin**: uno a cada servidor por turnos. Simple; ignora la carga real.
- **Least connections**: al servidor con menos conexiones activas. Mejor si las peticiones duran distinto.
- **Weighted**: más peso a servidores más potentes.
- **IP hash / sticky sessions**: el mismo cliente siempre al mismo servidor. **Evítalo si puedes** — rompe el principio stateless ([[Desarrollo Profesional/Diseño de Sistemas/Páginas/02 - Escalar de Cero a Millones|página 02]]); úsalo solo si de verdad necesitas afinidad.

El LB también hace **health checks**: sondea los servidores y deja de enviar tráfico a los que no responden, dando failover automático. Un LB es a su vez un punto único, así que en producción se despliega redundante (activo-pasivo o activo-activo).

### Colas de Mensajes en el Diseño

Una cola ([[Desarrollo Profesional/Sistemas Distribuidos/Páginas/02 - Mensajería y Brokers|Mensajería]]) en una arquitectura cumple cuatro funciones de diseño que conviene tener presentes al dibujar:

1. **Desacoplar**: productor y consumidor no se conocen ni dependen de estar vivos a la vez.
2. **Absorber picos (buffering)**: un pico de tráfico se encola y los workers lo drenan a su ritmo, en vez de tumbar el sistema. La cola actúa de amortiguador.
3. **Procesamiento asíncrono**: lo lento (vídeo, email, ML) sale del camino crítico; el usuario recibe respuesta inmediata.
4. **Escalado independiente**: si la cola crece, añades workers sin tocar los productores.

Señal de diseño: cuando veas "esto tarda y no hace falta que el usuario espere" o "esto debe sobrevivir a picos", piensa en una cola.

### Cómo Encaja Todo: Una Petición de Principio a Fin

Sigamos una petición de "ver un perfil con su foto" por la arquitectura completa para ver cada pieza en su sitio:

```
1. [Navegador] ¿tengo la foto en caché local? → sí: no pide. No: ↓
2. [CDN edge] ¿está la foto cacheada cerca? → sí: la sirve (~5ms). No: la trae del origen.
3. [DNS/GeoDNS] → enruta al datacenter más cercano.
4. [Load Balancer L7] → reparte a un web server sano (health-checked).
5. [Web server stateless] → procesa; ¿datos del perfil en caché?
6. [Redis] hit → responde sin tocar la BD. Miss → ↓
7. [BD: réplica de lectura] → consulta el perfil, se cachea en Redis con TTL.
8. (En paralelo, si el perfil tiene un evento que procesar)
   [Web] → publica en [Cola] → [Workers] lo procesan asíncronamente.
```

Cada pieza que estudiaste tiene un lugar y una razón. Diseñar es elegir cuáles necesitas para *tus* requisitos —no meterlas todas— y justificar cada una.

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - La caché tiene **muchos niveles** (navegador → CDN → app/Redis → BD). Principio: cachea **lo más cerca del consumidor** posible; cuanto antes cortas una petición, menos recursos gasta. Regla 80/20 para el qué.
> - Una **CDN** es una caché global de borde para contenido estático: sirve desde el edge cercano, descarga el origen y absorbe picos. Decisiones: TTL, invalidación (o URLs versionadas) y qué no cachear (lo personalizado).
> - El **load balancer** reparte y da alta disponibilidad. **L4** (rápido, ciego) vs **L7** (entiende HTTP, enruta por path/host). Algoritmos: round robin, least connections, weighted, IP hash (evita sticky sessions). Hace health checks y failover; se despliega redundante.
> - Una **cola** en el diseño: desacopla, absorbe picos (buffer), permite asíncrono y escalado independiente. Señal: "tarda y el usuario no debe esperar" o "debe sobrevivir a picos".
> - En una petición real cada pieza tiene su sitio; **diseñar es elegir las que tus requisitos exigen y justificarlas**, no meterlas todas.

### Para llevar a la práctica
- [ ] Traza una petición real de tu sistema por sus niveles de caché. ¿Dónde se podría cortar antes?
- [ ] Revisa tus cabeceras HTTP de estáticos (`Cache-Control`, `ETag`). ¿Aprovechas la caché del navegador y la CDN?
- [ ] Mira el algoritmo de tu load balancer y si usas sticky sessions innecesariamente.
- [ ] Identifica una operación síncrona lenta que mejoraría moviéndose a una cola.

### Recursos
- 📖 *System Design Interview, Vol. 1 y 2* — Alex Xu (building blocks a lo largo de los casos)
- 🌐 github.com/donnemartin/system-design-primer (secciones de caching, CDN, load balancing)
- 🌐 aws.amazon.com/builders-library — artículos sobre caching y balanceo a escala
- 📖 *Designing Data-Intensive Applications* — Martin Kleppmann

---
`#diseño-de-sistemas` `#cache` `#cdn` `#load-balancing` `#colas`
