# 🐍 06 — Módulos y Paquetes

[[Desarrollo Profesional/Python/Python|⬅️ Volver al índice]] | [[Desarrollo Profesional/Python/05 - POO|← 05]] | [[Desarrollo Profesional/Python/07 - Manejo de Errores|07 →]]

> [!abstract] Introducción
> En Python, cada archivo `.py` es un módulo y un conjunto de módulos relacionados es un paquete. El sistema de imports es el mecanismo que conecta todo. Entenderlo bien evita los errores más frustrantes que aparecen cuando los proyectos crecen: imports circulares, nombres colisionados, y el misterio de por qué Python importa el módulo equivocado.

## ¿De qué vamos a hablar?

Cómo funciona el sistema de imports de Python internamente, cómo estructurar un paquete profesional y las convenciones modernas con `pyproject.toml`.

### Conceptos que vamos a cubrir
- El sistema de imports: `sys.path` y cómo Python encuentra módulos
- `import` vs `from ... import`
- `__init__.py`: qué pone y qué no
- Imports relativos y absolutos
- `__all__`: controlar la API pública
- Estructura de paquete con src-layout

---

## El Concepto

### Cómo Funciona el Sistema de Imports

Cuando escribes `import modulo`, Python:

1. Busca en `sys.modules` — si ya fue importado, retorna la versión cacheada
2. Busca en `sys.path` en orden: directorio actual → `PYTHONPATH` → librerías instaladas
3. Carga el módulo, ejecuta su código de nivel superior, y lo almacena en `sys.modules`

```python
import sys

# Ver qué directorios busca Python
print(sys.path)

# Ver módulos ya importados
import json
print("json" in sys.modules)  # True

# Los módulos se importan una sola vez
# Si importas el mismo módulo dos veces, Python usa la versión cacheada
import json
import json  # no re-ejecuta json.py — usa sys.modules["json"]
```

### Formas de Importar

```python
# Import de módulo completo — acceso con prefijo
import os
import os.path
ruta = os.path.join("/home", "user", "archivo.txt")

# From import — importa nombres específicos al namespace actual
from os.path import join, exists
ruta = join("/home", "user", "archivo.txt")

# From import con alias
from collections import defaultdict as dd
grupos = dd(list)

# Import con alias — útil para nombres largos
import numpy as np          # convención universal
import pandas as pd

# Import de todo — EVITAR (contamina el namespace)
from os import *            # ❌ — no sabes qué importaste
from os.path import *       # ❌

# Cuándo sí tiene sentido:
# En __init__.py para re-exportar (controlado con __all__)
```

### `__init__.py` — El Corazón del Paquete

Un directorio con `__init__.py` es un paquete. Sin él, Python 3 lo trata como "namespace package" (que funciona pero diferente).

```
mi_paquete/
├── __init__.py      ← hace que el directorio sea un paquete
├── modulo_a.py
├── modulo_b.py
└── subpaquete/
    ├── __init__.py
    └── modulo_c.py
```

```python
# mi_paquete/__init__.py

# Opción 1: vacío — el paquete existe pero no re-exporta nada
# Los usuarios importan: from mi_paquete.modulo_a import Clase

# Opción 2: re-exportar la API pública
from mi_paquete.modulo_a import ClasePrincipal
from mi_paquete.modulo_b import funcion_util

# Ahora los usuarios pueden: from mi_paquete import ClasePrincipal
# En lugar de: from mi_paquete.modulo_a import ClasePrincipal

# __all__ define qué se exporta con "from paquete import *"
__all__ = ["ClasePrincipal", "funcion_util"]
```

**Convención moderna:** Mantener los `__init__.py` cortos. No poner lógica compleja — solo re-exportaciones.

### Imports Relativos vs Absolutos

```python
# Estructura:
# mi_paquete/
#   __init__.py
#   models.py
#   services/
#     __init__.py
#     usuario_service.py
#     pedido_service.py

# En usuario_service.py:

# Importación absoluta (recomendada)
from mi_paquete.models import Usuario

# Importación relativa (con . como prefijo)
from ..models import Usuario      # .. = subir un nivel (a mi_paquete/)
from .pedido_service import crear_pedido  # . = mismo directorio (services/)

# Cuándo usar relativas:
# Dentro de un paquete que podría renombrarse — las relativas no se rompen
# Con src-layout, las absolutas son las preferidas
```

### `__all__` — Controlar la API Pública

```python
# modulo.py

def funcion_publica() -> str:
    return "pública"

def _funcion_privada() -> str:
    return "privada (por convención)"

def funcion_interna() -> str:
    return "interna"

# __all__ define exactamente qué se expone
__all__ = ["funcion_publica"]

# Ahora: from modulo import * — solo importa funcion_publica
# Pero: from modulo import funcion_interna — sigue funcionando (es una guía, no forzado)
```

### Estructura de Proyecto Profesional

```
mi_proyecto/
├── src/
│   └── mi_paquete/
│       ├── __init__.py
│       ├── domain/
│       │   ├── __init__.py
│       │   ├── models.py
│       │   └── exceptions.py
│       ├── application/
│       │   ├── __init__.py
│       │   └── services.py
│       ├── infrastructure/
│       │   ├── __init__.py
│       │   ├── repositories.py
│       │   └── database.py
│       └── api/
│           ├── __init__.py
│           └── routes.py
├── tests/
│   ├── __init__.py
│   ├── unit/
│   └── integration/
├── pyproject.toml
└── .gitignore
```

```toml
# pyproject.toml — con src-layout
[tool.setuptools.packages.find]
where = ["src"]

# O con hatchling
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["src/mi_paquete"]
```

### Imports Circulares — El Error Más Frustrante

Un import circular ocurre cuando `A` importa `B` y `B` importa `A`. Python detecta el ciclo y falla o retorna un módulo parcialmente inicializado.

```python
# MAL DISEÑO — genera circular import
# models.py
from services import calcular  # importa services

# services.py
from models import Usuario    # importa models ← CIRCULAR

# SOLUCIONES:
# 1. Mover el import al interior de la función (lazy import)
# services.py
def usar_usuario():
    from models import Usuario  # se importa solo cuando se llama la función
    return Usuario()

# 2. Crear un módulo intermedio (mejor diseño)
# interfaces.py — define las abstracciones
# models.py — usa interfaces.py
# services.py — usa interfaces.py
# Ninguno importa al otro directamente

# 3. Reestructurar — los imports circulares suelen indicar problemas de diseño
```

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - Python busca módulos en `sys.path` en orden. El directorio de trabajo actual está primero — puede causar que importes el módulo equivocado si hay conflictos de nombres.
> - `__init__.py` convierte un directorio en paquete. Úsalo para re-exportar la API pública, mantenlo corto.
> - Las importaciones absolutas son las recomendadas en proyectos modernos con src-layout.
> - `__all__` define qué se exporta con `from paquete import *` y documenta la API pública.
> - Los imports circulares son casi siempre síntoma de diseño acoplado. La solución no es un lazy import — es separar responsabilidades.

### Para llevar a la práctica
- [ ] Crea un paquete con `__init__.py` que re-exporte solo la API pública usando `__all__`
- [ ] Configura un proyecto con src-layout y verifica que los imports absolutos funcionan
- [ ] Detecta y elimina un import circular en un proyecto usando un módulo de interfaces

### Recursos
- 🌐 docs.python.org/3/reference/import.html — el sistema de imports completo
- 📄 PEP 328 — Imports relativos
- 📖 *Python Packaging User Guide* — packaging.python.org
- 🌐 packaging.python.org/en/latest/guides/src-layout-vs-flat-layout

---
`#python` `#modulos` `#paquetes` `#imports`
