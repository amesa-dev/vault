# 🛡️ Resiliencia 01 — Timeouts, Reintentos y Backoff

[[Desarrollo Profesional/Patrones de Resiliencia/Patrones de Resiliencia|⬅️ Volver a Patrones de Resiliencia]] | [[Desarrollo Profesional/Patrones de Resiliencia/Páginas/02 - Circuit Breaker y Aislamiento|02 →]]

> [!abstract] Introducción
> Los tres patrones más básicos —y más mal implementados— de la resiliencia. Un timeout demasiado largo agota tus recursos esperando a un servicio muerto. Un reintento ingenuo convierte un pico transitorio en una tormenta que tumba la dependencia. Y un backoff sin jitter sincroniza a todos tus clientes para que reintenten exactamente a la vez. Esta página explica cómo hacerlo bien.

## ¿De qué vamos a hablar?

El primer perímetro de defensa frente a fallos transitorios: cortar a tiempo, reintentar con cabeza y no amplificar el problema.

### Conceptos que vamos a cubrir
- Timeouts: por qué "sin timeout" es el bug por defecto
- Reintentos: cuándo sí y cuándo nunca
- Backoff exponencial y por qué necesita jitter
- El problema del retry storm y la amplificación de carga
- Deadlines propagados a través de llamadas encadenadas

---

## El Concepto

### Timeouts — El Bug por Defecto es No Tenerlo

Casi toda librería de cliente HTTP/DB tiene un timeout por defecto demasiado alto o **infinito**. El resultado: cuando una dependencia se cuelga (no rechaza, simplemente no responde), tus hilos/conexiones se quedan bloqueados esperando. Con suficiente tráfico, **agotas tu pool de conexiones o hilos** y tu servicio cae aunque tu código esté perfecto. Esto es el origen de la mayoría de las caídas en cascada.

```python
import httpx

# ❌ Sin timeout — un servicio colgado bloquea tu hilo indefinidamente
respuesta = httpx.get("https://api.externa/datos")

# ✅ Timeouts explícitos y granulares
timeout = httpx.Timeout(
    connect=2.0,   # establecer la conexión TCP
    read=5.0,      # esperar la respuesta una vez conectado
    write=5.0,     # enviar el cuerpo de la petición
    pool=1.0,      # esperar una conexión libre del pool
)
respuesta = httpx.get("https://api.externa/datos", timeout=timeout)
```

Regla práctica: **el timeout debe basarse en el percentil alto (p99) de latencia normal de la dependencia, no en el peor caso imaginable.** Si el p99 es 200 ms, un timeout de 1 s es razonable; uno de 30 s significa que esperas 30 s a algo que normalmente tarda 0,2 s — eso no es paciencia, es un recurso secuestrado.

### Reintentos — Solo para lo Transitorio

Reintentar tiene sentido **solo** ante fallos transitorios: un timeout, un `503 Service Unavailable`, un `429 Too Many Requests`, una conexión reseteada. **Nunca** reintentes errores deterministas: un `400 Bad Request` o `404 Not Found` fallará igual las 5 veces — solo gastas recursos y latencia.

```python
import time
import random
import httpx

CODIGOS_REINTENTABLES = {429, 502, 503, 504}

def es_reintentable(exc_o_resp) -> bool:
    if isinstance(exc_o_resp, (httpx.TimeoutException, httpx.ConnectError)):
        return True
    if isinstance(exc_o_resp, httpx.Response):
        return exc_o_resp.status_code in CODIGOS_REINTENTABLES
    return False
```

> [!warning] Reintentos + no idempotencia = corrupción
> Reintentar un `POST /cobrar` que ya tuvo efecto (pero cuya respuesta se perdió) **cobra dos veces**. Solo reintenta operaciones idempotentes, o usa una `Idempotency-Key` (ver [[Desarrollo Profesional/Sistemas Distribuidos/Páginas/03 - Transacciones Distribuidas|Transacciones Distribuidas]]).

### Backoff Exponencial con Jitter

Reintentar inmediatamente martillea un servicio que probablemente ya está sobrecargado. El **backoff exponencial** espera cada vez más entre intentos: 1 s, 2 s, 4 s, 8 s... Pero hay un problema sutil: si 10.000 clientes fallan a la vez y todos hacen backoff exponencial puro, **todos reintentan exactamente en el mismo instante** (1 s, luego 2 s...), creando picos sincronizados que vuelven a tumbar el servicio. La solución es añadir **jitter** (aleatoriedad):

```python
def backoff_con_jitter(intento: int, base: float = 1.0, tope: float = 30.0) -> float:
    """Full jitter (recomendado por AWS): espera aleatoria en [0, delay_exponencial]."""
    delay_exponencial = min(tope, base * (2 ** intento))
    return random.uniform(0, delay_exponencial)

def llamar_con_reintentos(fn, max_intentos: int = 5):
    ultimo_error = None
    for intento in range(max_intentos):
        try:
            resp = fn()
            if not es_reintentable(resp):
                return resp
            ultimo_error = resp
        except Exception as exc:
            if not es_reintentable(exc):
                raise
            ultimo_error = exc

        if intento < max_intentos - 1:
            espera = backoff_con_jitter(intento)
            time.sleep(espera)   # en async: await asyncio.sleep(espera)
    raise RuntimeError(f"Agotados {max_intentos} intentos; último error: {ultimo_error}")
```

Las variantes de jitter (del célebre artículo de AWS *"Exponential Backoff and Jitter"*):
- **Full jitter**: `random(0, exp)` — la mejor opción general, dispersa al máximo.
- **Equal jitter**: `exp/2 + random(0, exp/2)` — garantiza una espera mínima.
- **Decorrelated jitter**: usa el delay anterior para calcular el siguiente; bueno para evitar agrupaciones.

### El Retry Storm y la Amplificación de Carga

El peligro más insidioso: los reintentos **multiplican la carga justo cuando el sistema menos lo aguanta**. Si cada cliente reintenta 3 veces, una dependencia degradada recibe de golpe 3× el tráfico, lo que la degrada más, lo que provoca más fallos, lo que provoca más reintentos... una **espiral de muerte**.

Defensas contra el retry storm:
- **Limita el número de reintentos** (3-5, no infinito).
- **Retry budget**: permite reintentos solo si suponen, p. ej., menos del 10% del tráfico total. Si se supera, deja de reintentar (lo hace el service mesh Envoy/Istio).
- **No reintentes en cada capa**: si A→B→C y los tres reintentan 3 veces, C recibe 3×3×3 = 27× la carga. Reintenta en **una sola capa** (normalmente la más cercana al fallo).
- Combínalo con un **circuit breaker** ([siguiente página](02%20-%20Circuit%20Breaker%20y%20Aislamiento)) que corte los reintentos cuando la dependencia está claramente caída.

### Deadlines Propagados

Mejor que un timeout local aislado es un **deadline** que viaja por toda la cadena de llamadas. Si el usuario tiene 3 s de paciencia, ese presupuesto se reparte: el gateway pasa "te quedan 2,8 s" al servicio A, que pasa "te quedan 2,5 s" a B. Cuando el deadline expira, **toda la cadena aborta** en lugar de que cada salto siga trabajando sobre una petición que el usuario ya abandonó. gRPC lo hace nativo con `deadlines`; en HTTP se propaga con una cabecera o un campo de contexto.

```python
import time

class Deadline:
    """Presupuesto de tiempo absoluto que se propaga por la cadena de llamadas."""
    def __init__(self, segundos: float) -> None:
        self.expira_en = time.monotonic() + segundos

    def restante(self) -> float:
        return max(0.0, self.expira_en - time.monotonic())

    def expirado(self) -> bool:
        return self.restante() <= 0

# Cada llamada usa lo que queda del presupuesto, no un timeout fijo
def llamar_servicio(deadline: Deadline):
    if deadline.expirado():
        raise TimeoutError("Deadline agotado antes de llamar")
    return httpx.get("https://servicio/b", timeout=deadline.restante())
```

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - **Sin timeout es el bug por defecto**: una dependencia colgada agota tus hilos/conexiones y te tumba. Fija timeouts basados en el p99 real, no en el peor caso imaginable.
> - **Reintenta solo fallos transitorios** (timeout, 429, 503) y solo operaciones idempotentes. Nunca reintentes un 400/404 ni un cobro sin clave de idempotencia.
> - El **backoff exponencial necesita jitter**; sin aleatoriedad, todos los clientes reintentan a la vez y resincronizan el pico. *Full jitter* es la opción por defecto.
> - Cuidado con el **retry storm**: limita reintentos, usa retry budgets, no reintentes en cada capa (la carga se multiplica) y combínalo con un circuit breaker.
> - Un **deadline propagado** por toda la cadena evita que servicios sigan trabajando sobre una petición ya abandonada.

### Para llevar a la práctica
- [ ] Audita los clientes HTTP/DB de tu servicio: ¿cuántos tienen timeout explícito? Pon timeouts a todos.
- [ ] Revisa tu lógica de reintentos: ¿añade jitter? ¿reintenta cosas no idempotentes? ¿reintenta en varias capas a la vez?
- [ ] Calcula la amplificación: si cada capa reintenta N veces y hay K capas, tu dependencia recibe N^K. ¿Es aceptable?

### Recursos
- 🌐 aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter — artículo de referencia
- 🌐 sre.google/sre-book/handling-overload — capítulo de sobrecarga del SRE Book de Google
- 📖 *Release It!* — Michael Nygard (timeouts y fallos en cascada)
- 🌐 grpc.io/docs/guides/deadlines — deadlines propagados en gRPC

---
`#resiliencia` `#timeouts` `#reintentos` `#backoff`
