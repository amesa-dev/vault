# 🛡️ Resiliencia 02 — Circuit Breaker y Aislamiento

[[Desarrollo Profesional/Patrones de Resiliencia/Patrones de Resiliencia|⬅️ Volver a Patrones de Resiliencia]] | [[Desarrollo Profesional/Patrones de Resiliencia/Páginas/01 - Timeouts Reintentos y Backoff|← 01]]

> [!abstract] Introducción
> Los timeouts y reintentos manejan fallos puntuales. Pero cuando una dependencia está claramente caída, seguir llamándola (y reintentando) solo desperdicia recursos y propaga el fallo. El circuit breaker corta el flujo cuando detecta que algo está roto, dándole tiempo a recuperarse. Junto con el bulkhead (aislar recursos para que un fallo no contamine al resto), el rate limiting y la degradación elegante, forman la segunda línea de defensa: contener el fallo para que no se convierta en una caída total.

## ¿De qué vamos a hablar?

Los patrones que **contienen** un fallo ya presente para que no se propague en cascada por todo el sistema.

### Conceptos que vamos a cubrir
- Circuit Breaker: los tres estados y cómo se mueven
- Bulkhead: aislar recursos para acotar el daño
- Rate limiting y load shedding: protegerse del exceso de carga
- Degradación elegante y fallbacks
- Cómo se combinan todos en una llamada real

---

## El Concepto

### Circuit Breaker — El Fusible del Software

La metáfora es el fusible eléctrico: cuando la corriente es peligrosa, se "funde" y corta el circuito para proteger la instalación. Un circuit breaker monitoriza las llamadas a una dependencia y, si la tasa de fallos supera un umbral, **deja de llamarla** y devuelve un error inmediato (o un fallback) sin esperar al timeout. Tiene tres estados:

```
        fallos < umbral
   ┌────────────────────────┐
   │                        │
   ▼                        │
[CLOSED] ──fallos≥umbral──→ [OPEN] ──pasa el tiempo de espera──→ [HALF-OPEN]
   ▲                          ▲                                      │
   │                          │ falla la prueba                      │ pasa la prueba
   │                          └──────────────────────────────────────┤
   └─────────────────── éxito de la prueba ────────────────────────┘
```

- **CLOSED** (cerrado, normal): las llamadas pasan. Se cuentan los fallos. Si superan el umbral en una ventana, se abre.
- **OPEN** (abierto): las llamadas **fallan inmediatamente** sin tocar la dependencia. Esto le da respiro para recuperarse y libera tus recursos. Tras un tiempo de espera, pasa a half-open.
- **HALF-OPEN** (semiabierto): deja pasar **unas pocas llamadas de prueba**. Si tienen éxito, asume que la dependencia se recuperó y vuelve a CLOSED. Si fallan, vuelve a OPEN y espera más.

```python
import time
from enum import Enum

class Estado(Enum):
    CLOSED = "closed"
    OPEN = "open"
    HALF_OPEN = "half_open"

class CircuitBreaker:
    def __init__(self, umbral_fallos: int = 5, espera_reset: float = 30.0,
                 pruebas_half_open: int = 1) -> None:
        self._umbral = umbral_fallos
        self._espera = espera_reset
        self._max_pruebas = pruebas_half_open
        self._estado = Estado.CLOSED
        self._fallos = 0
        self._abierto_en = 0.0
        self._pruebas_en_curso = 0

    def llamar(self, fn, *args, **kwargs):
        if self._estado == Estado.OPEN:
            if time.monotonic() - self._abierto_en >= self._espera:
                self._estado = Estado.HALF_OPEN
                self._pruebas_en_curso = 0
            else:
                raise CircuitoAbierto("Dependencia caída; circuito abierto")

        if self._estado == Estado.HALF_OPEN and self._pruebas_en_curso >= self._max_pruebas:
            raise CircuitoAbierto("Probando recuperación; espera")

        try:
            if self._estado == Estado.HALF_OPEN:
                self._pruebas_en_curso += 1
            resultado = fn(*args, **kwargs)
        except Exception:
            self._registrar_fallo()
            raise
        else:
            self._registrar_exito()
            return resultado

    def _registrar_fallo(self) -> None:
        self._fallos += 1
        if self._estado == Estado.HALF_OPEN or self._fallos >= self._umbral:
            self._estado = Estado.OPEN
            self._abierto_en = time.monotonic()

    def _registrar_exito(self) -> None:
        self._fallos = 0
        self._estado = Estado.CLOSED

class CircuitoAbierto(Exception): ...
```

La diferencia clave con un simple reintento: el circuit breaker tiene **memoria**. Sabe que la dependencia lleva fallando y deja de molestarla, en vez de que cada petición descubra el fallo de cero (y pague el timeout). En producción se usan librerías probadas: `pybreaker` en Python, Resilience4j en Java, o el circuit breaking del service mesh (Istio/Envoy).

### Bulkhead — Compartimentos Estancos

El nombre viene de los mamparos de un barco: si el casco se perfora, solo se inunda un compartimento, no todo el barco. En software, el bulkhead **aísla los recursos** (pools de conexiones, hilos, semáforos) por dependencia, de forma que si una se satura, **no consume los recursos de las demás**.

```python
import asyncio

class Bulkhead:
    """Limita la concurrencia hacia una dependencia con un semáforo.
    Si la dependencia se ralentiza, las llamadas de más se rechazan rápido
    en lugar de acumularse y agotar todos los recursos del proceso."""
    def __init__(self, max_concurrentes: int) -> None:
        self._sem = asyncio.Semaphore(max_concurrentes)

    async def ejecutar(self, coro):
        if self._sem.locked():
            # Sin huecos libres: falla rápido en vez de encolar indefinidamente
            raise BulkheadLleno("Demasiadas llamadas concurrentes a la dependencia")
        async with self._sem:
            return await coro

class BulkheadLleno(Exception): ...

# Cada dependencia tiene su propio bulkhead → un fallo en "pagos"
# no agota las conexiones disponibles para "catálogo"
bulkhead_pagos = Bulkhead(max_concurrentes=10)
bulkhead_catalogo = Bulkhead(max_concurrentes=50)
```

El caso clásico que evita: un único pool de 100 conexiones compartido. Si la dependencia lenta acapara las 100 esperando, **ninguna otra parte del sistema puede trabajar**, aunque sus dependencias estén perfectas. Con bulkheads separados, el daño queda acotado.

### Rate Limiting y Load Shedding

Resiliencia también es **protegerte de demasiada carga**, venga de fuera (clientes abusivos) o de dentro:

- **Rate limiting**: limitas cuántas peticiones aceptas por cliente/ventana. Los algoritmos clásicos —token bucket, sliding window— se detallan en [[Desarrollo Profesional/Diseño de Sistemas/Páginas/05 - Caso Rate Limiter|el caso de Diseño de Sistemas]].
- **Load shedding**: cuando el sistema está al límite (CPU/colas saturadas), **rechazas activamente** las peticiones de menor prioridad con un `503` rápido en vez de aceptarlas todas y colapsar. Es preferible servir bien al 90% que servir mal al 100%.

> [!tip] Falla rápido, no lento
> El principio que une circuit breaker, bulkhead y load shedding: cuando no puedes atender bien una petición, **recházala rápido**. Un `503` en 1 ms libera recursos; una petición aceptada que tarda 30 s y luego falla los secuestra. *Fail fast* protege al sistema entero.

### Degradación Elegante y Fallbacks

La resiliencia no es solo no caerse — es **dar el mejor servicio posible con lo que funciona**. Cuando una dependencia no responsable falla, en vez de propagar el error, ofrece una alternativa degradada:

```python
def obtener_recomendaciones(usuario_id):
    try:
        return breaker.llamar(servicio_ml.recomendar, usuario_id)
    except (CircuitoAbierto, TimeoutError):
        # Fallback: lo más vendido (cacheado), no recomendaciones personalizadas.
        # El usuario ve algo útil en lugar de una página rota.
        return cache.get("top_ventas_global", default=[])
```

Ejemplos de degradación: mostrar datos cacheados aunque estén un poco viejos, ocultar el módulo de recomendaciones si su servicio cae (pero dejar comprar), servir resultados sin personalizar. La clave es decidir **qué funcionalidad es esencial y cuál es prescindible** bajo estrés.

### Todo Junto

En una llamada real, los patrones se anidan en capas, de fuera hacia dentro:

```
Rate limiter (¿acepto esta petición?)
  └─ Bulkhead (¿hay recursos para esta dependencia?)
       └─ Circuit breaker (¿está la dependencia operativa?)
            └─ Reintentos + backoff con jitter (¿fallo transitorio?)
                 └─ Timeout (¿corto si tarda demasiado?)
                      └─ La llamada real
       (si algo de esto falla) → Fallback / degradación elegante
```

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - El **Circuit Breaker** es un fusible con memoria: tras N fallos se **abre** y falla rápido sin tocar la dependencia caída; tras un tiempo prueba (**half-open**) y se cierra si se recuperó. Evita pagar timeouts repetidos y da respiro a la dependencia.
> - El **Bulkhead** aísla recursos (pools, semáforos) por dependencia, para que una saturada no agote los recursos de las demás. Compartimentos estancos.
> - **Rate limiting** (limitar peticiones) y **load shedding** (rechazar las de baja prioridad bajo estrés) te protegen del exceso de carga. Es mejor servir bien al 90% que mal al 100%.
> - Principio rector: **falla rápido**. Un rechazo en 1 ms libera recursos; una petición aceptada que falla en 30 s los secuestra.
> - La **degradación elegante** ofrece una alternativa (caché, datos sin personalizar) en vez de propagar el error. Decide de antemano qué es esencial y qué es prescindible.
> - En la práctica se anidan: rate limiter → bulkhead → circuit breaker → reintentos+backoff → timeout → llamada, con fallback si algo falla.

### Para llevar a la práctica
- [ ] Identifica tu dependencia externa más crítica y envuélvela en un circuit breaker (usa `pybreaker`, no lo escribas a mano en producción).
- [ ] Revisa si compartes un único pool de conexiones entre dependencias distintas. Si sí, sepáralo en bulkheads.
- [ ] Para cada dependencia, define el fallback: ¿qué muestras si cae? ¿caché, valor por defecto, módulo oculto?

### Recursos
- 📖 *Release It!* — Michael Nygard (el origen de Circuit Breaker y Bulkhead como patrones)
- 🌐 martinfowler.com/bliki/CircuitBreaker.html
- 🌐 resilience4j.readme.io — la librería de referencia (conceptos aplicables a cualquier lenguaje)
- 🌐 sre.google/sre-book/handling-overload — load shedding y graceful degradation

---
`#resiliencia` `#circuit-breaker` `#bulkhead` `#rate-limiting`
