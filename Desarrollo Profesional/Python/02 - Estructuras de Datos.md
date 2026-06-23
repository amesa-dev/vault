# 🐍 02 — Estructuras de Datos

[[Desarrollo Profesional/Python/Python|⬅️ Volver al índice]] | [[Desarrollo Profesional/Python/01 - Tipos Primitivos y Variables|← 01]] | [[Desarrollo Profesional/Python/03 - Control de Flujo|03 →]]

> [!abstract] Introducción
> Python incluye en su librería estándar estructuras de datos potentes y bien diseñadas. Elegir la correcta para cada problema no es trivial y tiene impacto real en rendimiento y legibilidad. Esta página cubre las estructuras que usarás el 95% del tiempo, con el contexto de cuándo usar cada una.

## ¿De qué vamos a hablar?

Las cuatro estructuras de datos fundamentales de Python y sus variantes especializadas: cuándo usar cada una, sus complejidades de acceso/inserción y los patrones más comunes.

### Conceptos que vamos a cubrir
- list: la secuencia mutable universal
- dict: mapa clave-valor con inserción ordenada
- tuple: secuencia inmutable y su uso como valor
- set: colección sin duplicados y operaciones de conjunto
- Colecciones especializadas: deque, Counter, defaultdict, namedtuple

---

## El Concepto

### list — La Secuencia Mutable

Las listas son arrays dinámicos: almacenan elementos en orden, permiten duplicados y se pueden modificar.

```python
# Creación
numeros = [1, 2, 3, 4, 5]
mixta   = [1, "hola", True, None]   # puede mezclar tipos (aunque raro en código tipado)
vacia   = []

# Acceso — O(1)
print(numeros[0])    # 1
print(numeros[-1])   # 5 — último elemento
print(numeros[1:3])  # [2, 3] — slicing [inicio:fin] (fin excluido)
print(numeros[::2])  # [1, 3, 5] — cada 2 elementos
print(numeros[::-1]) # [5, 4, 3, 2, 1] — reverso

# Modificación
numeros.append(6)        # añadir al final — O(1) amortizado
numeros.insert(0, 0)     # insertar en posición — O(n)
numeros.pop()            # eliminar y retornar el último — O(1)
numeros.pop(0)           # eliminar por índice — O(n)
numeros.remove(3)        # eliminar primer elemento con ese valor — O(n)
numeros.extend([7, 8])   # añadir múltiples — O(k)

# Ordenamiento
numeros.sort()                    # in-place
numeros.sort(reverse=True)        # descendente
ordenado = sorted(numeros)        # retorna nueva lista, no modifica
ordenado = sorted(numeros, key=lambda x: -x)  # con key personalizada

# Búsqueda
print(3 in numeros)        # True — O(n)
print(numeros.index(3))    # índice del primer 3 — O(n)
print(numeros.count(3))    # cuántas veces aparece — O(n)

# List comprehensions — la forma Python de transformar listas
cuadrados = [x**2 for x in range(10)]
pares     = [x for x in range(20) if x % 2 == 0]
matriz    = [[i*j for j in range(3)] for i in range(3)]

# Desempaquetado (unpacking)
primero, *resto = [1, 2, 3, 4]   # primero=1, resto=[2, 3, 4]
a, b, *_, z = [1, 2, 3, 4, 5]   # a=1, b=2, z=5, _ descartado
```

**Complejidades clave:** append O(1), pop() O(1), insert(0) O(n), in O(n), indexación O(1).

### dict — El Mapa Clave-Valor

Los dicts son tablas hash. Desde Python 3.7, mantienen el orden de inserción. Las claves deben ser hashables (inmutables).

```python
# Creación
persona = {"nombre": "Andrés", "edad": 30}
vacio   = {}
desde_pares = dict([("a", 1), ("b", 2)])

# Acceso
print(persona["nombre"])           # "Andrés" — KeyError si no existe
print(persona.get("email"))        # None — no lanza error
print(persona.get("email", "N/A")) # "N/A" — valor por defecto

# Modificación
persona["email"] = "a@b.com"   # añadir o actualizar — O(1) amortizado
persona.update({"ciudad": "Madrid", "edad": 31})  # actualizar múltiples
del persona["email"]           # eliminar — O(1)
email = persona.pop("email", None)  # eliminar y retornar — O(1)

# Iteración
for clave in persona:                    # itera sobre claves
    print(clave)

for valor in persona.values():           # itera sobre valores
    print(valor)

for clave, valor in persona.items():     # itera sobre pares — lo más común
    print(f"{clave}: {valor}")

# Verificación
print("nombre" in persona)    # True — O(1) — busca en claves
print("Andrés" in persona.values())  # True — O(n) — busca en valores

# Dict comprehensions
cuadrados = {x: x**2 for x in range(5)}  # {0:0, 1:1, 2:4, 3:9, 4:16}
filtrado  = {k: v for k, v in persona.items() if v is not None}

# Fusión de dicts (Python 3.9+)
a = {"x": 1}
b = {"y": 2}
combinado = a | b        # nuevo dict fusionado
a |= b                   # actualiza a in-place

# Desempaquetado (Python 3.5+)
combinado = {**a, **b}   # equivalente a a | b
```

**Complejidades clave:** get/set/delete O(1) amortizado, `in` sobre claves O(1).

### tuple — Secuencia Inmutable

Las tuplas son como listas pero inmutables. Esto las hace:
- Hashables (si sus elementos lo son) → pueden ser claves de dict
- Ligeras en memoria y más rápidas de iterar
- Semánticamente distintas: una lista es una colección de elementos similares; una tuple es un registro con campos de significado específico

```python
# Creación
coordenada = (10, 20)
punto_3d   = (1.0, 2.0, 3.0)
unitaria   = (42,)      # tupla de un elemento — la coma es obligatoria
sin_paren  = 1, 2, 3   # el paréntesis es opcional

# Las tuplas son comparables y hashables
puntos = {(0, 0): "origen", (1, 0): "derecha"}  # como clave de dict

# Desempaquetado
x, y = coordenada
latitud, longitud, altitud = punto_3d

# Swap sin variable temporal (usando tuple packing/unpacking)
a, b = 1, 2
a, b = b, a   # a=2, b=1

# Namedtuple — tupla con campos nombrados
from collections import namedtuple
Punto = namedtuple("Punto", ["x", "y"])
p = Punto(x=10, y=20)
print(p.x)    # 10 — acceso por nombre
print(p[0])   # 10 — acceso por índice también funciona

# typing.NamedTuple — versión con type hints (preferida)
from typing import NamedTuple

class Coordenada(NamedTuple):
    latitud: float
    longitud: float
    altitud: float = 0.0  # valor por defecto

c = Coordenada(40.416775, -3.703790)
print(c.latitud)   # 40.416775
```

### set — Colección sin Duplicados

Los sets son colecciones desordenadas de elementos únicos hashables. Son tablas hash como los dicts pero solo con claves.

```python
# Creación
frutas = {"manzana", "pera", "naranja"}
letras = set("abracadabra")  # {'a', 'b', 'r', 'c', 'd'} — elimina duplicados
vacio  = set()               # {} crea dict vacío, no set

# Operaciones de conjunto
a = {1, 2, 3, 4}
b = {3, 4, 5, 6}

print(a | b)    # {1, 2, 3, 4, 5, 6} — unión
print(a & b)    # {3, 4} — intersección
print(a - b)    # {1, 2} — diferencia (en a pero no en b)
print(a ^ b)    # {1, 2, 5, 6} — diferencia simétrica (en uno pero no en ambos)

# Verificación de subconjunto
print({1, 2}.issubset({1, 2, 3}))     # True
print({1, 2, 3}.issuperset({1, 2}))   # True

# Modificación
frutas.add("mango")
frutas.discard("kiwi")    # no lanza error si no existe
frutas.remove("pera")     # lanza KeyError si no existe

# Caso de uso principal: eliminar duplicados con orden preservado
lista_con_dup = [3, 1, 4, 1, 5, 9, 2, 6, 5]
sin_dup = list(dict.fromkeys(lista_con_dup))  # [3, 1, 4, 5, 9, 2, 6] preserva orden
```

**Complejidades clave:** add/remove/in O(1) amortizado.

### Colecciones Especializadas

```python
from collections import deque, Counter, defaultdict

# deque — cola de doble extremo, O(1) en ambos extremos
cola = deque([1, 2, 3])
cola.appendleft(0)   # O(1) — insertar al inicio
cola.popleft()       # O(1) — extraer del inicio
cola.append(4)       # O(1) — añadir al final
cola = deque(maxlen=3)  # capacidad limitada, descarta el más antiguo
cola.extend([1, 2, 3, 4])  # deque([2, 3, 4]) — descartó el 1

# Counter — conteo de elementos
from collections import Counter
texto = "mississippi"
c = Counter(texto)
print(c)                     # Counter({'i': 4, 's': 4, 'p': 2, 'm': 1})
print(c.most_common(2))      # [('i', 4), ('s', 4)]
print(c['s'])                # 4
print(c['z'])                # 0 — no lanza KeyError

# Aritmética de counters
c1 = Counter("aab")
c2 = Counter("abc")
print(c1 + c2)    # Counter({'a': 3, 'b': 2, 'c': 1})
print(c1 - c2)    # Counter({'a': 1}) — solo positivos

# defaultdict — dict con valor por defecto para claves nuevas
from collections import defaultdict

# Agrupar elementos sin verificar si la clave existe
grupos = defaultdict(list)
for item in [("a", 1), ("b", 2), ("a", 3), ("b", 4)]:
    grupos[item[0]].append(item[1])
print(dict(grupos))  # {'a': [1, 3], 'b': [2, 4]}

# Contador sin Counter
conteo = defaultdict(int)
for letra in "mississippi":
    conteo[letra] += 1
```

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - **list**: secuencia mutable ordenada. `append` O(1), `insert(0)` O(n), `in` O(n). Usa comprehensions.
> - **dict**: mapa clave-valor, O(1) en acceso/inserción. `in` sobre claves es O(1), sobre values O(n). Mantiene orden de inserción desde 3.7.
> - **tuple**: inmutable, hashable, puede ser clave de dict. Usa `NamedTuple` cuando los campos tienen nombres.
> - **set**: sin duplicados, sin orden, O(1) para `in`. Ideal para eliminar duplicados y operaciones de conjunto.
> - **deque**: cuando necesitas O(1) en ambos extremos (no listas). **Counter**: para contar. **defaultdict**: para agrupar sin `if key in dict`.

### Para llevar a la práctica
- [ ] Dado una lista de palabras, usa `Counter` para encontrar las 3 más frecuentes
- [ ] Implementa un sistema de caché LRU con `deque(maxlen=n)` y un `dict`
- [ ] Usa `defaultdict(list)` para agrupar objetos por una propiedad
- [ ] Explica por qué `{} ` crea un dict y `set()` crea un set

### Recursos
- 🌐 docs.python.org/3/library/collections.html — colecciones estándar completas
- 📖 *Fluent Python* — Capítulo 2 (arrays y sequences) y 3 (dicts y sets)
- 🌐 docs.python.org/3/library/stdtypes.html#mapping-types-dict

---
`#python` `#estructuras-de-datos` `#lista` `#dict` `#set`
