# 🐍 07 — Manejo de Errores

[[Desarrollo Profesional/Python/Python|⬅️ Volver al índice]] | [[Desarrollo Profesional/Python/06 - Módulos y Paquetes|← 06]] | [[Desarrollo Profesional/Python/08 - Iteradores y Generadores|08 →]]

> [!abstract] Introducción
> El manejo de errores en Python va mucho más allá del try/except básico. Python sigue la filosofía EAFP (Easier to Ask Forgiveness than Permission): intenta hacer algo y maneja el error si falla, en lugar de verificar condiciones antes de actuar. Los context managers extienden esto al ciclo de vida de recursos. Entender bien ambos mecanismos produce código más robusto y más legible.

## ¿De qué vamos a hablar?

El sistema completo de excepciones de Python: la jerarquía de excepciones, cuándo capturar y cuándo no, cómo crear excepciones propias significativas, y los context managers para gestión de recursos.

### Conceptos que vamos a cubrir
- try/except/else/finally: el flujo completo
- La jerarquía de excepciones built-in
- Cuándo capturar, cuándo propagar, cuándo transformar
- Excepciones custom con herencia
- Context managers: `with` y `__enter__`/`__exit__`
- `contextlib`: shortcuts para context managers

---

## El Concepto

### try/except/else/finally

```python
def dividir(a: float, b: float) -> float:
    try:
        resultado = a / b          # código que puede fallar
    except ZeroDivisionError:
        print("División por cero")
        return 0.0
    except (TypeError, ValueError) as e:  # captura múltiples tipos
        print(f"Error de tipo: {e}")
        raise                              # re-lanza la excepción original
    else:
        # Se ejecuta si NO hubo excepciones en el try
        print(f"Resultado: {resultado}")
        return resultado
    finally:
        # Se ejecuta SIEMPRE — con o sin excepción
        print("Operación terminada")

# El bloque else es poderoso y poco conocido:
# Semánticamente significa "esto solo ocurre si todo fue bien"
# Es más claro que poner el código al final del try
```

### La Jerarquía de Excepciones

```
BaseException
├── SystemExit           # sys.exit() — NO capturar salvo casos extremos
├── KeyboardInterrupt    # Ctrl+C — NO capturar salvo cleanup específico
├── GeneratorExit        # cierre de generador
└── Exception            # base de todas las excepciones "normales"
    ├── ArithmeticError
    │   ├── ZeroDivisionError
    │   └── OverflowError
    ├── LookupError
    │   ├── KeyError      # acceso a clave inexistente en dict
    │   └── IndexError    # acceso a índice fuera de rango
    ├── ValueError        # valor incorrecto para el tipo correcto
    ├── TypeError         # tipo incorrecto
    ├── AttributeError    # acceso a atributo inexistente
    ├── OSError (IOError) # errores de sistema operativo/IO
    │   ├── FileNotFoundError
    │   ├── PermissionError
    │   └── TimeoutError
    ├── RuntimeError
    └── StopIteration
```

**Regla crítica:** Nunca captures `BaseException` ni `Exception` genéricamente sin un propósito muy claro. `except Exception: pass` es uno de los patrones más peligrosos — silencia errores reales y hace el debugging imposible.

```python
# ❌ MAL — silencia todo
try:
    resultado = operacion()
except Exception:
    pass

# ✅ BIEN — captura lo específico, deja pasar lo que no esperas
try:
    resultado = operacion()
except FileNotFoundError as e:
    logger.warning(f"Archivo no encontrado: {e}")
    resultado = valor_por_defecto
```

### EAFP vs LBYL

```python
# LBYL — Look Before You Leap (estilo C/Java)
def obtener_valor(diccionario: dict, clave: str) -> str:
    if clave in diccionario:                  # verifica antes
        if isinstance(diccionario[clave], str):
            return diccionario[clave]
    return ""

# EAFP — Easier to Ask Forgiveness than Permission (estilo Python)
def obtener_valor(diccionario: dict, clave: str) -> str:
    try:
        return diccionario[clave]
    except (KeyError, TypeError):
        return ""

# EAFP es más idiomático y evita race conditions
# (entre la verificación y el uso, el estado puede cambiar)

# En muchos casos, la forma más limpia es:
return diccionario.get(clave, "")  # para dicts con valor por defecto
```

### Excepciones Custom — Diseño Correcto

```python
# Base para el dominio — agrupa todas las excepciones del paquete
class PaqueteError(Exception):
    """Error base del paquete. Capturar esto para manejar todos los errores del paquete."""
    pass

# Excepciones específicas con contexto útil
class UsuarioNoEncontradoError(PaqueteError):
    def __init__(self, user_id: int) -> None:
        self.user_id = user_id
        super().__init__(f"Usuario con id {user_id} no encontrado")

class CredencialesInvalidasError(PaqueteError):
    def __init__(self, email: str) -> None:
        self.email = email
        super().__init__(f"Credenciales inválidas para {email}")

class LimiteAlcanzadoError(PaqueteError):
    def __init__(self, recurso: str, limite: int) -> None:
        self.recurso = recurso
        self.limite  = limite
        super().__init__(f"Límite de {limite} alcanzado para {recurso}")

# Uso
def obtener_usuario(user_id: int) -> dict:
    usuario = db.find(user_id)
    if not usuario:
        raise UsuarioNoEncontradoError(user_id)
    return usuario

try:
    u = obtener_usuario(999)
except UsuarioNoEncontradoError as e:
    print(f"ID buscado: {e.user_id}")  # acceso al atributo específico
```

### Exception Chaining — Preservar el Contexto

```python
# raise from — encadena excepciones: la nueva wrappea la original
try:
    datos = json.loads(texto_invalido)
except json.JSONDecodeError as e:
    raise ValueError(f"El archivo de configuración tiene formato inválido") from e
    # El traceback mostrará ambas excepciones y la relación entre ellas

# raise from None — suprime la excepción original
try:
    conexion = db.connect()
except psycopg2.Error as e:
    raise ConexionError("No se puede conectar a la base de datos") from None
    # No muestra el error interno de psycopg2 al usuario final
```

### Context Managers — Gestión de Recursos

El protocolo context manager garantiza que los recursos se liberan correctamente, incluso si hay una excepción:

```python
# Implementación con __enter__ y __exit__
class ConexionBD:
    def __init__(self, url: str) -> None:
        self.url = url
        self.conexion = None

    def __enter__(self) -> "ConexionBD":
        print(f"Conectando a {self.url}...")
        self.conexion = conectar(self.url)
        return self  # o self.conexion — lo que recibe 'as'

    def __exit__(
        self,
        exc_type: type | None,
        exc_val: Exception | None,
        exc_tb: object | None
    ) -> bool:
        print("Cerrando conexión...")
        if self.conexion:
            self.conexion.close()
        # Retorna True para suprimir la excepción (raramente deseable)
        # Retorna None/False para que la excepción se propague
        return False

# Uso
with ConexionBD("postgresql://localhost/mydb") as conn:
    conn.ejecutar("SELECT * FROM usuarios")
# La conexión se cierra automáticamente aquí — incluso si hay excepción

# Múltiples context managers en una sola línea
with open("entrada.txt") as entrada, open("salida.txt", "w") as salida:
    salida.write(entrada.read())
```

### `contextlib` — Context Managers sin Clases

```python
from contextlib import contextmanager, suppress, nullcontext

# @contextmanager — más simple que implementar __enter__/__exit__
@contextmanager
def cronometrar(nombre: str):
    import time
    inicio = time.perf_counter()
    try:
        yield  # aquí se ejecuta el código del with
    finally:
        fin = time.perf_counter()
        print(f"{nombre}: {fin - inicio:.3f}s")

with cronometrar("proceso"):
    [x**2 for x in range(1_000_000)]

# suppress — suprime excepciones específicas
from contextlib import suppress

with suppress(FileNotFoundError):
    os.remove("archivo_que_puede_no_existir.tmp")
# Equivalente a try/except FileNotFoundError: pass — pero más claro

# nullcontext — context manager que no hace nada (útil para condicionales)
from contextlib import nullcontext

def procesar(archivo: str, lock=None):
    ctx = lock or nullcontext()
    with ctx:
        # code here
        pass
```

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - El flujo completo es `try/except/else/finally`. El bloque `else` se ejecuta solo si no hubo excepción — para código que solo debe ejecutarse si todo fue bien.
> - Captura la excepción más específica posible. `except Exception: pass` es un antipatrón — silencia errores reales.
> - EAFP (Easier to Ask Forgiveness than Permission) es más idiomático en Python que verificar condiciones antes de actuar.
> - Las excepciones custom deben: heredar de una base del dominio, incluir atributos con contexto específico, y tener mensajes de error útiles.
> - Los context managers (`with`) garantizan la liberación de recursos. `@contextmanager` de `contextlib` es la forma más simple de crear uno.

### Para llevar a la práctica
- [ ] Crea una jerarquía de excepciones para un módulo: `ModuloError` base, con 3-4 excepciones específicas con atributos de contexto
- [ ] Escribe un context manager `transaccion()` que haga commit si no hay excepción y rollback si la hay
- [ ] Convierte un bloque try/except FileNotFoundError al patrón con `contextlib.suppress`

### Recursos
- 📖 *Fluent Python* — Capítulo 15 (context managers y generators)
- 🌐 docs.python.org/3/library/contextlib.html
- 🌐 docs.python.org/3/library/exceptions.html — jerarquía completa
- 📄 PEP 343 — The "with" Statement

---
`#python` `#excepciones` `#errores` `#context-managers`
