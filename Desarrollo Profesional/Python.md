# 🐍 Python

[[Desarrollo Profesional/Inicio Profesional|⬅️ Volver a Desarrollo Profesional]]

> [!abstract] Python
> Python es un lenguaje de programación de alto nivel, interpretado, con una filosofía que enfatiza la legibilidad del código. Es ampliamente utilizado en desarrollo web, scripting, automatización y ciencia de datos.

---

## 🔑 Estructuras de Datos Avanzadas

### 1. List Comprehensions (Comprensión de Listas)
Una forma concisa de crear listas.
```python
# Crear una lista de cuadrados para números pares
cuadrados = [x**2 for x in range(10) if x % 2 == 0]
# Resultado: [0, 4, 16, 36, 64]
```

### 2. Diccionarios y Sets Comprehensions
```python
# Comprensión de diccionario
mi_dict = {x: x**2 for x in range(5)}
# Resultado: {0: 0, 1: 1, 2: 4, 3: 9, 4: 16}
```

---

## 🎨 Programación Orientada a Objetos (POO)
Sintaxis básica de clases, constructor y métodos.

```python
class Personaje:
    def __init__(self, nombre: str, nivel: int = 1):
        self.nombre = nombre
        self.nivel = nivel

    def subir_nivel(self):
        self.nivel += 1
        print(f"¡{self.nombre} ha subido al nivel {self.nivel}!")

# Instanciación
heroe = Personaje("Aventurero")
heroe.subir_nivel()
```

---

## ⚡ Decoradores
Los decoradores permiten modificar o extender el comportamiento de una función o método sin cambiar directamente su código.

```python
def mi_decorador(func):
    def envoltura(*args, **kwargs):
        print("Antes de ejecutar la función...")
        resultado = func(*args, **kwargs)
        print("Después de ejecutar la función.")
        return resultado
    return envoltura

@mi_decorador
def saludar(nombre):
    print(f"Hola, {nombre}!")

saludar("Andrés")
```

---

## 🏷️ Tipado de Datos (Type Hinting)
Python es dinámico, pero soporta anotaciones de tipo estático para herramientas de análisis como `mypy`.

```python
from typing import List, Dict, Optional

def procesar_usuarios(usuarios: List[str]) -> Optional[Dict[str, int]]:
    if not usuarios:
        return None
    return {usuario: len(usuario) for usuario in usuarios}
```

---
`#python` `#programacion` `#backend` `#apuntes`
