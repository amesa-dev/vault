# 🐍 12 — Testing con Pytest

[[Desarrollo Profesional/Python/Python|⬅️ Volver al índice]] | [[Desarrollo Profesional/Python/11 - Concurrencia y Asyncio|← 11]]

> [!abstract] Introducción
> Pytest es el framework de testing de Python por excelencia. Su filosofía es opuesta al unittest de la librería estándar: minimiza el boilerplate, hace que los tests sean funciones simples, y ofrece un sistema de fixtures y plugins que escala desde el test más simple hasta suites de integración complejas. Un test bien escrito en Pytest se lee como una especificación del comportamiento que documenta.

## ¿De qué vamos a hablar?

Pytest desde la prueba básica hasta los patrones avanzados que se usan en proyectos profesionales: fixtures con scope, parametrize, mocking con monkeypatch y la estrategia de tests de arquitectura hexagonal.

### Conceptos que vamos a cubrir
- Tests básicos: afirmaciones y convenciones de Pytest
- Fixtures: configuración reutilizable y con scope
- `parametrize`: múltiples casos de prueba
- Mocking: `monkeypatch` y `unittest.mock`
- Tests asíncronos con pytest-asyncio
- Organización y estrategia: unit, integration, e2e

---

## El Concepto

### Tests Básicos

```python
# Pytest descubre automáticamente archivos test_*.py o *_test.py
# y funciones que empiezan por test_

# tests/test_calculadora.py
from mi_paquete.calculadora import sumar, dividir, Calculadora
import pytest

# Test básico — solo una función con assert
def test_sumar_dos_enteros():
    resultado = sumar(2, 3)
    assert resultado == 5

def test_sumar_negativos():
    assert sumar(-1, -1) == -2

# Test de excepción
def test_dividir_por_cero():
    with pytest.raises(ZeroDivisionError):
        dividir(10, 0)

def test_dividir_por_cero_mensaje():
    with pytest.raises(ZeroDivisionError, match="division by zero"):
        dividir(10, 0)

# Forzar fallo (para tests que todavía no están implementados)
def test_pendiente():
    pytest.fail("Este test necesita implementarse")

# Skip — saltar un test con razón
@pytest.mark.skip(reason="Funcionalidad no implementada")
def test_feature_nueva():
    pass

# skipif — saltar condicionalmente
import sys
@pytest.mark.skipif(sys.platform == "win32", reason="No funciona en Windows")
def test_solo_unix():
    pass
```

### Fixtures — Configuración Reutilizable

Las fixtures son funciones que proveen datos o recursos a los tests. Pytest las inyecta automáticamente por nombre:

```python
import pytest
from mi_paquete.models import Usuario, Pedido
from mi_paquete.repositorios import RepositorioUsuarioMemoria

# Fixture simple
@pytest.fixture
def usuario_basico() -> Usuario:
    return Usuario(id=1, nombre="Ana García", email="ana@ejemplo.com")

# Fixture con scope — cuánto tiempo vive
@pytest.fixture(scope="module")  # una sola vez por módulo
def conexion_db():
    conn = crear_conexion_test()
    yield conn              # yield en lugar de return — para cleanup
    conn.close()            # se ejecuta después de todos los tests del módulo

# Scopes disponibles:
# "function" (default) — una instancia por test
# "class" — una por clase de tests
# "module" — una por módulo
# "session" — una por ejecución completa de pytest

# Fixture con dependencias entre fixtures
@pytest.fixture
def repositorio(conexion_db):           # inyecta la fixture conexion_db
    return RepositorioUsuarioMemoria()  # o uno que use conexion_db

@pytest.fixture
def usuario_con_pedidos(usuario_basico, repositorio):
    repositorio.guardar(usuario_basico)
    pedido = Pedido(usuario_id=usuario_basico.id, total=100.0)
    repositorio.guardar_pedido(pedido)
    return usuario_basico, [pedido]

# Uso en tests
def test_obtener_pedidos_usuario(usuario_con_pedidos, repositorio):
    usuario, pedidos_esperados = usuario_con_pedidos
    pedidos = repositorio.obtener_pedidos(usuario.id)
    assert len(pedidos) == len(pedidos_esperados)
    assert pedidos[0].total == 100.0

# conftest.py — fixtures disponibles para todos los tests del directorio
# tests/conftest.py
@pytest.fixture(scope="session")
def cliente_http():
    """Cliente HTTP para tests de integración"""
    import httpx
    with httpx.Client(base_url="http://localhost:8000") as client:
        yield client
```

### `parametrize` — Múltiples Casos con un Solo Test

```python
# En lugar de escribir un test por caso:
@pytest.mark.parametrize("entrada, esperado", [
    (2, 4),
    (3, 9),
    (4, 16),
    (-2, 4),
    (0, 0),
])
def test_cuadrado(entrada: int, esperado: int) -> None:
    assert entrada ** 2 == esperado

# Múltiples parámetros con lógica compleja
@pytest.mark.parametrize("email, valido", [
    ("usuario@ejemplo.com", True),
    ("invalido",            False),
    ("@sin_usuario.com",    False),
    ("usuario@",            False),
    ("usuario@domain.co",   True),
])
def test_validar_email(email: str, valido: bool) -> None:
    from mi_paquete.validadores import validar_email
    assert validar_email(email) == valido

# Combinación de parametrize — genera el producto cartesiano
@pytest.mark.parametrize("a", [1, 2, 3])
@pytest.mark.parametrize("b", [10, 20])
def test_multiplicacion(a: int, b: int) -> None:
    assert a * b == b * a  # 6 tests en total
```

### Mocking con `monkeypatch` y `unittest.mock`

```python
from unittest.mock import patch, MagicMock, AsyncMock, call
import pytest

# monkeypatch — la forma Pytest de parchear
def test_obtener_tiempo(monkeypatch):
    monkeypatch.setattr("time.time", lambda: 1000.0)
    from mi_paquete import utils
    assert utils.timestamp_actual() == 1000.0

def test_variable_entorno(monkeypatch):
    monkeypatch.setenv("DATABASE_URL", "postgresql://test")
    monkeypatch.delenv("SECRET_KEY", raising=False)
    from mi_paquete.config import Config
    config = Config()
    assert config.database_url == "postgresql://test"

# unittest.mock.patch — más potente para clases y objetos
def test_servicio_llama_repositorio():
    mock_repo = MagicMock()
    mock_repo.buscar.return_value = Usuario(id=1, nombre="Test")

    servicio = UsuarioServicio(repositorio=mock_repo)
    usuario = servicio.obtener(1)

    # Verificar que se llamó con los argumentos correctos
    mock_repo.buscar.assert_called_once_with(1)
    assert usuario.nombre == "Test"

# patch como decorador
@patch("mi_paquete.servicios.requests.get")
def test_llamada_api_externa(mock_get):
    mock_get.return_value.status_code = 200
    mock_get.return_value.json.return_value = {"data": "test"}

    resultado = llamar_api("https://api.ejemplo.com/datos")

    mock_get.assert_called_once_with("https://api.ejemplo.com/datos", timeout=5)
    assert resultado == {"data": "test"}

# patch como context manager
def test_con_mock_contextmanager():
    with patch("mi_paquete.db.conectar") as mock_conectar:
        mock_conectar.return_value.__enter__ = lambda s: MagicMock()
        mock_conectar.return_value.__exit__ = MagicMock(return_value=False)
        # código que usa la conexión
```

### Tests Asíncronos

```python
import pytest
import asyncio

# pip install pytest-asyncio
import pytest_asyncio

@pytest.mark.asyncio
async def test_operacion_asincrona():
    resultado = await operacion_asincrona()
    assert resultado == "esperado"

# Fixture asíncrona
@pytest_asyncio.fixture
async def cliente_asincrono():
    async with aiohttp.ClientSession() as session:
        yield session

@pytest.mark.asyncio
async def test_llamada_http(cliente_asincrono):
    async with cliente_asincrono.get("https://api.ejemplo.com") as r:
        assert r.status == 200

# Mockear coroutines
from unittest.mock import AsyncMock

@pytest.mark.asyncio
async def test_servicio_asincrono():
    mock_repo = AsyncMock()
    mock_repo.buscar.return_value = {"id": 1, "nombre": "Test"}

    servicio = ServicioAsincrono(repositorio=mock_repo)
    resultado = await servicio.procesar(1)

    mock_repo.buscar.assert_awaited_once_with(1)
```

### Organización y Estrategia

```
tests/
├── conftest.py          ← fixtures compartidas por todo el proyecto
├── unit/                ← rápidos, sin I/O, sin BD
│   ├── conftest.py      ← fixtures de unit tests
│   ├── domain/
│   │   └── test_modelos.py
│   └── application/
│       └── test_servicios.py
├── integration/         ← usan BD real, servicios externos en contenedor
│   ├── conftest.py
│   └── test_repositorios.py
└── e2e/                 ← toda la aplicación corriendo
    └── test_flujos.py
```

```ini
# pytest.ini o en pyproject.toml
[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = [
    "--strict-markers",   # falla si se usa un marker no declarado
    "-v",                 # verbose
    "--tb=short",         # traceback corto
]
markers = [
    "slow: tests lentos (omitir con -m 'not slow')",
    "integration: tests de integración",
    "e2e: tests end-to-end",
]
asyncio_mode = "auto"    # para pytest-asyncio
```

```bash
# Ejecutar solo tests rápidos
pytest -m "not slow and not integration"

# Ver cobertura
pytest --cov=src --cov-report=html

# Ejecutar en paralelo (pip install pytest-xdist)
pytest -n auto
```

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - Pytest descubre tests automáticamente. Los tests son funciones simples — no necesitan clases ni herencia. El sistema de `assert` nativo da mensajes de error claros.
> - Las **fixtures** son la alternativa al setUp/tearDown: inyección de dependencias declarativa por nombre. El `scope` controla su ciclo de vida.
> - `@pytest.mark.parametrize` elimina la duplicación de tests con múltiples casos de entrada/salida.
> - `monkeypatch` y `unittest.mock` son complementarios: monkeypatch para configuración simple, mock para interacciones complejas con aserciones sobre llamadas.
> - Separa tests por velocidad y dependencias externas: unit (sin I/O) → integration (con BD) → e2e (aplicación completa).

### Para llevar a la práctica
- [ ] Convierte tres tests duplicados en uno con `@pytest.mark.parametrize`
- [ ] Crea una fixture de scope `"session"` para una conexión a BD de test que se crea una sola vez
- [ ] Añade `pytest --cov=src --cov-report=term-missing` a tu comando de test y mide la cobertura

### Recursos
- 📖 *Python Testing with pytest* — Brian Okken (la referencia definitiva)
- 🌐 docs.pytest.org — documentación oficial completa
- 🌐 pytest-mock.readthedocs.io — pytest-mock como alternativa a unittest.mock
- 📄 PEP 8 — Style guide (convenciones de naming en tests también aplican)

---
`#python` `#testing` `#pytest` `#fixtures` `#mocking`
