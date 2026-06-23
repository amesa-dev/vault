# 🐍 08 — Iteradores y Generadores

[[Desarrollo Profesional/Python/Python|⬅️ Volver al índice]] | [[Desarrollo Profesional/Python/07 - Manejo de Errores|← 07]] | [[Desarrollo Profesional/Python/09 - Decoradores|09 →]]

> [!abstract] Introducción
> Los generadores son una de las características más poderosas y subutilizadas de Python. Permiten crear secuencias de valores "perezosamente" — sin calcularlos todos de golpe ni cargarlos en memoria. Un generador puede producir una secuencia infinita con O(1) de memoria. Pero más importante: el protocolo de iteradores es la base de todo el modelo de datos de Python — entenderlo desbloquea el potencial del lenguaje.

## ¿De qué vamos a hablar?

El protocolo de iteradores de Python y los generadores: cómo funcionan internamente, cuándo usarlos y cómo el módulo `itertools` te da herramientas de procesamiento de datos sin igual.

### Conceptos que vamos a cubrir
- El protocolo iterador: `__iter__` y `__next__`
- Cómo funciona `for` internamente
- Funciones generadoras con `yield`
- `yield from` para delegación
- Generadores como corutinas (comunicación bidireccional)
- `itertools`: el módulo más infrautilizado

---

## El Concepto

### El Protocolo Iterador

Python define dos protocolos:

**Iterable:** Objeto que implementa `__iter__()` — retorna un iterador.
**Iterador:** Objeto que implementa `__next__()` — retorna el siguiente valor o lanza `StopIteration`.

```python
# Cómo funciona 'for' internamente
lista = [1, 2, 3]

# for x in lista: equivale a:
iterador = iter(lista)     # llama lista.__iter__()
while True:
    try:
        x = next(iterador) # llama iterador.__next__()
        print(x)
    except StopIteration:
        break

# Implementar el protocolo manualmente
class ContadorHasta:
    def __init__(self, limite: int) -> None:
        self.limite = limite

    def __iter__(self) -> "ContadorHasta":
        self.actual = 0
        return self  # el iterador se retorna a sí mismo

    def __next__(self) -> int:
        if self.actual >= self.limite:
            raise StopIteration
        valor = self.actual
        self.actual += 1
        return valor

for n in ContadorHasta(5):
    print(n)  # 0 1 2 3 4
```

### Funciones Generadoras con `yield`

Una función generadora usa `yield` para producir valores uno a uno. Cada llamada a `next()` ejecuta hasta el siguiente `yield` y pausa:

```python
def contar_hasta(limite: int):
    n = 0
    while n < limite:
        yield n    # pausa aquí, retorna n
        n += 1     # reanuda aquí en la siguiente llamada

# El generador no ejecuta nada hasta que se llama next()
gen = contar_hasta(5)
print(next(gen))  # 0 — ejecuta hasta el primer yield
print(next(gen))  # 1 — reanuda, ejecuta hasta el siguiente yield

# Normalmente se consume con for
for n in contar_hasta(5):
    print(n)

# Generadores vs listas — comparación de memoria
import sys

lista = [x**2 for x in range(1_000_000)]
gen   = (x**2 for x in range(1_000_000))

print(sys.getsizeof(lista))  # ~8 MB
print(sys.getsizeof(gen))    # ~104 bytes — el generador no ha calculado nada

# Casos de uso prácticos
def leer_lineas_csv(archivo: str):
    """Lee un archivo CSV línea a línea — sin cargar todo en memoria"""
    with open(archivo) as f:
        encabezado = next(f).strip().split(",")
        for linea in f:
            valores = linea.strip().split(",")
            yield dict(zip(encabezado, valores))

# Procesamos un CSV de 10GB con O(1) de memoria
for fila in leer_lineas_csv("datos_enormes.csv"):
    procesar(fila)

def fibonacci():
    """Generador infinito de Fibonacci"""
    a, b = 0, 1
    while True:
        yield a
        a, b = b, a + b

# Tomar solo los primeros N valores de una secuencia infinita
from itertools import islice
primeros_10 = list(islice(fibonacci(), 10))  # [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]
```

### `yield from` — Delegación de Generadores

`yield from` delega la iteración a otro iterable/generador:

```python
def cadena(*iterables):
    for iterable in iterables:
        yield from iterable  # equivale a: for item in iterable: yield item

list(cadena([1, 2], [3, 4], [5]))  # [1, 2, 3, 4, 5]

# Caso de uso real: aplanar estructuras anidadas
def aplanar(anidado):
    for item in anidado:
        if isinstance(item, (list, tuple)):
            yield from aplanar(item)  # recursividad sin acumular en memoria
        else:
            yield item

list(aplanar([1, [2, [3, 4]], [5, 6]]))  # [1, 2, 3, 4, 5, 6]
```

### `itertools` — El Módulo Más Poderoso

```python
import itertools

# Combinatorias
list(itertools.product([1,2], ["a","b"]))     # [(1,'a'),(1,'b'),(2,'a'),(2,'b')]
list(itertools.combinations([1,2,3], 2))      # [(1,2),(1,3),(2,3)]
list(itertools.permutations([1,2,3], 2))      # [(1,2),(1,3),(2,1),(2,3),(3,1),(3,2)]

# Secuencias infinitas
counter = itertools.count(start=0, step=2)   # 0, 2, 4, 6, ... (infinito)
cycler  = itertools.cycle("ABC")              # A, B, C, A, B, C, ...
repeater = itertools.repeat(42, times=3)     # 42, 42, 42

# Transformaciones de secuencias
# chain — concatenar iterables sin crear lista intermedia
for x in itertools.chain([1,2], [3,4], [5]):
    print(x)  # 1 2 3 4 5

# islice — slice de iteradores (incluso infinitos)
list(itertools.islice(itertools.count(), 5))  # [0, 1, 2, 3, 4]

# takewhile / dropwhile — filtrar hasta/desde una condición
list(itertools.takewhile(lambda x: x < 5, [1,2,3,6,2,1]))  # [1,2,3]
list(itertools.dropwhile(lambda x: x < 5, [1,2,3,6,2,1]))  # [6,2,1]

# groupby — agrupar por clave (los datos deben estar ordenados por esa clave)
datos = [
    {"tipo": "A", "valor": 1},
    {"tipo": "A", "valor": 2},
    {"tipo": "B", "valor": 3},
    {"tipo": "B", "valor": 4},
]
for tipo, grupo in itertools.groupby(datos, key=lambda x: x["tipo"]):
    print(tipo, list(grupo))

# accumulate — acumulación con función personalizada
import operator
list(itertools.accumulate([1,2,3,4,5], operator.add))    # [1, 3, 6, 10, 15] (suma acumulada)
list(itertools.accumulate([1,2,3,4,5], operator.mul))    # [1, 2, 6, 24, 120] (factorial!)

# pairwise (Python 3.10+) — pares consecutivos
list(itertools.pairwise([1,2,3,4]))  # [(1,2),(2,3),(3,4)]
```

### Pipeline de Procesamiento de Datos

Los generadores permiten construir pipelines de procesamiento eficientes en memoria:

```python
from typing import Iterable, Generator

def leer_logs(archivo: str) -> Generator[str, None, None]:
    with open(archivo) as f:
        yield from f

def filtrar_errores(lineas: Iterable[str]) -> Generator[str, None, None]:
    for linea in lineas:
        if "ERROR" in linea:
            yield linea

def parsear(lineas: Iterable[str]) -> Generator[dict, None, None]:
    for linea in lineas:
        partes = linea.strip().split(" ", 3)
        yield {
            "fecha": partes[0],
            "nivel": partes[1],
            "mensaje": partes[2] if len(partes) > 2 else "",
        }

# Pipeline lazy — ningún paso carga todo en memoria
pipeline = parsear(filtrar_errores(leer_logs("app.log")))

# Solo en este punto se procesa cada línea
for entrada in pipeline:
    print(entrada["mensaje"])
```

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - El protocolo iterador (`__iter__`, `__next__`) es la base de todo el bucle `for` en Python. Un iterable produce un iterador; el iterador produce valores uno a uno.
> - Los generadores pausan en cada `yield` y retoman desde ahí en el siguiente `next()`. Usan O(1) de memoria independientemente del tamaño de la secuencia.
> - `yield from` delega la iteración a otro iterable, simplificando generadores recursivos.
> - `itertools` proporciona herramientas de procesamiento de secuencias altamente optimizadas: `chain`, `islice`, `groupby`, `accumulate`, `pairwise`, etc.
> - El patrón pipeline (generadores encadenados) es la forma más eficiente de procesar grandes volúmenes de datos en Python.

### Para llevar a la práctica
- [ ] Implementa una clase `RangoDecimal` iterable que produzca `0.1, 0.2, ... hasta N`
- [ ] Crea un pipeline de generadores que lea un CSV, filtre filas, y transforme campos
- [ ] Usa `itertools.groupby` para agrupar una lista de objetos por una propiedad

### Recursos
- 📖 *Fluent Python* — Capítulo 14 (iterables, iteradores y generadores)
- 🌐 docs.python.org/3/library/itertools.html — referencia completa de itertools
- 📄 PEP 255 — Simple Generators
- 📄 PEP 380 — Syntax for Delegating to a Subgenerator (yield from)

---
`#python` `#generadores` `#iteradores` `#itertools` `#lazy`
