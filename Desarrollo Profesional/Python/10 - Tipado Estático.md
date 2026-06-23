# 🐍 10 — Tipado Estático

[[Desarrollo Profesional/Python/Python|⬅️ Volver al índice]] | [[Desarrollo Profesional/Python/09 - Decoradores|← 09]] | [[Desarrollo Profesional/Python/11 - Concurrencia y Asyncio|11 →]]

> [!abstract] Introducción
> Python es dinámicamente tipado, pero desde Python 3.5 tiene soporte para type hints y desde entonces el ecosistema de tipado estático ha madurado enormemente. Mypy, pyright y ruff pueden verificar tipos estáticamente — no en runtime, sino en tiempo de desarrollo. El objetivo no es convertir Python en Java: es añadir una capa de documentación ejecutable que el IDE y las herramientas de CI pueden verificar automáticamente.

## ¿De qué vamos a hablar?

El sistema de tipado estático de Python desde los tipos básicos hasta los mecanismos avanzados: `Protocol` para duck typing tipado, `TypeVar` y genéricos, `TypedDict` para dicts con forma conocida, y `Literal` para valores exactos.

### Conceptos que vamos a cubrir
- Type hints modernos: la sintaxis actual
- `Optional`, `Union`, `Any` — y cuándo evitar cada uno
- `Protocol` — duck typing con verificación estática
- `TypeVar` y funciones genéricas
- `TypedDict`, `NamedTuple`, `Literal`
- `mypy` y `pyright`: configuración y uso

---

## El Concepto

### Sintaxis Moderna de Type Hints

```python
# Python 3.10+ — la sintaxis más limpia
def procesar(
    nombre: str,
    edad: int,
    email: str | None = None,      # union con | en lugar de Union[]
    etiquetas: list[str] = [],     # tipos genéricos sin importar
) -> dict[str, str | int]:
    return {"nombre": nombre, "edad": edad}

# Python 3.9 — tipos built-in sin typing (list, dict, tuple, set...)
def filtrar(items: list[int]) -> list[int]:
    return [x for x in items if x > 0]

# Python 3.8 y anterior — necesitas importar de typing
from typing import List, Dict, Optional, Union, Tuple
def antigua(items: List[int]) -> Optional[Dict[str, int]]:
    pass

# Anotaciones de variables
nombre: str = "Andrés"
activo: bool = True
datos: dict[str, list[int]] = {}

# Final — valor que no cambia después de la asignación
from typing import Final
MAX_INTENTOS: Final = 3
URL_BASE: Final[str] = "https://api.ejemplo.com"
```

### `Optional` y `Union`

```python
from typing import Optional

# Optional[X] es azúcar para X | None
def buscar_usuario(id: int) -> Optional[dict]:  # puede retornar None
    return db.find(id) or None

# Python 3.10+
def buscar_usuario(id: int) -> dict | None:
    pass

# Union — cuando puede ser más de dos tipos
from typing import Union
def serializar(valor: Union[str, int, bool, None]) -> str:
    return str(valor)

# Python 3.10+
def serializar(valor: str | int | bool | None) -> str:
    return str(valor)

# Any — el tipo que desactiva las verificaciones
from typing import Any
def procesar_cualquiera(datos: Any) -> Any:  # evitar cuando sea posible
    return datos
```

### `Protocol` — Duck Typing con Verificación Estática

`Protocol` permite el duck typing estructural verificado estáticamente — la forma pythónica de decir "cualquier objeto que tenga estos métodos":

```python
from typing import Protocol, runtime_checkable

class Serializable(Protocol):
    def serializar(self) -> str: ...
    def deserializar(self, datos: str) -> None: ...

class Comparable(Protocol):
    def __lt__(self, otro: "Comparable") -> bool: ...
    def __eq__(self, otro: object) -> bool: ...

# Una clase que cumple el protocolo SIN declararlo explícitamente
class Producto:
    def __init__(self, nombre: str, precio: float) -> None:
        self.nombre = nombre
        self.precio = precio

    def serializar(self) -> str:
        return f"{self.nombre}:{self.precio}"

    def deserializar(self, datos: str) -> None:
        self.nombre, precio_str = datos.split(":")
        self.precio = float(precio_str)

# Función que acepta cualquier cosa que cumpla Serializable
def guardar(obj: Serializable, archivo: str) -> None:
    with open(archivo, "w") as f:
        f.write(obj.serializar())

guardar(Producto("Teclado", 89.99), "producto.txt")  # ✅ — Producto cumple Serializable

# @runtime_checkable — permite isinstance() con el protocolo
@runtime_checkable
class Closeable(Protocol):
    def close(self) -> None: ...

print(isinstance(open("archivo.txt"), Closeable))  # True
```

### `TypeVar` — Genéricos en Python

```python
from typing import TypeVar, Generic, Sequence

T = TypeVar("T")
KT = TypeVar("KT")
VT = TypeVar("VT")
Comparable = TypeVar("Comparable", bound="SupportsLessThan")

# Función genérica
def primero(secuencia: Sequence[T]) -> T | None:
    return secuencia[0] if secuencia else None

primero([1, 2, 3])       # TypeVar T = int → retorna int | None
primero(["a", "b"])      # TypeVar T = str → retorna str | None

# Con bound — restringe qué tipos puede ser T
from typing import Protocol

class SupportsLessThan(Protocol):
    def __lt__(self, otro: object) -> bool: ...

S = TypeVar("S", bound=SupportsLessThan)

def minimo(secuencia: Sequence[S]) -> S:
    return min(secuencia)

minimo([3, 1, 4, 1, 5])   # ✅ — int tiene __lt__
minimo(["c", "a", "b"])   # ✅ — str tiene __lt__

# Clase genérica
class Pila(Generic[T]):
    def __init__(self) -> None:
        self._items: list[T] = []

    def push(self, item: T) -> None:
        self._items.append(item)

    def pop(self) -> T:
        if not self._items:
            raise IndexError("Pila vacía")
        return self._items.pop()

    def peek(self) -> T | None:
        return self._items[-1] if self._items else None

pila_int: Pila[int] = Pila()
pila_int.push(1)
pila_int.push(2)
n = pila_int.pop()  # mypy sabe que n es int
```

### `TypedDict`, `NamedTuple`, `Literal`

```python
from typing import TypedDict, Required, NotRequired

# TypedDict — dict con forma conocida y verificada
class UsuarioDict(TypedDict):
    id: int
    nombre: str
    email: str
    edad: NotRequired[int]  # campo opcional (Python 3.11+)

# Python 3.11+: Required/NotRequired en campos individuales
class ConfigDict(TypedDict, total=False):  # total=False hace todos opcionales
    host: Required[str]  # este sí es requerido
    puerto: int
    ssl: bool

usuario: UsuarioDict = {"id": 1, "nombre": "Andrés", "email": "a@b.com"}

# Literal — valores exactos
from typing import Literal

def cambiar_estado(estado: Literal["activo", "inactivo", "pendiente"]) -> None:
    print(f"Estado: {estado}")

cambiar_estado("activo")    # ✅
cambiar_estado("borrado")   # Error: no es uno de los valores válidos

# Combinando Literal con Union
HttpMethod = Literal["GET", "POST", "PUT", "DELETE", "PATCH"]
LogLevel   = Literal["DEBUG", "INFO", "WARNING", "ERROR", "CRITICAL"]

def log(nivel: LogLevel, mensaje: str) -> None:
    print(f"[{nivel}] {mensaje}")

# TypeAlias (Python 3.10+) — alias de tipo explícito
type Vector = list[float]         # Python 3.12+ con keyword type
from typing import TypeAlias       # Python 3.10-3.11
Vector: TypeAlias = list[float]
```

### Mypy — Verificación Estática

```bash
# Instalación
pip install mypy

# Verificar un archivo o proyecto
mypy src/
mypy --strict src/  # modo estricto — el más exigente

# Configuración en pyproject.toml
[tool.mypy]
python_version = "3.12"
strict = true
warn_return_any = true
warn_unused_configs = true
ignore_missing_imports = false  # falla si no hay tipos para una librería
```

```python
# Comentarios de tipo para casos especiales
variable = []          # type: list[int] — forma antigua, evitar en código nuevo
variable: list[int] = []  # forma moderna

# Ignorar líneas específicas
resultado = funcion()  # type: ignore[no-any-return]
resultado = funcion()  # type: ignore  # ignora cualquier error en esta línea

# reveal_type — mypy muestra el tipo inferido (solo para debugging)
reveal_type(resultado)  # mypy dice: Revealed type is "builtins.int"
```

### `ParamSpec` y `Concatenate` — Tipado de Decoradores

```python
from typing import ParamSpec, Callable, TypeVar

P = ParamSpec("P")  # captura los parámetros de una función
R = TypeVar("R")    # captura el tipo de retorno

# Decorador correctamente tipado — preserva la firma de la función original
def loggear(func: Callable[P, R]) -> Callable[P, R]:
    from functools import wraps

    @wraps(func)
    def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
        print(f"Llamando {func.__name__}")
        return func(*args, **kwargs)

    return wrapper

@loggear
def sumar(a: int, b: int) -> int:
    return a + b

# El IDE y mypy saben que sumar todavía acepta (a: int, b: int) → int
sumar(1, 2)        # ✅
sumar("a", "b")    # Error de tipo — se preservó la firma original
```

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - La sintaxis moderna de Python 3.10+ usa `|` para unions y tipos built-in sin importar (`list[int]` en lugar de `List[int]`).
> - `Protocol` habilita duck typing verificado estáticamente — cualquier clase que implemente los métodos del Protocol es compatible sin declararlo explícitamente.
> - `TypeVar` y `Generic` permiten clases y funciones genéricas que preservan la información de tipo.
> - `TypedDict` da forma conocida a dicts. `Literal` restringe a valores exactos. `Final` declara constantes.
> - `ParamSpec` es la herramienta para tipar decoradores correctamente — preserva la firma completa de la función decorada.

### Para llevar a la práctica
- [ ] Convierte todos los dicts de configuración de tu proyecto a `TypedDict`
- [ ] Crea un `Protocol` para un repositorio: `Repositorio[T]` con `buscar`, `guardar`, `eliminar`
- [ ] Añade `mypy --strict` a tu pipeline de CI y corrige todos los errores

### Recursos
- 📖 *Fluent Python* — Capítulo 8 (type hints en funciones) y 15 (tipado más avanzado)
- 🌐 mypy.readthedocs.io — documentación oficial de mypy
- 📄 PEP 544 — Protocols: Structural subtyping
- 📄 PEP 604 — Allow writing union types as X | Y
- 🌐 github.com/microsoft/pyright — pyright, el type checker de Microsoft (más rápido que mypy)

---
`#python` `#tipos` `#mypy` `#protocol` `#genericos`
