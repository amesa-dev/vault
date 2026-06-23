# 🟥 Redis 02 — Patrones de Uso

[[Desarrollo Profesional/Redis/Redis|⬅️ Volver a Redis]] | [[Desarrollo Profesional/Redis/Páginas/01 - Estructuras y Modelo|← 01]]

> [!abstract] Introducción
> Las estructuras de Redis cobran sentido en los patrones donde se aplican. Esta página recoge los cinco usos que aparecen una y otra vez en sistemas a escala: caching (con sus estrategias e invalidación, que es "uno de los dos problemas difíciles de la informática"), locks distribuidos (con sus trampas de seguridad), pub/sub para notificaciones, streams para colas de eventos, y rate limiting. Cada patrón viene con su implementación y, más importante, con sus garantías y peligros.

## ¿De qué vamos a hablar?

Los patrones recurrentes que resuelven Redis, con sus implementaciones correctas y sus trampas.

### Conceptos que vamos a cubrir
- Caching: cache-aside, write-through, write-behind
- Invalidación de caché y el problema de la coherencia
- Locks distribuidos: cómo hacerlos seguros (y por qué es difícil)
- Pub/Sub vs Streams
- Rate limiting con Redis

---

## El Concepto

### Caching: las Estrategias

El uso estrella. La idea: guardar en Redis (rápido) el resultado de operaciones costosas (consultas a BD, llamadas a APIs) para no repetirlas. Hay tres estrategias según cómo se escribe:

**Cache-aside (lazy loading)** — la más común. La aplicación gestiona la caché: mira primero ahí, y si no está (cache miss) va a la BD y guarda el resultado.

```python
def obtener_usuario(user_id: int):
    clave = f"usuario:{user_id}"
    cacheado = r.get(clave)
    if cacheado is not None:
        return json.loads(cacheado)          # cache hit

    usuario = db.query_usuario(user_id)       # cache miss → fuente de verdad
    r.set(clave, json.dumps(usuario), ex=300) # guardar con TTL de 5 min
    return usuario
```
*Pro*: solo cachea lo que se pide; resiliente (si Redis cae, vas a la BD). *Contra*: el primer acceso siempre es lento (miss); riesgo de datos obsoletos hasta que expire el TTL.

**Write-through** — cada escritura va a la caché *y* a la BD a la vez (síncrono). La caché siempre está fresca, pero cada escritura es más lenta.

**Write-behind (write-back)** — escribes en la caché y la BD se actualiza de forma asíncrona después. Escrituras rapidísimas, pero riesgo de perder datos si Redis cae antes de persistir.

> [!warning] El problema difícil: la invalidación
> "Solo hay dos problemas difíciles en informática: la invalidación de caché y nombrar cosas" (Phil Karlton). El reto: cuando el dato cambia en la BD, ¿cómo evitas servir la versión vieja de la caché? Estrategias: **TTL corto** (aceptas estar desactualizado un rato — simple y suficiente a menudo), **invalidación explícita** (al actualizar la BD, borras la clave de caché — más fresco pero hay que acordarse en cada escritura), o **versionado de claves**. No hay solución perfecta; eliges el equilibrio entre frescura y complejidad.

Dos patologías de caché a conocer:
- **Cache stampede / thundering herd**: una clave popular expira y miles de peticiones simultáneas van todas a la BD a regenerarla a la vez. Mitigación: un lock para que solo una regenere, o recálculo probabilístico antes de expirar.
- **Cache penetration**: peticiones de claves que no existen ni en BD pasan siempre de largo. Mitigación: cachear también el "no existe" (con TTL corto) o un Bloom filter.

### Locks Distribuidos

Cuando varios procesos/instancias compiten por un recurso (procesar un job una sola vez, evitar doble cobro), necesitas un lock que funcione *entre máquinas*. Redis se usa mucho para esto, pero es **fácil hacerlo mal**:

```python
import uuid

# ✅ Lock con dueño único y TTL (evita locks colgados si el dueño muere)
def adquirir_lock(recurso: str, ttl_ms: int = 10000) -> str | None:
    token = str(uuid.uuid4())               # identifica a ESTE dueño
    # SET NX: solo si no existe. PX: expira solo (nadie queda bloqueado para siempre)
    ok = r.set(f"lock:{recurso}", token, nx=True, px=ttl_ms)
    return token if ok else None

# Liberar SOLO si sigo siendo el dueño (atómico con Lua)
LIBERAR = """
if redis.call("get", KEYS[1]) == ARGV[1] then
    return redis.call("del", KEYS[1])
else
    return 0
end
"""
def liberar_lock(recurso: str, token: str) -> None:
    r.eval(LIBERAR, 1, f"lock:{recurso}", token)
```

Las dos trampas que el código de arriba evita:
- **Sin TTL**: si el proceso que tiene el lock muere, el lock queda para siempre y nadie más puede trabajar. El `PX` garantiza que se libere solo.
- **Liberar el lock de otro**: si el proceso A tarda más que el TTL, el lock expira, B lo adquiere, y entonces A termina y hace `DEL` — borrando el lock de **B**. Por eso se libera solo si el token coincide, y de forma atómica con Lua.

> [!warning] Los locks de Redis no son perfectos
> Incluso bien implementado, un lock sobre un solo Redis no garantiza exclusión mutua absoluta en todos los escenarios (un GC pause largo puede hacer que A crea que tiene el lock cuando ya expiró). El algoritmo **Redlock** (varios nodos Redis) intenta robustecerlo, pero es debatido. La regla: para correctitud crítica (dinero), **no confíes solo en el lock** — combínalo con idempotencia y comprobaciones en la BD (ver [[Desarrollo Profesional/Sistemas Distribuidos/Páginas/03 - Transacciones Distribuidas|Transacciones Distribuidas]]). Un lock de Redis es una *optimización* para reducir contención, no una garantía dura.

### Pub/Sub vs Streams

Dos formas de mensajería en Redis, con garantías muy distintas:

- **Pub/Sub**: publicas en un canal y los suscriptores conectados *en ese momento* lo reciben. **Fire-and-forget**: si un suscriptor está desconectado, **pierde el mensaje** (no se almacena). Útil para notificaciones efímeras (invalidar caché en todas las instancias, eventos en tiempo real que no importa perder).
- **Streams**: un log append-only persistente (como un Kafka ligero). Los mensajes se guardan, hay **consumer groups** con confirmación (ACK) y reintento de no-confirmados, y puedes releer desde cualquier punto. Para colas de trabajo fiables donde no puedes perder mensajes, usa Streams, no Pub/Sub.

```python
# Stream como cola de trabajo con consumer group (at-least-once, con ACK)
r.xadd("tareas", {"tipo": "email", "destino": "ana@..."})       # productor

r.xgroup_create("tareas", "workers", id="0", mkstream=True)
mensajes = r.xreadgroup("workers", "worker-1", {"tareas": ">"}, count=1)
# ...procesar...
r.xack("tareas", "workers", mensaje_id)   # confirmar; sin ACK, se reentrega
```

La elección Pub/Sub vs Streams es la misma distinción cola-vs-log de [[Desarrollo Profesional/Sistemas Distribuidos/Páginas/02 - Mensajería y Brokers|Sistemas Distribuidos 02]], a escala de Redis.

### Rate Limiting

Limitar peticiones por cliente. Redis es ideal por su atomicidad y TTL. El enfoque más simple, **fixed window**:

```python
def permitir(cliente: str, limite: int = 100, ventana_s: int = 60) -> bool:
    clave = f"rate:{cliente}:{int(time.time()) // ventana_s}"  # clave por ventana
    actual = r.incr(clave)        # atómico
    if actual == 1:
        r.expire(clave, ventana_s)  # la primera vez, fija el TTL de la ventana
    return actual <= limite
```

El fixed window tiene un defecto en los bordes (se pueden colar 2× peticiones en el cambio de ventana). Para producción se usa **sliding window** (con sorted sets, guardando timestamps) o **token bucket**. Los algoritmos en detalle, en [[Desarrollo Profesional/Diseño de Sistemas/Páginas/05 - Caso Rate Limiter|el caso de Diseño de Sistemas]].

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - **Caching**: *cache-aside* (la app mira la caché, y en miss va a la BD y guarda — lo más común y resiliente), *write-through* (escribe en ambos, siempre fresco), *write-behind* (BD asíncrona, rápido pero arriesgado).
> - La **invalidación** es el problema difícil: TTL corto (simple, aceptas obsolescencia), invalidación explícita (fresco, hay que acordarse) o versionado. Cuidado con **cache stampede** (lock al regenerar) y **penetration** (cachear el "no existe").
> - **Locks distribuidos**: `SET NX PX` con token único, y libera solo si el token coincide (atómico con Lua). Sin TTL → locks colgados; sin token → borras el lock de otro. Aun así **no son garantía dura**: combínalos con idempotencia para correctitud crítica.
> - **Pub/Sub** (fire-and-forget, pierde mensajes si no estás conectado) vs **Streams** (log persistente con consumer groups y ACK, para colas fiables). Misma distinción cola-vs-log.
> - **Rate limiting**: fixed window con `INCR`+`EXPIRE` es simple; sliding window/token bucket para producción.

### Para llevar a la práctica
- [ ] Identifica una consulta cara y repetida en tu sistema y cachéala con cache-aside + TTL.
- [ ] Si usas locks de Redis, revisa que tengan TTL y que liberes solo tu propio token con Lua.
- [ ] Comprueba si usas Pub/Sub para algo que no puede perder mensajes. Si sí, migra a Streams.
- [ ] Añade un rate limiter básico a un endpoint público para protegerlo de abuso.

### Recursos
- 🌐 redis.io/docs/latest/develop/use/patterns — patrones oficiales
- 🌐 redis.io/docs/latest/develop/use/patterns/distributed-locks — locks y Redlock (con sus matices)
- 📄 martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html — la crítica clásica a Redlock
- 📖 *Redis in Action* — Josiah Carlson (capítulos de caching y colas)

---
`#redis` `#caching` `#locks-distribuidos` `#pubsub` `#streams`
