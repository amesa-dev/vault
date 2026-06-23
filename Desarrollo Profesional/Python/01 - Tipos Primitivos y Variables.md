# 🐍 01 — Tipos Primitivos y Variables

[[Desarrollo Profesional/Python/Python|⬅️ Volver al índice]] | [[Desarrollo Profesional/Python/00 - Introducción y Setup|← 00]] | [[Desarrollo Profesional/Python/02 - Estructuras de Datos|02 →]]

> [!abstract] Introducción
> Python es dinámicamente tipado — las variables no tienen tipo, los valores sí. Esto significa que una variable puede apuntar a un entero y luego a una cadena sin error. Sin embargo, los tipos existen, son importantes, y entenderlos evita errores sutiles especialmente al trabajar con conversiones y comparaciones.

## ¿De qué vamos a hablar?

Los tipos primitivos de Python y cómo trabajan internamente. Cuándo usar cada uno, cómo convertir entre ellos, y los errores más comunes que comete alguien que viene de otros lenguajes.

### Conceptos que vamos a cubrir
- int, float, str, bool, None y sus particularidades
- Sistema de referencias y mutabilidad
- Operadores y coerciones
- f-strings y formateo moderno
- Type hints con primitivos

---

## El Concepto

### int — Enteros

Python no tiene límite de tamaño para enteros. Un `int` puede ser arbitrariamente grande (a diferencia de C o Java con sus límites de 32/64 bits).

```python
x = 42
y = -17
z = 10_000_000    # guiones bajos como separadores visuales

# Bases numéricas
binario = 0b1010   # 10 en decimal
octal   = 0o17     # 15 en decimal
hexa    = 0xFF     # 255 en decimal

# Operaciones
print(7 // 2)   # 3  — división entera (floor division)
print(7 % 2)    # 1  — módulo
print(2 ** 10)  # 1024 — exponenciación
print(7 / 2)    # 3.5 — división siempre retorna float
```

> `7 / 2` retorna `3.5`, no `3`. Para división entera usa `//`.

### float — Punto Flotante

Los floats en Python son IEEE 754 de doble precisión (64 bits). Esto significa que hay errores de representación inherentes.

```python
x = 3.14
y = 1.5e3     # 1500.0 — notación científica
z = float('inf')  # infinito

# El error clásico de punto flotante
print(0.1 + 0.2)          # 0.30000000000000004 — no es un bug de Python
print(0.1 + 0.2 == 0.3)   # False — nunca compares floats con ==

# La forma correcta: usar math.isclose
import math
print(math.isclose(0.1 + 0.2, 0.3))  # True

# Para aritmética decimal exacta (dinero, contabilidad)
from decimal import Decimal
precio = Decimal('10.50')
iva    = Decimal('0.21')
total  = precio * (1 + iva)  # aritmética exacta
```

### str — Cadenas

Las cadenas en Python son inmutables, secuencias de Unicode. No hay tipo `char` — un carácter es simplemente una cadena de longitud 1.

```python
# Comillas simples o dobles — equivalentes
nombre = 'Andrés'
apellido = "Mesa"

# Multilínea
texto = """
Primera línea
Segunda línea
"""

# Operaciones básicas
print(len(nombre))           # 6
print(nombre[0])             # 'A'
print(nombre[-1])            # 's' — índice negativo desde el final
print(nombre[1:4])           # 'ndr' — slicing

# Métodos útiles
print("hola".upper())        # 'HOLA'
print("  hola  ".strip())    # 'hola'
print("a,b,c".split(","))    # ['a', 'b', 'c']
print(",".join(["a", "b"]))  # 'a,b'
print("hola".startswith("ho"))  # True
print("hola".replace("l", "r")) # 'hora'

# f-strings (Python 3.6+) — la forma moderna de formatear
edad = 30
print(f"Tengo {edad} años")                # 'Tengo 30 años'
print(f"El doble es {edad * 2}")           # 'El doble es 60'
print(f"{3.14159:.2f}")                    # '3.14' — formato
print(f"{10000:,}")                        # '10,000' — separadores
print(f"{nombre!r}")                       # repr() en lugar de str()

# Python 3.12+: f-strings sin restricciones
dato = {'clave': 'valor'}
print(f"Valor: {dato['clave']}")           # antes no se podía usar la misma comilla
```

### bool — Booleanos

`bool` es una subclase de `int` en Python. `True` es literalmente `1` y `False` es `0`.

```python
print(True + True)    # 2
print(True * 5)       # 5
print(type(True))     # <class 'bool'>

# Valores "falsy" — evaluados como False en contexto booleano
falsy = [False, None, 0, 0.0, "", [], {}, set(), ()]

# Valores "truthy" — todo lo demás
truthy = [True, 1, -1, "a", [0], {"x": 1}]

# Uso práctico de truthy/falsy
lista = []
if lista:           # más idiomático que if len(lista) > 0
    print("tiene elementos")
else:
    print("vacía")

# Operadores booleanos retornan el valor, no True/False
print(0 or "default")   # "default" — retorna el primer truthy
print(1 and "ok")       # "ok" — retorna el último si todos son truthy
print(None or 0 or [])  # [] — retorna el último si todos son falsy

# Operador walrus := (Python 3.8+) — asigna y evalúa
import re
texto = "Hola mundo"
if m := re.search(r"\w+", texto):
    print(m.group())  # 'Hola'
```

### None — El Valor Nulo

`None` es un singleton — solo hay una instancia. Siempre compara con `is`, nunca con `==`.

```python
x = None
print(x is None)    # True — forma correcta
print(x == None)    # True — funciona pero es incorrecto por estilo

# Patrón común: valor por defecto opcional
def conectar(host: str, puerto: int | None = None) -> None:
    puerto = puerto or 5432  # si puerto es None, usa 5432
```

### Sistema de Referencias

Python no tiene variables en el sentido de "cajas que contienen valores". Tiene **referencias** a objetos en memoria.

```python
a = [1, 2, 3]
b = a           # b apunta al MISMO objeto que a
b.append(4)
print(a)        # [1, 2, 3, 4] — a también cambió

# Para copiar, usa copy()
import copy
a = [1, 2, 3]
b = a.copy()    # copia superficial
b.append(4)
print(a)        # [1, 2, 3] — a no cambió

# Para objetos anidados, copy profunda
b = copy.deepcopy(a)
```

Los tipos inmutables (int, str, tuple, bool) no tienen este problema porque no se pueden modificar.

### Type Hints con Primitivos

```python
# Python 3.10+ — union con |
def procesar(valor: int | float) -> str:
    return str(valor)

# Python 3.9+ — tipos built-in sin importar typing
def filtrar(items: list[str]) -> list[str]:
    return [i for i in items if i]

# Optional[X] equivale a X | None
from typing import Optional
def buscar(id: int) -> Optional[str]:  # puede retornar str o None
    pass

# Python 3.10+ más idiomático
def buscar(id: int) -> str | None:
    pass
```

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - `int` no tiene límite de tamaño. `7 / 2` siempre retorna float; usa `//` para división entera.
> - Los floats tienen errores de representación. Nunca los compares con `==` — usa `math.isclose()`. Para dinero, usa `Decimal`.
> - Las cadenas son inmutables y Unicode. Los f-strings (`f"{valor}"`) son la forma moderna de formatear.
> - `True` es `1` y `False` es `0` — son subclases de int. Los valores "falsy" son `None, 0, "", [], {}, set()`.
> - Python maneja referencias, no valores. Con tipos mutables (list, dict), asignar `b = a` no copia — ambas variables apuntan al mismo objeto.

### Para llevar a la práctica
- [ ] Verifica que `0.1 + 0.2 == 0.3` es `False` en tu intérprete y explica por qué
- [ ] Escribe una función con type hints que acepte `int | None` y retorne `str`
- [ ] Practica la diferencia entre `a = b` y `a = b.copy()` con una lista

### Recursos
- 📖 *Fluent Python* — Capítulo 2 (modelo de datos de Python)
- 🌐 docs.python.org/3/library/stdtypes.html — referencia oficial de tipos built-in
- 🌐 docs.python.org/3/library/decimal.html — Decimal para aritmética exacta

---
`#python` `#tipos` `#variables` `#fundamentos`
