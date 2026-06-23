# 🐍 09 — Decoradores

[[Desarrollo Profesional/Python/Python|⬅️ Volver al índice]] | [[Desarrollo Profesional/Python/08 - Iteradores y Generadores|← 08]] | [[Desarrollo Profesional/Python/10 - Tipado Estático|10 →]]

> [!abstract] Introducción
> Los decoradores son una de las características más elegantes de Python. Son funciones que envuelven otras funciones o clases para modificar o extender su comportamiento sin cambiar su código. `@property`, `@classmethod`, `@staticmethod`, `@dataclass`, `@lru_cache` — todo lo que has visto con `@` es un decorador. Esta página te enseña a crear los tuyos propios.

## ¿De qué vamos a hablar?

Los decoradores desde sus fundamentos matemáticos (son funciones de orden superior) hasta los patrones avanzados: decoradores con argumentos, decoradores de clase y el stacking.

### Conceptos que vamos a cubrir
- Decoradores simples de función
- `functools.wraps`: preservar metadatos
- Decoradores con argumentos (decorator factories)
- Stacking de decoradores
- Decoradores de clase
- Casos de uso reales: logging, timing, retry, caché, validación

---

## El Concepto

### La Mecánica

Un decorador es una función que recibe una función y retorna una función (o clase). El `@` es azúcar sintáctica:

```python
def mi_decorador(func):
    def wrapper(*args, **kwargs):
        print(f"Antes de {func.__name__}")
        resultado = func(*args, **kwargs)
        print(f"Después de {func.__name__}")
        return resultado
    return wrapper

@mi_decorador
def saludar(nombre: str) -> str:
    return f"Hola, {nombre}!"

# Equivale exactamente a:
# saludar = mi_decorador(saludar)

saludar("Andrés")
# "Antes de saludar"
# "Después de saludar"
```

### `functools.wraps` — Preservar Metadatos

Sin `@wraps`, el decorador oculta la identidad de la función original:

```python
from functools import wraps

# Sin wraps — pierde la identidad
def decorador_malo(func):
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)
    return wrapper

@decorador_malo
def mi_funcion():
    """Esta es mi función"""
    pass

print(mi_funcion.__name__)  # 'wrapper' — MAL
print(mi_funcion.__doc__)   # None — perdimos el docstring

# Con wraps — preserva metadatos
def decorador_bueno(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)
    return wrapper

@decorador_bueno
def mi_funcion():
    """Esta es mi función"""
    pass

print(mi_funcion.__name__)  # 'mi_funcion' — BIEN
print(mi_funcion.__doc__)   # 'Esta es mi función' — BIEN
```

> Regla: Siempre usa `@functools.wraps(func)` en el wrapper. Sin ello, el debugging, la documentación automática y los sistemas que inspeccionan funciones se rompen.

### Decoradores con Argumentos — Decorator Factories

Para pasar argumentos al decorador, necesitas una función que retorne un decorador:

```python
from functools import wraps
import time

def reintentar(max_intentos: int = 3, espera: float = 1.0, excepciones: tuple = (Exception,)):
    """Decorador que reintenta la función en caso de excepción"""
    def decorador(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for intento in range(1, max_intentos + 1):
                try:
                    return func(*args, **kwargs)
                except excepciones as e:
                    if intento == max_intentos:
                        raise
                    print(f"Intento {intento}/{max_intentos} falló: {e}. Esperando {espera}s...")
                    time.sleep(espera)
        return wrapper
    return decorador

@reintentar(max_intentos=3, espera=0.5, excepciones=(ConnectionError, TimeoutError))
def llamar_api(url: str) -> dict:
    # simula una llamada que puede fallar
    response = requests.get(url, timeout=5)
    response.raise_for_status()
    return response.json()

# Uso con parentheses — el decorador tiene argumentos
@reintentar(max_intentos=5)
def otra_funcion():
    pass

# Uso sin paréntesis — con defaults
@reintentar()
def tercera_funcion():
    pass
```

### Decoradores Prácticos

```python
import time
import logging
from functools import wraps
from typing import TypeVar, Callable, Any

F = TypeVar("F", bound=Callable[..., Any])

# Decorador de timing
def cronometrar(func: F) -> F:
    @wraps(func)
    def wrapper(*args, **kwargs):
        inicio = time.perf_counter()
        try:
            return func(*args, **kwargs)
        finally:
            fin = time.perf_counter()
            print(f"{func.__name__}: {fin - inicio:.4f}s")
    return wrapper  # type: ignore[return-value]

# Decorador de logging
def loggear(nivel: str = "INFO"):
    def decorador(func):
        logger = logging.getLogger(func.__module__)
        log_fn = getattr(logger, nivel.lower())

        @wraps(func)
        def wrapper(*args, **kwargs):
            log_fn(f"Llamando a {func.__name__} con args={args}, kwargs={kwargs}")
            try:
                resultado = func(*args, **kwargs)
                log_fn(f"{func.__name__} retornó: {resultado!r}")
                return resultado
            except Exception as e:
                logger.error(f"{func.__name__} falló con: {e!r}")
                raise
        return wrapper
    return decorador

# Decorador de validación de tipos en runtime
def validar_tipos(func):
    import inspect
    hints = func.__annotations__

    @wraps(func)
    def wrapper(*args, **kwargs):
        sig = inspect.signature(func)
        bound = sig.bind(*args, **kwargs)
        bound.apply_defaults()

        for nombre, valor in bound.arguments.items():
            if nombre in hints and nombre != "return":
                tipo_esperado = hints[nombre]
                if not isinstance(valor, tipo_esperado):
                    raise TypeError(
                        f"Argumento '{nombre}': se esperaba {tipo_esperado.__name__}, "
                        f"se recibió {type(valor).__name__}"
                    )
        return func(*args, **kwargs)
    return wrapper

@validar_tipos
def dividir(a: float, b: float) -> float:
    return a / b

dividir(10.0, 2.0)  # ✅
dividir(10, 2)      # TypeError: se esperaba float, se recibió int
```

### Stacking de Decoradores

```python
@decorador_a
@decorador_b
@decorador_c
def funcion():
    pass

# Equivale a:
funcion = decorador_a(decorador_b(decorador_c(funcion)))
# El orden de aplicación es de abajo hacia arriba
# El orden de ejecución del wrapper es de arriba hacia abajo

@cronometrar
@loggear("DEBUG")
@reintentar(max_intentos=3)
def llamar_servicio():
    pass
# Ejecución: cronometrar (más externo) → loggear → reintentar → llamar_servicio
```

### Decoradores de Clase

Los decoradores pueden aplicarse a clases, no solo a funciones:

```python
from dataclasses import dataclass

def singleton(cls):
    """Convierte una clase en Singleton"""
    instancias = {}

    @wraps(cls)
    def obtener_instancia(*args, **kwargs):
        if cls not in instancias:
            instancias[cls] = cls(*args, **kwargs)
        return instancias[cls]
    return obtener_instancia

@singleton
class Configuracion:
    def __init__(self):
        self.valores = {}

c1 = Configuracion()
c2 = Configuracion()
print(c1 is c2)  # True — misma instancia

# Decorador que añade métodos a una clase
def con_repr_automatico(cls):
    """Añade un __repr__ automático basado en los campos"""
    @wraps(cls, updated=[])
    class Wrapped(cls):
        def __repr__(self) -> str:
            campos = {k: v for k, v in self.__dict__.items() if not k.startswith("_")}
            args_str = ", ".join(f"{k}={v!r}" for k, v in campos.items())
            return f"{cls.__name__}({args_str})"
    return Wrapped

@con_repr_automatico
class Producto:
    def __init__(self, nombre: str, precio: float):
        self.nombre = nombre
        self.precio = precio

print(Producto("Teclado", 89.99))  # Producto(nombre='Teclado', precio=89.99)
```

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - Un decorador es una función que recibe una función y retorna una función. `@mi_decorador` es azúcar para `funcion = mi_decorador(funcion)`.
> - Siempre usa `@functools.wraps(func)` en el wrapper — preserva `__name__`, `__doc__` y otros metadatos.
> - Para decoradores con argumentos, necesitas una función que retorne un decorador (tres niveles de anidación).
> - El stacking de decoradores aplica de dentro hacia fuera, pero los wrappers ejecutan de fuera hacia dentro.
> - Los decoradores de clase se aplican a toda la clase, no a un método — útiles para Singleton, registro automático, validación de invariantes.

### Para llevar a la práctica
- [ ] Implementa `@cache_con_ttl(segundos=60)` que cachea el resultado de una función por N segundos
- [ ] Crea `@requiere_autenticacion` que verifica si el primer argumento de una función tiene un campo `autenticado=True`
- [ ] Combina `@cronometrar` + `@loggear` sobre una función y verifica el orden de ejecución

### Recursos
- 📖 *Fluent Python* — Capítulo 9 (decoradores y closures)
- 🌐 docs.python.org/3/library/functools.html
- 📄 PEP 318 — Decorators for Functions and Methods
- 🌐 realpython.com/primer-on-python-decorators

---
`#python` `#decoradores` `#orden-superior` `#metaprogramacion`
