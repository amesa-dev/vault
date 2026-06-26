# 🐍 00 — Introducción y Setup

[[Desarrollo Profesional/Python/Python|⬅️ Volver al índice de Python]] | [[Desarrollo Profesional/Python/01 - Tipos Primitivos y Variables|01 →]]

> [!abstract] Introducción
> Antes de escribir la primera línea de Python útil, hay que entender cómo funciona el ecosistema: el intérprete, los entornos virtuales y las herramientas modernas de gestión de proyectos. Saltarse este paso genera confusión con paquetes, versiones y entornos que se mezclan sin control.

## ¿De qué vamos a hablar?

Este módulo establece las bases del entorno de trabajo. Aprenderás a instalar Python correctamente, a aislar dependencias por proyecto con entornos virtuales y a entender la estructura de un proyecto moderno.

### Conceptos que vamos a cubrir
- Cómo funciona el intérprete CPython
- Entornos virtuales: por qué son imprescindibles
- pip vs uv: gestión de paquetes moderna
- pyproject.toml: el estándar moderno de configuración
- Estructura de un proyecto Python profesional

---

## El Concepto

### El Intérprete de Python

Python es un lenguaje interpretado. El código fuente (`.py`) pasa por el intérprete CPython, que lo compila a bytecode (`.pyc`) y lo ejecuta en la máquina virtual de Python.

```
código.py → (compilación) → bytecode (.pyc) → (interpretación) → ejecución
```

**CPython** es la implementación de referencia (la que instalas con `python.org`). Existen otras como PyPy (más rápida, JIT compilation) pero CPython es el estándar.

**Versiones:** Python 3.10+ es el mínimo razonable hoy. Python 3.12 introduce mejoras de rendimiento significativas. Nunca uses Python 2 en código nuevo.

```bash
python --version     # o python3 --version
python3.12 --version # versión específica
```

### Entornos Virtuales — El Concepto Más Importante

Por defecto, `pip install` instala paquetes globalmente en el sistema. Esto crea conflictos cuando distintos proyectos necesitan versiones distintas del mismo paquete.

**La solución:** Entornos virtuales. Un entorno virtual es una copia aislada del intérprete Python con su propio directorio de paquetes. Cada proyecto tiene el suyo.

```bash
# Crear un entorno virtual
python -m venv .venv

# Activarlo (Linux/Mac)
source .venv/bin/activate

# Activarlo (Windows)
.venv\Scripts\activate

# Verificar que estás dentro
which python  # debe apuntar a .venv/bin/python

# Desactivar
deactivate
```

> Convención: llamar `.venv` al entorno virtual y añadirlo al `.gitignore`.

### pip: Gestión de Paquetes

`pip` es el gestor de paquetes estándar.

```bash
pip install requests              # instalar
pip install requests==2.31.0      # versión específica
pip install "requests>=2.28"      # rango de versiones
pip uninstall requests            # desinstalar
pip list                          # listar instalados
pip freeze > requirements.txt     # exportar dependencias

# Instalar desde requirements.txt
pip install -r requirements.txt
```

### uv: El Estándar Moderno (2024+)

`uv` es un gestor de paquetes y proyectos Python escrito en Rust, desarrollado por Astral. Es 10-100x más rápido que pip y resuelve muchos de sus problemas.

```bash
# Instalar uv
curl -LsSf https://astral.sh/uv/install.sh | sh

# Crear proyecto nuevo
uv init mi-proyecto
cd mi-proyecto

# Añadir dependencias
uv add requests
uv add pytest --dev        # dependencia de desarrollo

# Ejecutar código
uv run python main.py

# Sincronizar entorno
uv sync
```

`uv` gestiona automáticamente el entorno virtual y el `pyproject.toml`.

### pyproject.toml — Configuración Moderna

El estándar moderno para configurar proyectos Python (PEP 517/518/621). Reemplaza `setup.py`, `setup.cfg` y `requirements.txt`.

```toml
[project]
name = "mi-proyecto"
version = "0.1.0"
description = "Descripción del proyecto"
requires-python = ">=3.12"
dependencies = [
    "requests>=2.31",
    "pydantic>=2.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0",
    "mypy>=1.8",
    "ruff>=0.3",
]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.mypy]
strict = true
python_version = "3.12"

[tool.ruff]
line-length = 88
```

### Estructura de un Proyecto Python Profesional

```
mi-proyecto/
├── .venv/                  # entorno virtual (en .gitignore)
├── src/
│   └── mi_proyecto/        # código fuente (patrón src-layout)
│       ├── __init__.py
│       ├── main.py
│       └── utils.py
├── tests/
│   ├── __init__.py
│   └── test_main.py
├── .gitignore
├── pyproject.toml
└── README.md
```

**src-layout:** Poner el código en `src/` previene importaciones accidentales del código fuente en lugar del paquete instalado. Es el patrón recomendado para proyectos que se publicarán como paquete.

### Herramientas de Calidad de Código

```bash
# ruff: linter y formatter ultra-rápido (reemplaza flake8, black, isort)
uv add ruff --dev
ruff check .        # lint
ruff format .       # format

# mypy: verificación de tipos estáticos
uv add mypy --dev
mypy src/

# pre-commit: ejecutar checks antes de cada commit
uv add pre-commit --dev
pre-commit install
```

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - Python es interpretado: código → bytecode → ejecución en la VM. CPython es la implementación de referencia.
> - Los entornos virtuales son obligatorios: cada proyecto tiene su propio `.venv` con sus propias dependencias aisladas.
> - `uv` es el estándar moderno: más rápido que pip, gestiona el entorno virtual automáticamente.
> - `pyproject.toml` centraliza toda la configuración del proyecto en un solo archivo.
> - La estructura recomendada usa `src/` para el código fuente y `tests/` para los tests.

### Para llevar a la práctica
- [ ] Instala `uv` y crea un proyecto nuevo con `uv init`
- [ ] Añade `ruff` y `mypy` como dependencias de desarrollo
- [ ] Verifica que el entorno virtual se activa correctamente con `which python`
- [ ] Crea la estructura `src/mi_paquete/` con un `__init__.py` vacío

### Recursos
- 🌐 docs.astral.sh/uv — documentación oficial de uv
- 📄 PEP 517, 518, 621 — los estándares detrás de pyproject.toml
- 🌐 packaging.python.org — guía oficial de packaging de Python
- 📖 *Python Packaging User Guide* — packaging.python.org/guides

---
`#python` `#setup` `#entorno` `#herramientas`
