# 🐍 05 — Programación Orientada a Objetos

[[Desarrollo Profesional/Python/Python|⬅️ Volver al índice]] | [[Desarrollo Profesional/Python/04 - Funciones|← 04]] | [[Desarrollo Profesional/Python/06 - Módulos y Paquetes|06 →]]

> [!abstract] Introducción
> Python tiene un sistema de orientación a objetos completo y flexible. A diferencia de Java o C++, todo en Python es un objeto — incluyendo las clases mismas. Esto permite metaclases, decoradores de clase, y el potente protocolo de dunder methods que hace que los objetos customizados se comporten como tipos nativos del lenguaje.

## ¿De qué vamos a hablar?

POO en Python desde la sintaxis básica hasta los mecanismos avanzados que lo distinguen: el Method Resolution Order en herencia múltiple, los dunder methods, las properties y los dataclasses modernos.

### Conceptos que vamos a cubrir
- Clases, instancias, y métodos (instance, class, static)
- Herencia simple y múltiple — MRO
- Dunder methods: `__init__`, `__repr__`, `__str__`, `__eq__`, y más
- Properties: getters y setters idiomáticos
- Dataclasses: POO sin boilerplate
- Clases abstractas (ABC)

---

## El Concepto

### Clases Básicas

```python
class Vehiculo:
    # Atributo de clase — compartido por todas las instancias
    ruedas: int = 4

    def __init__(self, marca: str, modelo: str, año: int) -> None:
        # Atributos de instancia — únicos para cada objeto
        self.marca  = marca
        self.modelo = modelo
        self.año    = año
        self._kilometros = 0  # convención: _ prefijo = "privado" (no forzado)

    # Método de instancia — recibe self
    def conducir(self, km: int) -> None:
        self._kilometros += km
        print(f"Conduciendo {km} km. Total: {self._kilometros} km")

    # Método de clase — recibe cls (accede a la clase, no a la instancia)
    @classmethod
    def desde_descripcion(cls, descripcion: str) -> "Vehiculo":
        """Factory method — crea instancia desde string 'Marca Modelo Año'"""
        partes = descripcion.split()
        return cls(partes[0], partes[1], int(partes[2]))

    # Método estático — no recibe self ni cls (es una función en el namespace)
    @staticmethod
    def es_año_valido(año: int) -> bool:
        return 1886 <= año <= 2030  # primer automóvil: 1886

    def __repr__(self) -> str:
        return f"Vehiculo(marca={self.marca!r}, modelo={self.modelo!r}, año={self.año})"

v = Vehiculo("Toyota", "Corolla", 2022)
v2 = Vehiculo.desde_descripcion("Honda Civic 2021")
```

### Herencia y MRO

```python
class Animal:
    def __init__(self, nombre: str) -> None:
        self.nombre = nombre

    def hablar(self) -> str:
        return f"{self.nombre} hace un sonido"

class Mascota:
    def __init__(self, dueño: str) -> None:
        self.dueño = dueño

    def descripcion(self) -> str:
        return f"Mascota de {self.dueño}"

# Herencia múltiple — Python usa el algoritmo C3 para el MRO
class Perro(Animal, Mascota):
    def __init__(self, nombre: str, dueño: str, raza: str) -> None:
        # super() en herencia múltiple sigue el MRO
        Animal.__init__(self, nombre)
        Mascota.__init__(self, dueño)
        self.raza = raza

    def hablar(self) -> str:
        return f"{self.nombre} ladra: ¡Guau!"

# Ver el MRO (Method Resolution Order)
print(Perro.__mro__)
# (<class 'Perro'>, <class 'Animal'>, <class 'Mascota'>, <class 'object'>)

p = Perro("Rex", "Andrés", "Pastor Alemán")
p.hablar()       # "Rex ladra: ¡Guau!" — usa el método de Perro
p.descripcion()  # "Mascota de Andrés" — hereda de Mascota

# super() con herencia múltiple — delega al siguiente en el MRO
class A:
    def metodo(self) -> str:
        return "A"

class B(A):
    def metodo(self) -> str:
        return "B → " + super().metodo()

class C(A):
    def metodo(self) -> str:
        return "C → " + super().metodo()

class D(B, C):
    def metodo(self) -> str:
        return "D → " + super().metodo()

print(D().metodo())  # "D → B → C → A"
# MRO de D: D → B → C → A → object
```

### Dunder Methods — El Protocolo de Objetos

Los dunder methods (double underscore) permiten que los objetos se integren con la sintaxis y las funciones built-in de Python:

```python
from __future__ import annotations
from typing import Any

class Vector:
    def __init__(self, x: float, y: float) -> None:
        self.x = x
        self.y = y

    # Representación para debugging — eval(repr(v)) debería recrear el objeto
    def __repr__(self) -> str:
        return f"Vector({self.x}, {self.y})"

    # Representación para usuarios — str(v)
    def __str__(self) -> str:
        return f"({self.x}, {self.y})"

    # Aritmética
    def __add__(self, otro: Vector) -> Vector:
        return Vector(self.x + otro.x, self.y + otro.y)

    def __mul__(self, escalar: float) -> Vector:
        return Vector(self.x * escalar, self.y * escalar)

    def __rmul__(self, escalar: float) -> Vector:
        return self.__mul__(escalar)  # permite 3 * vector (no solo vector * 3)

    # Comparación
    def __eq__(self, otro: Any) -> bool:
        if not isinstance(otro, Vector):
            return NotImplemented
        return self.x == otro.x and self.y == otro.y

    def __hash__(self) -> int:
        return hash((self.x, self.y))  # si defines __eq__, debes definir __hash__

    # Longitud — len(v)
    def __len__(self) -> int:
        return 2  # siempre tiene x e y

    # Acceso por índice — v[0], v[1]
    def __getitem__(self, idx: int) -> float:
        if idx == 0: return self.x
        if idx == 1: return self.y
        raise IndexError(f"Índice {idx} fuera de rango")

    # Context manager — with v:
    def __enter__(self) -> Vector:
        return self

    def __exit__(self, *args: Any) -> None:
        pass

    # Valor booleano — bool(v)
    def __bool__(self) -> bool:
        return self.x != 0 or self.y != 0

v1 = Vector(1, 2)
v2 = Vector(3, 4)
print(v1 + v2)       # Vector(4, 6)
print(v1 * 3)        # Vector(3, 6)
print(3 * v1)        # Vector(3, 6) — usa __rmul__
print(v1 == v1)      # True
print(bool(Vector(0, 0)))  # False
```

### Properties — Getters y Setters Idiomáticos

```python
class Temperatura:
    def __init__(self, celsius: float) -> None:
        self._celsius = celsius  # almacenamos en Celsius

    @property
    def celsius(self) -> float:
        return self._celsius

    @celsius.setter
    def celsius(self, valor: float) -> None:
        if valor < -273.15:
            raise ValueError("Temperatura inferior al cero absoluto")
        self._celsius = valor

    @property
    def fahrenheit(self) -> float:
        return self._celsius * 9/5 + 32

    @fahrenheit.setter
    def fahrenheit(self, valor: float) -> None:
        self.celsius = (valor - 32) * 5/9

    @property
    def kelvin(self) -> float:
        return self._celsius + 273.15

t = Temperatura(100)
print(t.celsius)      # 100.0
print(t.fahrenheit)   # 212.0
print(t.kelvin)       # 373.15

t.fahrenheit = 68     # usa el setter de fahrenheit
print(t.celsius)      # 20.0
```

### Dataclasses — POO sin Boilerplate

Las dataclasses (Python 3.7+) generan automáticamente `__init__`, `__repr__`, `__eq__` y más:

```python
from dataclasses import dataclass, field
from typing import ClassVar

@dataclass
class Producto:
    nombre: str
    precio: float
    stock: int = 0
    categorias: list[str] = field(default_factory=list)  # mutable default
    _id: int = field(default=0, repr=False)  # no aparece en repr

    # Atributo de clase — no forma parte de __init__
    IVA: ClassVar[float] = 0.21

    def precio_con_iva(self) -> float:
        return self.precio * (1 + self.IVA)

    # __post_init__ se llama después del __init__ generado
    def __post_init__(self) -> None:
        if self.precio < 0:
            raise ValueError("El precio no puede ser negativo")

@dataclass(frozen=True)  # inmutable — genera __hash__
class Punto:
    x: float
    y: float

@dataclass(order=True)  # genera __lt__, __le__, __gt__, __ge__
class Empleado:
    salario: float
    nombre: str = field(compare=False)  # excluir de comparaciones

p = Producto("Teclado", 89.99, 10, ["periféricos"])
print(p)   # Producto(nombre='Teclado', precio=89.99, stock=10, categorias=['periféricos'])
print(p.precio_con_iva())  # 108.88...
```

### Clases Abstractas (ABC)

```python
from abc import ABC, abstractmethod

class Forma(ABC):
    @abstractmethod
    def area(self) -> float:
        """Retorna el área de la forma"""
        ...

    @abstractmethod
    def perimetro(self) -> float:
        """Retorna el perímetro de la forma"""
        ...

    # Método concreto — disponible en todas las subclases
    def descripcion(self) -> str:
        return f"{self.__class__.__name__}: área={self.area():.2f}, perímetro={self.perimetro():.2f}"

class Circulo(Forma):
    def __init__(self, radio: float) -> None:
        self.radio = radio

    def area(self) -> float:
        return 3.14159 * self.radio ** 2

    def perimetro(self) -> float:
        return 2 * 3.14159 * self.radio

# Forma()  # TypeError: Can't instantiate abstract class
c = Circulo(5)
print(c.descripcion())  # "Circulo: área=78.54, perímetro=31.42"
```

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - Los métodos de instancia reciben `self`, los de clase `cls` (`@classmethod`), y los estáticos ninguno (`@staticmethod`).
> - En herencia múltiple, Python usa el algoritmo C3 para el MRO. `super()` sigue el MRO, no apunta directamente al padre.
> - Los dunder methods integran los objetos con la sintaxis de Python: `+`, `[]`, `len()`, `bool()`, `with`. Define `__repr__` siempre, `__str__` cuando la representación de usuario es diferente.
> - `@property` es el patrón idiomático para getters/setters en Python — no la convención `getX()/setX()`.
> - Las `@dataclass` generan `__init__`, `__repr__`, `__eq__` automáticamente. Son el estándar moderno para clases de datos.

### Para llevar a la práctica
- [ ] Crea una clase `Dinero` con dunder methods para `+`, `-`, `*`, `==`, `<` y el formateo correcto de decimales
- [ ] Convierte una clase existente a `@dataclass` y observa el código que se elimina
- [ ] Implementa una jerarquía de formas geométricas con ABC: `Forma` → `Poligono` → `Triangulo`, `Rectangulo`

### Recursos
- 📖 *Fluent Python* — Capítulo 11 (interfaces) y 16-17 (operadores y dunder methods)
- 🌐 docs.python.org/3/library/dataclasses.html
- 🌐 docs.python.org/3/library/abc.html
- 📄 PEP 557 — Data Classes

---
`#python` `#poo` `#clases` `#dunder` `#dataclasses`
