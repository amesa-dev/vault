# 🐍 11 — Concurrencia y Asyncio

[[Desarrollo Profesional/Python/Python|⬅️ Volver al índice]] | [[Desarrollo Profesional/Python/10 - Tipado Estático|← 10]] | [[Desarrollo Profesional/Python/12 - Testing con Pytest|12 →]]

> [!abstract] Introducción
> La concurrencia en Python tiene una particularidad fundamental: el GIL (Global Interpreter Lock) impide que múltiples threads ejecuten bytecode Python simultáneamente. Esto hace que threading sea útil para tareas I/O-bound pero inútil para CPU-bound. Asyncio resuelve el problema del I/O concurrente con un enfoque diferente — un solo thread con un event loop que gestiona miles de conexiones. Entender estas diferencias determina qué modelo usar en cada situación.

## ¿De qué vamos a hablar?

Los tres modelos de concurrencia de Python: threading (para I/O-bound), multiprocessing (para CPU-bound) y asyncio (para I/O-bound de alta escala) — cuándo usar cada uno y cómo combinarlos.

### Conceptos que vamos a cubrir
- El GIL: qué es y por qué importa
- Threading: cuándo es útil y sus limitaciones
- Multiprocessing: paralelismo real para CPU-bound
- Asyncio: el event loop y async/await
- Patrones prácticos: gather, as_completed, timeouts

---

## El Concepto

### El GIL — El Elefante en la Sala

El Global Interpreter Lock es un mutex que protege las estructuras internas del intérprete CPython. Solo un thread puede ejecutar bytecode Python a la vez.

**Consecuencias:**
- **I/O-bound:** El GIL se libera durante operaciones de I/O (red, disco, socket). Los threads sí pueden ser útiles aquí — mientras un thread espera, otro ejecuta.
- **CPU-bound:** El GIL no se libera durante cómputo intensivo. Múltiples threads con trabajo CPU-bound no son más rápidos que un solo thread.

```
Modelo de decisión:

¿Qué tipo de trabajo?
├── I/O-bound (red, disco, BD)
│   ├── Muchas conexiones, alta escala → asyncio
│   └── Pocas tareas, código síncrono existente → threading
└── CPU-bound (cómputo intensivo, procesamiento de datos)
    └── multiprocessing
```

### Threading — Concurrencia para I/O-bound

```python
import threading
from concurrent.futures import ThreadPoolExecutor, as_completed
import requests
import time

# Thread básico
def descargar(url: str) -> str:
    response = requests.get(url, timeout=10)
    return response.text[:100]

# Forma moderna: ThreadPoolExecutor
urls = [
    "https://httpbin.org/delay/1",
    "https://httpbin.org/delay/1",
    "https://httpbin.org/delay/1",
]

# Secuencial: ~3 segundos
inicio = time.perf_counter()
resultados = [descargar(url) for url in urls]
print(f"Secuencial: {time.perf_counter() - inicio:.1f}s")

# Con ThreadPoolExecutor: ~1 segundo
inicio = time.perf_counter()
with ThreadPoolExecutor(max_workers=10) as executor:
    # submit — control fino
    futuros = {executor.submit(descargar, url): url for url in urls}
    for futuro in as_completed(futuros):
        url = futuros[futuro]
        try:
            resultado = futuro.result()
            print(f"OK: {url}")
        except Exception as e:
            print(f"Error en {url}: {e}")

    # map — más simple, en orden
    resultados = list(executor.map(descargar, urls))

print(f"ThreadPool: {time.perf_counter() - inicio:.1f}s")

# Thread-safety: los threads comparten memoria — usa Lock para datos compartidos
contador = 0
lock = threading.Lock()

def incrementar_seguro():
    global contador
    with lock:   # solo un thread a la vez
        contador += 1
```

### Multiprocessing — Paralelismo Real para CPU-bound

```python
from concurrent.futures import ProcessPoolExecutor
import multiprocessing
import math

def factorizar(n: int) -> list[int]:
    """Operación CPU-bound"""
    factores = []
    d = 2
    while d * d <= n:
        while n % d == 0:
            factores.append(d)
            n //= d
        d += 1
    if n > 1:
        factores.append(n)
    return factores

numeros = [10**15 + 3, 10**15 + 7, 10**15 + 9, 10**15 + 11]

# Con ProcessPoolExecutor — usa múltiples procesos (sin GIL)
with ProcessPoolExecutor(max_workers=multiprocessing.cpu_count()) as executor:
    resultados = list(executor.map(factorizar, numeros))

# Pool de procesos para más control
from multiprocessing import Pool

with Pool(processes=4) as pool:
    # starmap — cuando la función recibe múltiples argumentos
    resultados = pool.starmap(math.pow, [(2, n) for n in range(10)])

# Compartir estado entre procesos — más complejo que threads
from multiprocessing import Value, Array

contador_compartido = Value("i", 0)  # integer compartido
array_compartido = Array("d", [0.0] * 10)  # array de doubles compartido
```

### Asyncio — Event Loop para I/O de Alta Escala

Asyncio usa un **event loop** en un solo thread. Las coroutines ceden el control al loop cuando esperan I/O, permitiendo que otras coroutines ejecuten:

```python
import asyncio
import aiohttp
import time
from typing import Any

# Coroutine básica
async def saludar(nombre: str) -> str:
    await asyncio.sleep(1)  # simula I/O — cede el control al event loop
    return f"Hola, {nombre}!"

# Ejecutar una coroutine
resultado = asyncio.run(saludar("Andrés"))

# async/await en detalle
async def main() -> None:
    # await — espera a que la coroutine termine
    resultado = await saludar("Andrés")

    # asyncio.gather — ejecuta múltiples coroutines concurrentemente
    inicio = time.perf_counter()
    nombres = ["Ana", "Luis", "Eva", "Carlos"]
    resultados = await asyncio.gather(*[saludar(n) for n in nombres])
    print(f"gather: {time.perf_counter() - inicio:.1f}s")  # ~1s, no ~4s

asyncio.run(main())
```

### Peticiones HTTP Asíncronas con aiohttp

```python
import asyncio
import aiohttp
from typing import Any

async def fetch(session: aiohttp.ClientSession, url: str) -> dict[str, Any]:
    async with session.get(url) as response:
        response.raise_for_status()
        return await response.json()

async def fetch_todos(urls: list[str]) -> list[dict]:
    async with aiohttp.ClientSession() as session:
        tareas = [fetch(session, url) for url in urls]
        resultados = await asyncio.gather(*tareas, return_exceptions=True)

    exitosos = [r for r in resultados if not isinstance(r, Exception)]
    errores   = [r for r in resultados if isinstance(r, Exception)]

    if errores:
        print(f"Errores: {len(errores)}")

    return exitosos

# asyncio.as_completed — procesar a medida que terminan (no esperar a todos)
async def procesar_progresivo(urls: list[str]) -> None:
    async with aiohttp.ClientSession() as session:
        tareas = [asyncio.create_task(fetch(session, url)) for url in urls]

        for tarea in asyncio.as_completed(tareas):
            try:
                resultado = await tarea
                print(f"Recibido: {resultado.get('url')}")
            except Exception as e:
                print(f"Error: {e}")
```

### Semaphore — Limitar Concurrencia

```python
async def descargar_con_limite(urls: list[str], max_concurrente: int = 10) -> list[str]:
    semaforo = asyncio.Semaphore(max_concurrente)  # máximo 10 a la vez

    async def descargar_uno(session: aiohttp.ClientSession, url: str) -> str:
        async with semaforo:  # bloquea si ya hay max_concurrente activos
            async with session.get(url) as r:
                return await r.text()

    async with aiohttp.ClientSession() as session:
        tareas = [descargar_uno(session, url) for url in urls]
        return await asyncio.gather(*tareas)
```

### Timeouts y Cancelación

```python
async def con_timeout(coro, timeout_seg: float):
    try:
        return await asyncio.wait_for(coro, timeout=timeout_seg)
    except asyncio.TimeoutError:
        print(f"Timeout después de {timeout_seg}s")
        return None

# Cancelar tareas
async def main():
    tarea = asyncio.create_task(operacion_larga())

    await asyncio.sleep(5)  # esperamos 5 segundos

    tarea.cancel()  # cancelamos
    try:
        await tarea
    except asyncio.CancelledError:
        print("Tarea cancelada")
```

### Combinar Asyncio con Código Bloqueante

```python
import asyncio
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor

async def main():
    loop = asyncio.get_event_loop()

    # Ejecutar función bloqueante (I/O) en thread pool sin bloquear el event loop
    with ThreadPoolExecutor() as pool:
        resultado = await loop.run_in_executor(pool, funcion_bloqueante_io, arg)

    # Ejecutar función CPU-bound en process pool
    with ProcessPoolExecutor() as pool:
        resultado = await loop.run_in_executor(pool, funcion_cpu_intensiva, arg)
```

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - El GIL impide paralelismo CPU real con threads. **Threading:** útil para I/O-bound. **Multiprocessing:** para CPU-bound. **Asyncio:** para I/O-bound de alta escala (miles de conexiones).
> - `ThreadPoolExecutor` y `ProcessPoolExecutor` son la API moderna. Evita `Thread` y `Process` directamente.
> - Asyncio usa un event loop en un solo thread. `await` cede el control al loop. `asyncio.gather()` ejecuta coroutines concurrentemente.
> - Usa `asyncio.Semaphore` para limitar la concurrencia (evitar sobrecargar APIs externas o la BD).
> - Para código bloqueante dentro de asyncio, usa `loop.run_in_executor` — no llames funciones bloqueantes directamente dentro de coroutines.

### Para llevar a la práctica
- [ ] Mide la diferencia entre hacer 10 peticiones HTTP secuencialmente vs con `asyncio.gather`
- [ ] Implementa un scraper asíncrono con `aiohttp` y `asyncio.Semaphore(5)` para respetar límites de la API
- [ ] Usa `ProcessPoolExecutor` para paralelizar un cálculo intensivo sobre una lista de datos

### Recursos
- 📖 *Fluent Python* — Capítulo 19-21 (concurrencia con threads, asyncio)
- 📖 *Python Concurrency with asyncio* — Matthew Fowler (el libro más completo sobre asyncio)
- 🌐 docs.python.org/3/library/asyncio.html
- 📄 PEP 3156 — Asynchronous I/O Support Rebooted

---
`#python` `#asyncio` `#concurrencia` `#threading` `#multiprocessing`
