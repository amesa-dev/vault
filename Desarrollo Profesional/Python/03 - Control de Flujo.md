# 🐍 03 — Control de Flujo

[[Desarrollo Profesional/Python/Python|⬅️ Volver al índice]] | [[Desarrollo Profesional/Python/02 - Estructuras de Datos|← 02]] | [[Desarrollo Profesional/Python/04 - Funciones|04 →]]

> [!abstract] Introducción
> El control de flujo determina qué código se ejecuta y cuándo. Python tiene las estructuras estándar (if, for, while) más algunas que lo distinguen de otros lenguajes: el `else` en bucles, las comprehensions como alternativa a `map/filter`, y el `match-case` de Python 3.10 que es uno de los pattern matching más expresivos de cualquier lenguaje mainstream.

## ¿De qué vamos a hablar?

Más allá de la sintaxis básica — que ya conoces — el foco está en los patrones idiomáticos de Python: cuándo usar comprehensions vs bucles explícitos, cómo funciona el `else` en bucles, y cómo el `match-case` supera al switch tradicional.

### Conceptos que vamos a cubrir
- if/elif/else y operador ternario
- for: iteración sobre secuencias, enumerate, zip
- while y el else en bucles
- Comprehensions: list, dict, set, generator
- match-case: pattern matching estructural (Python 3.10+)

---

## El Concepto

### if/elif/else

```python
nota = 75

if nota >= 90:
    calificacion = "A"
elif nota >= 80:
    calificacion = "B"
elif nota >= 70:
    calificacion = "C"
else:
    calificacion = "F"

# Operador ternario (expresión condicional)
estado = "aprobado" if nota >= 60 else "suspenso"

# Condiciones compuestas
edad = 25
tiene_carnet = True
puede_conducir = edad >= 18 and tiene_carnet

# Python permite encadenar comparaciones (muy legible)
x = 5
if 0 < x < 10:          # equivale a 0 < x and x < 10
    print("entre 0 y 10")

temperatura = 20
if 15 <= temperatura <= 25:
    print("temperatura agradable")
```

### for — Iteración

Python itera sobre cualquier objeto iterable, no solo rangos numéricos.

```python
# Iterar sobre secuencias
for fruta in ["manzana", "pera", "naranja"]:
    print(fruta)

# range() — genera números sin crear lista
for i in range(5):          # 0, 1, 2, 3, 4
    print(i)
for i in range(2, 10, 2):   # 2, 4, 6, 8
    print(i)

# enumerate() — cuando necesitas índice Y valor
for i, fruta in enumerate(["manzana", "pera"]):
    print(f"{i}: {fruta}")  # "0: manzana", "1: pera"

for i, fruta in enumerate(["manzana", "pera"], start=1):
    print(f"{i}: {fruta}")  # "1: manzana", "2: pera"

# zip() — iterar sobre múltiples secuencias en paralelo
nombres = ["Ana", "Luis", "Eva"]
edades  = [25, 30, 22]
for nombre, edad in zip(nombres, edades):
    print(f"{nombre} tiene {edad} años")

# zip(strict=True) lanza error si las listas tienen distinto tamaño (Python 3.10+)
for n, e in zip(nombres, edades, strict=True):
    print(n, e)

# Iteración sobre dict
persona = {"nombre": "Andrés", "edad": 30}
for clave in persona:                  # solo claves
    print(clave)
for clave, valor in persona.items():  # pares
    print(f"{clave}: {valor}")

# break y continue
for n in range(10):
    if n == 3:
        continue   # salta esta iteración
    if n == 7:
        break      # sale del bucle
    print(n)       # 0 1 2 4 5 6
```

### while y el else en Bucles

```python
# while básico
contador = 0
while contador < 5:
    print(contador)
    contador += 1

# El else en bucles — único en Python
# Se ejecuta si el bucle termina normalmente (sin break)
for n in range(5):
    if n == 10:
        break
else:
    print("Bucle completado sin break")  # esto se imprime

# Uso práctico: búsqueda
def buscar_primo(numeros: list[int]) -> int | None:
    for n in numeros:
        if all(n % i != 0 for i in range(2, n)):
            break   # encontramos un primo
    else:
        return None  # no encontramos primo — el for terminó sin break
    return n
```

### Comprehensions — La Forma Pythónica

Las comprehensions son expresiones que crean colecciones de forma concisa. Son más rápidas que los bucles explícitos equivalentes porque están optimizadas internamente.

```python
# List comprehension
cuadrados = [x**2 for x in range(10)]
pares     = [x for x in range(20) if x % 2 == 0]
aplanada  = [item for sublista in [[1,2],[3,4]] for item in sublista]

# Dict comprehension
cuadrados_dict = {x: x**2 for x in range(5)}
invertido       = {v: k for k, v in {"a": 1, "b": 2}.items()}

# Set comprehension
letras_unicas = {c.lower() for c in "Andrés Mesa" if c.isalpha()}

# Generator expression — como list comprehension pero lazy (no carga todo en memoria)
suma_cuadrados = sum(x**2 for x in range(1000000))  # no crea la lista intermedia
primer_par = next(x for x in range(100) if x > 50 and x % 2 == 0)  # 52

# Cuándo NO usar comprehensions
# Si la lógica es compleja, un bucle explícito es más legible
# Regla: si cabe en una línea legible, comprehension. Si no, bucle for.
```

**Rendimiento:** Las comprehensions son 30-50% más rápidas que bucles equivalentes con `append()`. Las generator expressions usan O(1) de memoria (vs O(n) de list comprehensions).

### match-case — Pattern Matching (Python 3.10+)

El `match` de Python no es un switch/case de C. Es **pattern matching estructural** — puede descomponer objetos, tuplas y dataclasses.

```python
# Matching básico
def describir_codigo(codigo: int) -> str:
    match codigo:
        case 200:
            return "OK"
        case 404:
            return "Not Found"
        case 500 | 502 | 503:  # OR de patrones
            return "Server Error"
        case c if c >= 400:   # guard — condición adicional
            return f"Client Error ({c})"
        case _:               # wildcard — cualquier valor
            return "Unknown"

# Matching de secuencias
def procesar_comando(comando: list[str]) -> None:
    match comando:
        case ["salir"]:
            print("Cerrando...")
        case ["ir", destino]:              # desempaqueta y asigna
            print(f"Yendo a {destino}")
        case ["ir", destino, "rapido"]:
            print(f"Yendo rápido a {destino}")
        case ["ir", *resto]:              # rest pattern
            print(f"Ir con args: {resto}")
        case _:
            print("Comando no reconocido")

procesar_comando(["ir", "norte"])         # "Yendo a norte"
procesar_comando(["ir", "sur", "rapido"]) # "Yendo rápido a sur"

# Matching de objetos (dataclasses y objetos con __match_args__)
from dataclasses import dataclass

@dataclass
class Punto:
    x: float
    y: float

@dataclass
class Circulo:
    centro: Punto
    radio: float

@dataclass
class Rectangulo:
    esquina: Punto
    ancho: float
    alto: float

def area(forma) -> float:
    match forma:
        case Circulo(radio=r):          # extrae el atributo radio
            return 3.14159 * r ** 2
        case Rectangulo(ancho=w, alto=h):
            return w * h
        case _:
            raise ValueError("Forma desconocida")

# Matching de dicts
def procesar_evento(evento: dict) -> None:
    match evento:
        case {"tipo": "clic", "x": x, "y": y}:
            print(f"Clic en ({x}, {y})")
        case {"tipo": "tecla", "codigo": codigo}:
            print(f"Tecla: {codigo}")
        case {"tipo": tipo, **resto}:  # captura el tipo y el resto
            print(f"Evento desconocido: {tipo}, datos: {resto}")
```

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - `for x in iterable` es la forma Python. Usa `enumerate()` para índice+valor, `zip()` para iterar listas en paralelo.
> - El `else` en bucles se ejecuta si el bucle termina **sin** `break`. Útil para búsquedas donde necesitas saber si encontraste algo.
> - Las comprehensions son más rápidas que bucles con `append()` y más legibles si caben en una línea. Las generator expressions son lazy — úsalas cuando no necesitas la lista completa.
> - `match-case` es mucho más que un switch: puede descomponer secuencias, dataclasses y dicts. Las guards (`case x if condición`) añaden condiciones adicionales.

### Para llevar a la práctica
- [ ] Reescribe un bucle `for` + `append` como list comprehension
- [ ] Usa `match-case` para modelar el procesamiento de un comando CLI que acepta subcomandos
- [ ] Implementa una búsqueda con el patrón `for/else` para indicar si se encontró el elemento

### Recursos
- 🌐 docs.python.org/3/reference/compound_stmts.html#the-match-statement
- 📄 PEP 634 — Structural Pattern Matching (la especificación completa)
- 📖 *Fluent Python* — Capítulo 2 (comprehensions y generadores)

---
`#python` `#control-flujo` `#comprehensions` `#match` `#bucles`
