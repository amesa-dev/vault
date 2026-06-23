# 🐍 04 — Funciones

[[Desarrollo Profesional/Python/Python|⬅️ Volver al índice]] | [[Desarrollo Profesional/Python/03 - Control de Flujo|← 03]] | [[Desarrollo Profesional/Python/05 - POO|05 →]]

> [!abstract] Introducción
> En Python las funciones son objetos de primera clase: se pueden pasar como argumentos, retornar desde otras funciones y almacenar en variables. Esta característica, combinada con el sistema de argumentos flexible de Python (*args, **kwargs), hace que las funciones sean la unidad de abstracción más poderosa del lenguaje.

## ¿De qué vamos a hablar?

Desde la definición básica hasta los patrones avanzados: closures, funciones de orden superior y cómo el sistema de argumentos de Python permite APIs flexibles y expresivas.

### Conceptos que vamos a cubrir
- Parámetros: posicionales, nombrados, *args, **kwargs, keyword-only
- Funciones como objetos de primera clase
- Closures y el alcance (scope)
- Lambda y limitaciones
- Funciones de orden superior: map, filter, functools

---

## El Concepto

### Parámetros: El Sistema Completo

```python
# Parámetros posicionales y nombrados
def crear_usuario(nombre: str, edad: int, activo: bool = True) -> dict:
    return {"nombre": nombre, "edad": edad, "activo": activo}

crear_usuario("Andrés", 30)                    # posicionales
crear_usuario(nombre="Andrés", edad=30)        # nombrados
crear_usuario("Andrés", edad=30, activo=False) # mezcla

# *args — argumentos posicionales variables
def sumar(*numeros: int) -> int:
    return sum(numeros)

sumar(1, 2, 3, 4, 5)  # 15
sumar(*[1, 2, 3])      # 6 — unpacking de lista

# **kwargs — argumentos nombrados variables
def configurar(**opciones: str) -> None:
    for clave, valor in opciones.items():
        print(f"{clave} = {valor}")

configurar(host="localhost", puerto="5432", db="produccion")

# Combinando todos los tipos
def funcion_completa(
    pos1: int,
    pos2: int,
    /,              # todo lo anterior es solo posicional (Python 3.8+)
    normal: str,
    *args: int,     # posicionales variables
    kw_only: bool,  # keyword-only (solo nombrado, no posicional)
    **kwargs: str,  # nombrados variables
) -> None:
    print(pos1, pos2, normal, args, kw_only, kwargs)

funcion_completa(1, 2, "x", 3, 4, kw_only=True, extra="valor")
# funcion_completa(pos1=1, ...) — Error: pos1 es solo posicional

# Parámetros keyword-only — después del *
def conectar(host: str, *, puerto: int = 5432, ssl: bool = False) -> None:
    """puerto y ssl son keyword-only — obligan a usarlos con nombre"""
    print(f"Conectando a {host}:{puerto} (ssl={ssl})")

conectar("localhost")              # ✅
conectar("localhost", puerto=5433) # ✅
conectar("localhost", 5433)        # Error: porta es keyword-only
```

**Por qué keyword-only:** Hace las APIs más robustas. Si alguien llama `conectar("localhost", True)` sin `ssl=`, el código sería ambiguo. Con keyword-only, el error es inmediato.

### Funciones como Objetos de Primera Clase

```python
# Las funciones son objetos — tienen atributos
def saludar(nombre: str) -> str:
    return f"Hola, {nombre}!"

print(type(saludar))          # <class 'function'>
print(saludar.__name__)       # 'saludar'
print(saludar.__doc__)        # docstring si tiene

# Se pueden asignar a variables
mi_funcion = saludar
mi_funcion("Andrés")          # "Hola, Andrés!"

# Se pueden guardar en listas y dicts
operaciones = {
    "suma":  lambda a, b: a + b,
    "resta": lambda a, b: a - b,
    "mult":  lambda a, b: a * b,
}
resultado = operaciones["suma"](3, 4)  # 7

# Se pueden pasar como argumentos
def aplicar(funcion, valor):
    return funcion(valor)

aplicar(str.upper, "hola")  # "HOLA"
aplicar(len, [1, 2, 3])     # 3

# Se pueden retornar desde funciones (factory pattern)
def crear_multiplicador(factor: int):
    def multiplicar(x: int) -> int:
        return x * factor
    return multiplicar

doblar   = crear_multiplicador(2)
triplicar = crear_multiplicador(3)

doblar(5)    # 10
triplicar(5) # 15
```

### Closures — Funciones que Recuerdan

Un closure es una función que "recuerda" las variables de su scope externo, incluso después de que la función externa haya retornado.

```python
def crear_contador(inicio: int = 0):
    cuenta = inicio  # esta variable "vive" en el closure

    def incrementar(paso: int = 1) -> int:
        nonlocal cuenta  # declara que queremos modificar la variable del scope externo
        cuenta += paso
        return cuenta

    def obtener() -> int:
        return cuenta

    return incrementar, obtener

incrementar, obtener = crear_contador(10)
incrementar()   # 11
incrementar(5)  # 16
obtener()       # 16
```

**`nonlocal`:** Necesario cuando quieres **modificar** una variable del scope externo (no solo leerla). Sin `nonlocal`, Python crearía una variable local nueva.

**Uso práctico:** Closures son la base de los decoradores y de patrones como el factory de funciones. Son más ligeras que las clases para estado simple.

### Lambda — Funciones Anónimas

Las lambdas son funciones de una expresión. Su limitación es que no pueden contener sentencias (solo expresiones).

```python
# Lambda básica
doble = lambda x: x * 2
doble(5)  # 10

# Útil como argumento a sorted, map, filter
personas = [{"nombre": "Eva", "edad": 22}, {"nombre": "Luis", "edad": 30}]
ordenado = sorted(personas, key=lambda p: p["edad"])

# Con múltiples parámetros
sumar = lambda a, b: a + b
sumar(3, 4)  # 7

# Cuándo NO usar lambda
# Esta lambda no aporta nada — usa la función directamente
lista = [1, 2, 3]
resultado = map(lambda x: str(x), lista)  # ❌ innecesario
resultado = map(str, lista)               # ✅ más claro
```

**Regla:** Si la lambda tiene más de una expresión simple, define una función con nombre. Las lambdas sin nombre son difíciles de debuggear y de documentar.

### Funciones de Orden Superior

```python
# map — aplica función a cada elemento (retorna iterator)
nums = [1, 2, 3, 4, 5]
cuadrados = list(map(lambda x: x**2, nums))  # [1, 4, 9, 16, 25]
# Alternativa más pythónica:
cuadrados = [x**2 for x in nums]

# filter — filtra elementos según predicado
pares = list(filter(lambda x: x % 2 == 0, nums))  # [2, 4]
# Alternativa:
pares = [x for x in nums if x % 2 == 0]

# reduce — acumula resultado (necesita import)
from functools import reduce
producto = reduce(lambda acc, x: acc * x, nums)  # 120 (1*2*3*4*5)

# functools — herramientas para funciones de orden superior
from functools import partial, lru_cache

# partial — fija algunos argumentos de una función
def potencia(base: int, exponente: int) -> int:
    return base ** exponente

cuadrado = partial(potencia, exponente=2)
cubo     = partial(potencia, exponente=3)
cuadrado(5)  # 25
cubo(3)      # 27

# lru_cache — memoización automática con Least Recently Used cache
@lru_cache(maxsize=128)
def fibonacci(n: int) -> int:
    if n <= 1:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)

fibonacci(50)  # instantáneo con caché

# cache (Python 3.9+) — lru_cache sin límite de tamaño
from functools import cache

@cache
def factorial(n: int) -> int:
    return 1 if n <= 1 else n * factorial(n - 1)
```

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - Python tiene el sistema de argumentos más flexible de los lenguajes mainstream: posicionales, nombrados, *args, **kwargs, positional-only (`/`) y keyword-only (`*`).
> - Las funciones son objetos: se pasan, retornan y almacenan como cualquier valor. Esta es la base del paradigma funcional en Python.
> - Los closures recuerdan el scope externo. Usa `nonlocal` cuando necesitas modificar una variable del scope externo.
> - Las lambdas son para expresiones simples en línea. Si necesitas más de una expresión, usa `def`.
> - `functools.lru_cache` / `functools.cache` añaden memoización a cualquier función pura con una sola línea.

### Para llevar a la práctica
- [ ] Escribe una función `retry(funcion, intentos, *args, **kwargs)` que reintenta una función N veces en caso de excepción
- [ ] Implementa un closure que funcione como acumulador con historial de operaciones
- [ ] Usa `functools.partial` para crear versiones especializadas de una función genérica

### Recursos
- 📖 *Fluent Python* — Capítulo 7 (funciones como objetos de primera clase)
- 📄 PEP 3102 — Keyword-Only Arguments
- 📄 PEP 570 — Positional-Only Parameters
- 🌐 docs.python.org/3/library/functools.html

---
`#python` `#funciones` `#closures` `#orden-superior`
