# 🏛️ Caso de Estudio — Rate Limiter

[[Desarrollo Profesional/Diseño de Sistemas/Diseño de Sistemas|⬅️ Volver a Diseño de Sistemas]] | [[Desarrollo Profesional/Diseño de Sistemas/Páginas/04 - Caché CDN Balanceo y Colas|← 04]] | [[Desarrollo Profesional/Diseño de Sistemas/Páginas/06 - Caso URL Shortener|06 →]]

> [!abstract] Introducción
> Un rate limiter controla cuántas peticiones acepta un sistema por cliente y unidad de tiempo. Protege de abusos, de ataques de denegación de servicio, de clientes con bugs que bombardean tu API, y controla costes. Es un caso de estudio ideal para empezar porque es acotado pero esconde decisiones jugosas: qué algoritmo usar, dónde colocar el limitador y —lo más difícil— cómo hacerlo funcionar de forma consistente cuando tienes muchos servidores. Seguimos el método de la [[Desarrollo Profesional/Diseño de Sistemas/Páginas/01 - El Método y Estimaciones|página 01]].

## ¿De qué vamos a hablar?

El diseño completo de un rate limiter: requisitos, los algoritmos clásicos con sus trade-offs, dónde ubicarlo y cómo hacerlo distribuido.

### Conceptos que vamos a cubrir
- Requisitos y dónde colocar el limitador
- Token Bucket y Leaking Bucket
- Fixed Window, Sliding Window Log y Sliding Window Counter
- El rate limiter distribuido: el problema de la consistencia
- Cabeceras, respuestas y reglas

---

## El Diseño

### 1. Requisitos

**Funcionales**: limitar peticiones por cliente (por IP, por API key, por usuario) según reglas configurables (p. ej. 100 req/min). Devolver `429 Too Many Requests` al exceder.

**No funcionales**: baja latencia (está en el camino crítico de *toda* petición — no puede añadir apenas overhead), alta disponibilidad (si el limitador cae, decide: ¿*fail open* dejando pasar todo, o *fail closed* bloqueando? — normalmente fail open para no tumbar el servicio), preciso, y **funciona en un entorno distribuido** con muchos servidores.

**¿Dónde se coloca?** Tres opciones: en el cliente (poco fiable, se puede saltar), en el servidor (junto a la app), o —lo más común— en un **middleware / API Gateway** antes de llegar a la app. Un gateway (o un service mesh) es el sitio natural: rechaza pronto, antes de gastar recursos.

### 2. Los Algoritmos

**Token Bucket.** Un cubo con capacidad N tokens, que se rellena a un ritmo fijo (R tokens/seg). Cada petición consume un token; si no hay, se rechaza. Permite **ráfagas** (hasta vaciar el cubo) y luego limita al ritmo de relleno. Es el más usado (lo usan AWS, Stripe).

```python
import time

class TokenBucket:
    def __init__(self, capacidad: int, ritmo_relleno: float) -> None:
        self.capacidad = capacidad
        self.ritmo = ritmo_relleno          # tokens por segundo
        self.tokens = capacidad
        self.ultimo = time.monotonic()

    def permitir(self) -> bool:
        ahora = time.monotonic()
        # Rellena según el tiempo transcurrido (sin hilo de fondo)
        self.tokens = min(self.capacidad, self.tokens + (ahora - self.ultimo) * self.ritmo)
        self.ultimo = ahora
        if self.tokens >= 1:
            self.tokens -= 1
            return True
        return False
```

**Leaking Bucket.** Las peticiones entran en una cola (el cubo) y se procesan a ritmo constante (el cubo "gotea"). Suaviza el tráfico a una tasa fija de salida, pero no permite ráfagas y puede añadir latencia (las peticiones esperan en cola). Bueno cuando necesitas un caudal de salida estable.

**Fixed Window Counter.** Cuentas peticiones en ventanas fijas (cada minuto del reloj). Simple y poca memoria, pero tiene un defecto en los bordes: en el cambio de ventana se pueden colar **hasta 2× el límite** (las del final de una ventana + las del principio de la siguiente, en pocos segundos).

```python
def fixed_window(cliente, limite=100, ventana=60):
    clave = f"rate:{cliente}:{int(time.time()) // ventana}"
    n = redis.incr(clave)
    if n == 1:
        redis.expire(clave, ventana)
    return n <= limite
```

**Sliding Window Log.** Guardas el timestamp de cada petición (en un sorted set) y cuentas las que caen en la ventana móvil [ahora - ventana, ahora]. Preciso, sin el problema de los bordes, pero consume **mucha memoria** (un timestamp por petición).

**Sliding Window Counter.** El equilibrio práctico: combina el contador de la ventana actual con una fracción ponderada de la anterior. Aproxima la ventana deslizante con poca memoria. Es lo que usa la mayoría en producción.

```
peticiones ≈ contador_actual + contador_anterior × (solapamiento de la ventana anterior)
```

| Algoritmo | Memoria | Ráfagas | Precisión bordes |
|-----------|---------|---------|------------------|
| Token bucket | O(1) | Sí | Buena |
| Leaking bucket | O(1) cola | No | Suaviza |
| Fixed window | O(1) | — | **Mala** (2×) |
| Sliding log | O(n) | — | Perfecta |
| Sliding counter | O(1) | — | Buena |

### 3. El Rate Limiter Distribuido

Aquí está lo difícil. Si tienes 10 servidores y cada uno guarda su contador en memoria local, un cliente con límite de 100/min podría hacer 100 en *cada* servidor = 1.000/min. El estado del contador **debe ser compartido**. La solución estándar: un almacén central rápido, **Redis** ([[Desarrollo Profesional/Redis/Redis|Redis]]), que todos los servidores consultan.

Pero surge una **condición de carrera**: dos peticiones simultáneas leen el contador (99), ambas creen que pueden pasar, ambas incrementan (101) — se coló una de más. Las soluciones:
- **Operaciones atómicas de Redis**: `INCR` es atómico. Para el sliding window con sorted sets, varias operaciones deben ser atómicas → **script Lua** (`EVAL`), que ejecuta el "leer-decidir-escribir" sin interrupción ([[Desarrollo Profesional/Redis/Páginas/01 - Estructuras y Modelo|Redis 01]]).

```lua
-- Rate limit atómico con sliding window (sorted set de timestamps) en Lua
local clave = KEYS[1]
local ahora = tonumber(ARGV[1])
local ventana = tonumber(ARGV[2])
local limite = tonumber(ARGV[3])
redis.call('ZREMRANGEBYSCORE', clave, 0, ahora - ventana)  -- limpia las viejas
local actual = redis.call('ZCARD', clave)
if actual < limite then
    redis.call('ZADD', clave, ahora, ahora)
    redis.call('EXPIRE', clave, ventana)
    return 1   -- permitido
end
return 0       -- rechazado
```

El compromiso adicional: cada petición hace una llamada de red a Redis (latencia). Para mitigarlo a escala extrema se usan trucos como contadores locales aproximados sincronizados periódicamente, aceptando algo de imprecisión a cambio de latencia. De nuevo, [[Desarrollo Profesional/Sistemas Distribuidos/Páginas/01 - Fundamentos y CAP|el compromiso consistencia vs latencia]].

### 4. Detalles del Diseño

- **Respuesta**: `429 Too Many Requests` con cabeceras informativas: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset` y `Retry-After` (cuántos segundos esperar). Un buen cliente respeta `Retry-After` con backoff ([[Desarrollo Profesional/Patrones de Resiliencia/Páginas/01 - Timeouts Reintentos y Backoff|Resiliencia 01]]).
- **Reglas**: configurables y almacenadas (en caché), por endpoint y por tipo de cliente (un usuario `free` y uno `enterprise` tienen límites distintos).
- **Identificación del cliente**: por API key (lo mejor), por user ID, o por IP (cuidado: NAT/proxies comparten IP).

### 5. Cuellos de Botella y Mejoras

- **Redis como punto único**: replícalo y considera Redis Cluster; decide la política fail-open/closed si Redis no responde.
- **Latencia de la llamada a Redis**: contadores locales aproximados para casos de altísimo volumen.
- **Rate limiting jerárquico**: límites por segundo *y* por minuto *y* por día simultáneos.
- Combínalo con el resto de defensas de [[Desarrollo Profesional/Patrones de Resiliencia/Páginas/02 - Circuit Breaker y Aislamiento|Resiliencia]] (load shedding, bulkheads): el rate limiter es la primera línea, no la única.

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - Un rate limiter protege de abuso/DoS y controla coste. Va idealmente en un **API Gateway/middleware** (rechaza pronto). En el camino crítico → debe ser **rápido**; si cae, normalmente **fail open**.
> - Algoritmos: **token bucket** (permite ráfagas, el más usado), leaking bucket (suaviza, sin ráfagas), **fixed window** (simple pero 2× en los bordes), sliding log (preciso, mucha memoria), **sliding counter** (el equilibrio práctico).
> - El reto real es **distribuido**: contadores locales no sirven (un cliente multiplicaría su límite por el nº de servidores). El estado va en un **Redis compartido**.
> - Cuidado con la **condición de carrera** del contador: usa operaciones atómicas (`INCR`) o **scripts Lua** para el "leer-decidir-escribir" indivisible. Compromiso: latencia de red a Redis vs precisión.
> - Responde `429` con `Retry-After` y cabeceras `X-RateLimit-*`. Reglas configurables por endpoint/tipo de cliente. Combínalo con load shedding y circuit breakers.

### Para llevar a la práctica
- [ ] Implementa el token bucket de arriba y compáralo con el sliding window counter en comportamiento ante ráfagas.
- [ ] Escribe el script Lua del sliding window y razona por qué `INCR` solo no basta para esa variante.
- [ ] Decide para tu API la política fail-open vs fail-closed si el almacén del limitador cae.
- [ ] Añade las cabeceras `X-RateLimit-*` y `Retry-After` a un endpoint y verifica que tu cliente las respeta.

### Recursos
- 📖 *System Design Interview, Vol. 1* — Alex Xu (capítulo 4: Design a Rate Limiter)
- 🌐 stripe.com/blog/rate-limiters — cómo Stripe implementa rate limiting con Redis
- 🌐 cloudflare.com/learning/bots/what-is-rate-limiting — fundamentos
- 📄 Conexión con [[Desarrollo Profesional/Redis/Páginas/02 - Patrones de Uso|Redis: rate limiting]]

---
`#diseño-de-sistemas` `#rate-limiter` `#caso-estudio` `#redis`
