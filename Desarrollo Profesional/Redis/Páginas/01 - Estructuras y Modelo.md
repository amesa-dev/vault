# 🟥 Redis 01 — Estructuras y Modelo

[[Desarrollo Profesional/Redis/Redis|⬅️ Volver a Redis]] | [[Desarrollo Profesional/Redis/Páginas/02 - Patrones de Uso|02 →]]

> [!abstract] Introducción
> Mucha gente piensa en Redis como "un diccionario clave-valor rápido", y se pierde el 80% de su valor. Redis es en realidad un servidor de **estructuras de datos**: además de strings, ofrece listas, sets, hashes, sorted sets, streams y más, cada uno con operaciones atómicas optimizadas. Entender qué estructura encaja con qué problema —y cómo Redis maneja la atomicidad, la persistencia y la expiración— es lo que separa usarlo bien de usarlo como una caché tonta.

## ¿De qué vamos a hablar?

El modelo de datos de Redis y las propiedades del sistema que determinan cómo y para qué usarlo.

### Conceptos que vamos a cubrir
- Por qué es rápido: memoria y un solo hilo
- Las estructuras de datos y para qué sirve cada una
- Atomicidad: comandos, transacciones y scripts Lua
- Persistencia: RDB vs AOF (y qué significa "in-memory")
- Expiración (TTL) y políticas de eviction

---

## El Concepto

### Por Qué es Rápido (y la Implicación del Hilo Único)

Dos decisiones de diseño explican la latencia de microsegundos de Redis:

1. **Todo vive en RAM.** No hay accesos a disco en el camino de lectura/escritura (la persistencia ocurre aparte). RAM es ~100.000× más rápida que un disco.
2. **Los comandos se ejecutan en un solo hilo**, uno tras otro. Esto suena a limitación pero es una ventaja: **cada comando es atómico por construcción** (no hay dos comandos a la vez, no hay condiciones de carrera entre ellos) y no se paga el coste de locks. La red y la persistencia sí usan otros hilos, pero la ejecución de comandos es secuencial.

La implicación práctica del hilo único: **un comando lento bloquea a todos los demás.** Un `KEYS *` en una base de millones de claves, o un `SMEMBERS` sobre un set gigante, congela el servidor entero mientras se ejecuta. Evita comandos O(n) sobre estructuras grandes en producción (usa `SCAN` en vez de `KEYS`).

### Las Estructuras de Datos

El núcleo del valor de Redis. Cada tipo resuelve problemas distintos:

| Estructura | Qué es | Casos de uso típicos |
|------------|--------|----------------------|
| **String** | bytes (texto, número, JSON, binario) | caché de valores, contadores (`INCR`), flags |
| **Hash** | mapa campo→valor dentro de una clave | representar un objeto (usuario con campos), sin serializar todo |
| **List** | lista enlazada ordenada | colas (`LPUSH`/`RPOP`), historiales, feeds recientes |
| **Set** | conjunto sin orden, sin duplicados | etiquetas, miembros únicos, operaciones de conjuntos (intersección de seguidores) |
| **Sorted Set (ZSet)** | set con un *score* numérico que ordena | **leaderboards**, colas de prioridad, rate limiting por ventana, índices ordenados |
| **Stream** | log de eventos append-only (como Kafka mini) | colas de eventos con consumer groups, event sourcing ligero |
| **Bitmap / HyperLogLog** | bits / conteo aproximado de cardinalidad | usuarios activos diarios, conteos únicos con memoria mínima |

```python
import redis
r = redis.Redis()

# String como contador atómico
r.incr("visitas:pagina:home")                 # atómico, sin race condition

# Hash como objeto
r.hset("usuario:42", mapping={"nombre": "Ana", "plan": "pro"})
r.hget("usuario:42", "plan")                   # b"pro" — sin leer el objeto entero

# Sorted set como leaderboard
r.zadd("ranking", {"ana": 1500, "luis": 1800, "eva": 1650})
r.zrevrange("ranking", 0, 2, withscores=True)  # top 3 ordenado por score
```

Elegir la estructura correcta no es un detalle: usar un sorted set para un leaderboard te da el top-N en O(log n) en vez de ordenar a mano en la aplicación.

### Atomicidad: Comandos, Transacciones y Lua

- **Cada comando** es atómico (hilo único). `INCR`, `LPUSH`, etc. nunca se entrelazan.
- **Transacciones (`MULTI`/`EXEC`)**: agrupan varios comandos para que se ejecuten juntos sin que otro cliente se cuele en medio. Pero **no son como las transacciones SQL**: no hay rollback si un comando falla, y no permiten lógica condicional (no puedes "leer X y según su valor hacer Y").
- **Scripts Lua (`EVAL`)**: para operaciones atómicas con lógica (leer, decidir, escribir, todo de una). El script se ejecuta entero sin interrupción. Es la forma correcta de implementar un lock seguro o un rate limiter, donde necesitas atomicidad de varias operaciones con condicionales (lo verás en la [página 02](02%20-%20Patrones%20de%20Uso)).

```python
# WATCH para optimistic locking: aborta si la clave cambió desde que la leíste
with r.pipeline() as pipe:
    while True:
        try:
            pipe.watch("saldo")
            saldo = int(pipe.get("saldo"))
            pipe.multi()
            pipe.set("saldo", saldo - 100)
            pipe.execute()    # falla si "saldo" cambió tras el WATCH → reintenta
            break
        except redis.WatchError:
            continue
```

### Persistencia: ¿"In-Memory" Significa Volátil?

Redis vive en RAM, pero **puede persistir a disco** para sobrevivir reinicios. Dos mecanismos (combinables):

- **RDB (snapshots)**: vuelca toda la base a un fichero cada cierto tiempo. Compacto y rápido de cargar, pero **pierdes lo escrito entre snapshots** si hay un crash. Bueno para backups.
- **AOF (Append-Only File)**: registra cada operación de escritura en un log. Más duradero (puedes configurar `fsync` por segundo o por escritura), pero el fichero crece y la recuperación es más lenta. 

La elección depende de cuánto puedes permitirte perder: caché pura → quizá sin persistencia; datos que importan → AOF con `everysec`. Aun con persistencia, **trata Redis como una caché o un almacén complementario**, no como la fuente de verdad de datos críticos, salvo que lo configures explícitamente para durabilidad fuerte (y aceptes el coste).

### Expiración y Eviction

Como la RAM es finita, Redis tiene dos mecanismos para no llenarse:

- **TTL (Time To Live)**: pones caducidad a una clave con `EXPIRE` o `SET ... EX`. Tras el tiempo, Redis la borra automáticamente. Es la base del caching: los datos cacheados expiran y se refrescan.

```python
r.set("sesion:abc", datos, ex=3600)   # expira en 1 hora
r.ttl("sesion:abc")                    # segundos restantes
```

- **Políticas de eviction (`maxmemory-policy`)**: cuando Redis alcanza el límite de memoria, decide qué desalojar. Las más usadas:
  - `allkeys-lru`: desaloja las claves menos usadas recientemente (LRU). **El defecto sensato para una caché.**
  - `allkeys-lfu`: las menos usadas en frecuencia (LFU). Mejor si hay claves "calientes" estables.
  - `volatile-ttl`: desaloja primero las que expiran antes.
  - `noeviction`: rechaza escrituras nuevas cuando se llena (el defecto). Peligroso para una caché — empezarás a recibir errores.

Configurar bien `maxmemory` y la política de eviction es lo que evita que Redis se quede sin memoria y empiece a fallar o a usar swap (que destruye su rendimiento).

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - Redis es rápido por dos razones: vive en **RAM** y ejecuta comandos en **un solo hilo** (cada comando es atómico, sin locks). Implicación: **un comando O(n) lento bloquea todo** — evita `KEYS`, usa `SCAN`.
> - No es solo clave-valor: ofrece **strings, hashes, lists, sets, sorted sets, streams, bitmaps/HLL**. Elegir la estructura correcta (ZSet para leaderboards, sets para unicidad) cambia la complejidad de tu solución.
> - Atomicidad: cada comando lo es; `MULTI`/`EXEC` agrupa (sin rollback ni lógica); **Lua (`EVAL`)** para operaciones atómicas con condicionales; `WATCH` para optimistic locking.
> - Persistencia opcional: **RDB** (snapshots, puedes perder lo último) y **AOF** (log de escrituras, más duradero). Aun así, trátalo como caché/complemento, no como fuente de verdad por defecto.
> - **TTL** caduca claves (base del caching); las **políticas de eviction** (`allkeys-lru` para caché) deciden qué desalojar al llenarse. Configura `maxmemory` o Redis fallará o usará swap.

### Para llevar a la práctica
- [ ] Para un caso tuyo que resuelvas con strings + JSON, mira si un hash o un sorted set encaja mejor.
- [ ] Comprueba si en algún sitio usas `KEYS *` en producción. Cámbialo a `SCAN`.
- [ ] Revisa la `maxmemory-policy` de tu Redis. Si es una caché y está en `noeviction`, vas a recibir errores al llenarse.
- [ ] Asegúrate de que tus claves de caché tienen TTL. Sin TTL, crecen para siempre.

### Recursos
- 🌐 redis.io/docs/latest/develop/data-types — las estructuras de datos explicadas
- 🌐 redis.io/docs/latest/operate/oss_and_stack/management/persistence — RDB vs AOF
- 📖 *Redis in Action* — Josiah Carlson
- 🌐 redis.io/docs/latest/develop/interact/transactions — transacciones y WATCH

---
`#redis` `#estructuras-datos` `#persistencia` `#ttl`
