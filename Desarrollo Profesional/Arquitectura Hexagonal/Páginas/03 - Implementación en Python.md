# 📐 Hexagonal 03 — Implementación Completa en Python

[[Desarrollo Profesional/Arquitectura Hexagonal/Arquitectura Hexagonal|⬅️ Volver a Hexagonal]] | [[Desarrollo Profesional/Arquitectura Hexagonal/Páginas/02 - Puertos y Adaptadores|← 02]]

> [!abstract] Introducción
> Una implementación real de arquitectura hexagonal en Python para un sistema de gestión de pedidos. El código muestra la estructura completa de directorios, el flujo de una petición HTTP a través de todas las capas, y cómo los tests prueban la lógica de negocio sin infraestructura.

## ¿De qué vamos a hablar?

Un ejemplo completo y realista: un servicio de pedidos con FastAPI, PostgreSQL y un sistema de notificaciones — todo organizado en capas hexagonales.

### Conceptos que vamos a cubrir
- Estructura de directorios de un proyecto hexagonal en Python
- La capa de dominio: entidades, Value Objects, puertos
- La capa de aplicación: casos de uso
- La capa de infraestructura: adaptadores HTTP y de persistencia
- Tests sin infraestructura

---

## El Concepto

### Estructura de Directorios

```
src/
├── domain/                    ← núcleo — sin dependencias externas
│   ├── __init__.py
│   ├── pedido.py              ← Entity + Value Objects
│   ├── puertos.py             ← interfaces (Protocol)
│   └── excepciones.py        ← excepciones de dominio
│
├── application/               ← casos de uso
│   ├── __init__.py
│   └── servicio_pedidos.py   ← orquestación
│
└── infrastructure/            ← adaptadores
    ├── http/
    │   └── pedidos_router.py  ← FastAPI routes
    ├── persistence/
    │   └── postgres_repo.py   ← RepositorioPedidoPostgres
    ├── notifications/
    │   └── smtp_notif.py      ← adaptador de email
    └── container.py           ← composición de dependencias

tests/
├── unit/
│   └── domain/
│       └── test_pedido.py     ← tests del dominio puro
└── integration/
    └── application/
        └── test_servicio.py   ← tests de casos de uso con adaptadores en memoria
```

### La Capa de Dominio

```python
# src/domain/pedido.py
from dataclasses import dataclass, field
from uuid import UUID, uuid4
from decimal import Decimal
from enum import Enum

class EstadoPedido(Enum):
    BORRADOR = "borrador"
    CONFIRMADO = "confirmado"
    ENVIADO = "enviado"
    CANCELADO = "cancelado"

@dataclass(frozen=True)
class LineaPedido:
    producto_id: UUID
    cantidad: int
    precio_unitario: Decimal

    @property
    def subtotal(self) -> Decimal:
        return self.precio_unitario * self.cantidad

@dataclass
class Pedido:
    id: UUID
    cliente_id: UUID
    lineas: list[LineaPedido] = field(default_factory=list)
    estado: EstadoPedido = EstadoPedido.BORRADOR

    def agregar_linea(self, producto_id: UUID, cantidad: int, precio: Decimal) -> None:
        if self.estado != EstadoPedido.BORRADOR:
            raise PedidoNoModificable(f"El pedido está en estado {self.estado.value}")
        if cantidad <= 0:
            raise ValueError("La cantidad debe ser positiva")
        self.lineas.append(LineaPedido(producto_id, cantidad, precio))

    def confirmar(self) -> None:
        if not self.lineas:
            raise PedidoVacio("No se puede confirmar un pedido sin líneas")
        if self.estado != EstadoPedido.BORRADOR:
            raise PedidoNoModificable(f"No se puede confirmar desde {self.estado.value}")
        self.estado = EstadoPedido.CONFIRMADO

    def cancelar(self, motivo: str) -> None:
        if self.estado in (EstadoPedido.ENVIADO, EstadoPedido.CANCELADO):
            raise PedidoNoModificable(f"No se puede cancelar desde {self.estado.value}")
        self.estado = EstadoPedido.CANCELADO

    @property
    def total(self) -> Decimal:
        return sum((l.subtotal for l in self.lineas), start=Decimal("0"))

    def __eq__(self, otro: object) -> bool:
        return isinstance(otro, Pedido) and self.id == otro.id

    def __hash__(self) -> int:
        return hash(self.id)

# src/domain/excepciones.py
class DomainError(Exception): ...
class PedidoVacio(DomainError): ...
class PedidoNoModificable(DomainError): ...
class StockInsuficiente(DomainError): ...

# src/domain/puertos.py
from typing import Protocol

class RepositorioPedido(Protocol):
    def obtener(self, id: UUID) -> "Pedido | None": ...
    def guardar(self, pedido: "Pedido") -> None: ...

class ServicioInventario(Protocol):
    def verificar_y_reservar(self, producto_id: UUID, cantidad: int) -> None: ...

class ServicioNotificaciones(Protocol):
    def notificar_confirmacion(self, cliente_id: UUID, pedido_id: UUID, total: Decimal) -> None: ...
```

### La Capa de Aplicación

```python
# src/application/servicio_pedidos.py
from dataclasses import dataclass
from uuid import UUID, uuid4
from decimal import Decimal

from domain.pedido import Pedido, LineaPedido
from domain.puertos import RepositorioPedido, ServicioInventario, ServicioNotificaciones
from domain.excepciones import StockInsuficiente

@dataclass(frozen=True)
class AgregarLineaCommand:
    pedido_id: UUID
    producto_id: UUID
    cantidad: int
    precio_unitario: Decimal

@dataclass(frozen=True)
class ConfirmarPedidoCommand:
    pedido_id: UUID

class ServicioPedidosImpl:
    def __init__(
        self,
        repositorio: RepositorioPedido,
        inventario: ServicioInventario,
        notificaciones: ServicioNotificaciones,
    ) -> None:
        self._repo = repositorio
        self._inventario = inventario
        self._notif = notificaciones

    def crear_pedido(self, cliente_id: UUID) -> UUID:
        pedido = Pedido(id=uuid4(), cliente_id=cliente_id)
        self._repo.guardar(pedido)
        return pedido.id

    def agregar_linea(self, cmd: AgregarLineaCommand) -> None:
        pedido = self._obtener_o_fallar(cmd.pedido_id)

        # Verificar stock antes de añadir (puerto secundario)
        disponible = self._inventario.verificar_y_reservar(cmd.producto_id, cmd.cantidad)

        pedido.agregar_linea(cmd.producto_id, cmd.cantidad, cmd.precio_unitario)
        self._repo.guardar(pedido)

    def confirmar_pedido(self, cmd: ConfirmarPedidoCommand) -> None:
        pedido = self._obtener_o_fallar(cmd.pedido_id)
        pedido.confirmar()
        self._repo.guardar(pedido)
        # Notificar — puerto secundario
        self._notif.notificar_confirmacion(pedido.cliente_id, pedido.id, pedido.total)

    def _obtener_o_fallar(self, id: UUID) -> Pedido:
        pedido = self._repo.obtener(id)
        if pedido is None:
            raise ValueError(f"Pedido {id} no encontrado")
        return pedido
```

### La Capa de Infraestructura — Adaptador HTTP

```python
# src/infrastructure/http/pedidos_router.py
from fastapi import APIRouter, HTTPException, Depends, status
from pydantic import BaseModel
from uuid import UUID
from decimal import Decimal

from application.servicio_pedidos import ServicioPedidosImpl, AgregarLineaCommand, ConfirmarPedidoCommand
from domain.excepciones import DomainError

router = APIRouter(prefix="/pedidos", tags=["Pedidos"])

class AgregarLineaBody(BaseModel):
    producto_id: UUID
    cantidad: int
    precio_unitario: Decimal

@router.post("/", status_code=status.HTTP_201_CREATED)
async def crear_pedido(
    cliente_id: UUID,
    servicio: ServicioPedidosImpl = Depends(obtener_servicio)
) -> dict:
    pedido_id = servicio.crear_pedido(cliente_id)
    return {"pedido_id": str(pedido_id)}

@router.post("/{pedido_id}/lineas")
async def agregar_linea(
    pedido_id: UUID,
    body: AgregarLineaBody,
    servicio: ServicioPedidosImpl = Depends(obtener_servicio)
) -> dict:
    try:
        cmd = AgregarLineaCommand(
            pedido_id=pedido_id,
            producto_id=body.producto_id,
            cantidad=body.cantidad,
            precio_unitario=body.precio_unitario
        )
        servicio.agregar_linea(cmd)
        return {"ok": True}
    except DomainError as e:
        raise HTTPException(status_code=422, detail=str(e))

@router.post("/{pedido_id}/confirmar")
async def confirmar_pedido(
    pedido_id: UUID,
    servicio: ServicioPedidosImpl = Depends(obtener_servicio)
) -> dict:
    try:
        servicio.confirmar_pedido(ConfirmarPedidoCommand(pedido_id=pedido_id))
        return {"ok": True}
    except DomainError as e:
        raise HTTPException(status_code=422, detail=str(e))
```

### Tests Sin Infraestructura

```python
# tests/unit/domain/test_pedido.py
import pytest
from uuid import uuid4
from decimal import Decimal
from domain.pedido import Pedido, EstadoPedido
from domain.excepciones import PedidoVacio, PedidoNoModificable

def test_pedido_nuevo_en_borrador():
    pedido = Pedido(id=uuid4(), cliente_id=uuid4())
    assert pedido.estado == EstadoPedido.BORRADOR
    assert pedido.total == Decimal("0")

def test_confirmar_pedido_con_lineas():
    pedido = Pedido(id=uuid4(), cliente_id=uuid4())
    pedido.agregar_linea(uuid4(), cantidad=2, precio=Decimal("10.00"))
    pedido.confirmar()
    assert pedido.estado == EstadoPedido.CONFIRMADO

def test_no_se_puede_confirmar_pedido_vacio():
    pedido = Pedido(id=uuid4(), cliente_id=uuid4())
    with pytest.raises(PedidoVacio):
        pedido.confirmar()

def test_no_se_puede_agregar_linea_a_pedido_confirmado():
    pedido = Pedido(id=uuid4(), cliente_id=uuid4())
    pedido.agregar_linea(uuid4(), cantidad=1, precio=Decimal("10.00"))
    pedido.confirmar()
    with pytest.raises(PedidoNoModificable):
        pedido.agregar_linea(uuid4(), cantidad=1, precio=Decimal("5.00"))

# tests/integration/application/test_servicio.py
from uuid import uuid4
from decimal import Decimal
import pytest

# Adaptadores en memoria — sin BD, sin red
class InventarioMemoria:
    def verificar_y_reservar(self, producto_id, cantidad):
        pass  # siempre hay stock en tests

class NotificacionesMemoria:
    def __init__(self):
        self.enviadas = []

    def notificar_confirmacion(self, cliente_id, pedido_id, total):
        self.enviadas.append((cliente_id, pedido_id, total))

def test_flujo_completo_crear_y_confirmar():
    repo = RepositorioPedidoMemoria()
    notif = NotificacionesMemoria()
    servicio = ServicioPedidosImpl(repo, InventarioMemoria(), notif)

    # Crear pedido
    pedido_id = servicio.crear_pedido(uuid4())

    # Agregar línea
    servicio.agregar_linea(AgregarLineaCommand(
        pedido_id=pedido_id,
        producto_id=uuid4(),
        cantidad=3,
        precio_unitario=Decimal("15.00")
    ))

    # Confirmar
    servicio.confirmar_pedido(ConfirmarPedidoCommand(pedido_id=pedido_id))

    # Verificar
    pedido = repo.obtener(pedido_id)
    assert pedido.estado.value == "confirmado"
    assert pedido.total == Decimal("45.00")
    assert len(notif.enviadas) == 1  # se notificó
```

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - La estructura de directorios refleja la arquitectura: `domain/`, `application/`, `infrastructure/`. Las dependencias de imports solo van hacia dentro.
> - El dominio (`domain/`) no importa nada de `application/` ni `infrastructure/`. Es puro Python sin frameworks.
> - Los tests de aplicación usan adaptadores en memoria — son tests de integración sin infraestructura real. Son rápidos y deterministas.
> - Un error común es poner lógica de negocio en el router (controlador HTTP). El router solo traduce y delega.

### Para llevar a la práctica
- [ ] Crea la estructura de carpetas `domain/`, `application/`, `infrastructure/` en tu proyecto
- [ ] Mueve las reglas de negocio más importantes a clases en `domain/` que no importen nada externo
- [ ] Escribe un test de integración usando adaptadores en memoria para el flujo principal de tu aplicación

### Recursos
- 📖 *Architecture Patterns with Python* — Harry Percival y Bob Gregory (el mejor libro de hexagonal en Python)
- 🌐 github.com/cosmicpython/code — el código del libro anterior
- 🌐 github.com/Enforcer/clean-architecture — ejemplo real en Python

---
`#hexagonal` `#python` `#implementacion` `#fastapi` `#testing`
