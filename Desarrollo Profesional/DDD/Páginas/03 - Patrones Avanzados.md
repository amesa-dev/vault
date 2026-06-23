# 🧩 DDD 03 — Patrones Avanzados

[[Desarrollo Profesional/DDD/DDD|⬅️ Volver a DDD]] | [[Desarrollo Profesional/DDD/Páginas/02 - Bloques Tácticos|← 02]]

> [!abstract] Introducción
> Una vez dominados los bloques tácticos, hay patrones que extienden DDD para sistemas más complejos: los Domain Events desacoplan los bounded contexts, CQRS separa lecturas y escrituras para escalar cada una independientemente, y Event Sourcing cambia el modelo de persistencia — en lugar de guardar el estado actual, guardas la historia de eventos que lo produjeron.

## ¿De qué vamos a hablar?

Los tres patrones avanzados de DDD más usados en sistemas distribuidos y de alta escala, con implementaciones en Python.

### Conceptos que vamos a cubrir
- Domain Events: comunicación desacoplada entre Aggregates y Contexts
- CQRS: separar el modelo de lectura del de escritura
- Event Sourcing: el estado como derivado de una serie de eventos

---

## El Concepto

### Domain Events — Cosas que Ocurrieron en el Dominio

Un Domain Event representa algo que ocurrió en el dominio que es relevante para otras partes del sistema. Se nombran en pasado. Son inmutables (son hechos históricos):

```python
from dataclasses import dataclass, field
from uuid import UUID, uuid4
from datetime import datetime, timezone

# Eventos de dominio — inmutables, nombrados en pasado
@dataclass(frozen=True)
class PedidoConfirmado:
    pedido_id: UUID
    cliente_id: UUID
    total: float
    ocurrido_en: datetime = field(default_factory=lambda: datetime.now(timezone.utc))
    event_id: UUID = field(default_factory=uuid4)

@dataclass(frozen=True)
class PagoRechazado:
    pedido_id: UUID
    motivo: str
    ocurrido_en: datetime = field(default_factory=lambda: datetime.now(timezone.utc))
    event_id: UUID = field(default_factory=uuid4)

# Aggregate que produce Domain Events
@dataclass
class Pedido:
    id: UUID
    cliente_id: UUID
    estado: str = "borrador"
    _eventos: list = field(default_factory=list, repr=False)

    def confirmar(self, total: float) -> None:
        if self.estado != "pendiente_pago":
            raise ValueError(f"No se puede confirmar desde estado: {self.estado}")

        self.estado = "confirmado"
        evento = PedidoConfirmado(
            pedido_id=self.id,
            cliente_id=self.cliente_id,
            total=total
        )
        self._eventos.append(evento)

    def obtener_eventos_pendientes(self) -> list:
        return list(self._eventos)

    def limpiar_eventos(self) -> None:
        self._eventos.clear()

# Event Bus — para publicar y suscribirse a eventos
from typing import Callable, Any, Type

class EventBus:
    def __init__(self) -> None:
        self._suscriptores: dict[type, list[Callable]] = {}

    def suscribir(self, tipo_evento: Type, handler: Callable) -> None:
        self._suscriptores.setdefault(tipo_evento, []).append(handler)

    def publicar(self, evento: Any) -> None:
        handlers = self._suscriptores.get(type(evento), [])
        for handler in handlers:
            handler(evento)

# Handlers en otros bounded contexts
class NotificacionesHandler:
    def al_confirmar_pedido(self, evento: PedidoConfirmado) -> None:
        print(f"Enviando email de confirmación para pedido {evento.pedido_id}")

class InventarioHandler:
    def al_confirmar_pedido(self, evento: PedidoConfirmado) -> None:
        print(f"Reservando stock para pedido {evento.pedido_id}")

# Wiring
bus = EventBus()
notif = NotificacionesHandler()
inv = InventarioHandler()
bus.suscribir(PedidoConfirmado, notif.al_confirmar_pedido)
bus.suscribir(PedidoConfirmado, inv.al_confirmar_pedido)
```

### CQRS — Command Query Responsibility Segregation

CQRS separa las operaciones que cambian el estado (Commands) de las que solo leen (Queries). Esto permite optimizar cada lado independientemente — el lado de escritura puede usar el modelo de dominio rico, y el lado de lectura puede usar proyecciones planas y rápidas:

```python
from dataclasses import dataclass
from uuid import UUID
from typing import Any

# COMMANDS — intenciones de cambio
@dataclass(frozen=True)
class ConfirmarPedidoCommand:
    pedido_id: UUID
    metodo_pago: str

@dataclass(frozen=True)
class CancelarPedidoCommand:
    pedido_id: UUID
    motivo: str

# QUERIES — peticiones de lectura
@dataclass(frozen=True)
class ObtenerResumenPedidoQuery:
    pedido_id: UUID

@dataclass(frozen=True)
class ListarPedidosClienteQuery:
    cliente_id: UUID
    pagina: int = 1
    por_pagina: int = 20

# Command Handlers — usan el modelo de dominio rico
class ConfirmarPedidoHandler:
    def __init__(self, repo: "RepositorioPedido", bus: EventBus) -> None:
        self._repo = repo
        self._bus = bus

    def handle(self, cmd: ConfirmarPedidoCommand) -> None:
        pedido = self._repo.obtener(cmd.pedido_id)
        if pedido is None:
            raise ValueError(f"Pedido {cmd.pedido_id} no encontrado")

        pedido.confirmar(total=pedido.calcular_total())

        self._repo.guardar(pedido)

        # Publicar eventos producidos
        for evento in pedido.obtener_eventos_pendientes():
            self._bus.publicar(evento)
        pedido.limpiar_eventos()

# Query Handlers — usan proyecciones optimizadas para lectura
@dataclass
class ResumenPedidoDTO:
    id: str
    estado: str
    total: float
    fecha: str
    num_items: int

class ObtenerResumenPedidoHandler:
    def __init__(self, db_lectura: Any) -> None:
        self._db = db_lectura  # podría ser un view de BD, Redis, etc.

    def handle(self, query: ObtenerResumenPedidoQuery) -> ResumenPedidoDTO | None:
        # SQL directo, sin pasar por el modelo de dominio
        fila = self._db.execute(
            "SELECT id, estado, total, fecha, num_items "
            "FROM vista_pedidos WHERE id = %s",
            (str(query.pedido_id),)
        ).fetchone()

        if not fila:
            return None

        return ResumenPedidoDTO(
            id=fila["id"],
            estado=fila["estado"],
            total=fila["total"],
            fecha=fila["fecha"].isoformat(),
            num_items=fila["num_items"]
        )
```

### Event Sourcing — El Estado como Historia de Eventos

Event Sourcing cambia el modelo de persistencia: en lugar de guardar el estado actual, guardas la secuencia de eventos que llevaron a ese estado. El estado actual se reconstruye reproduciendo los eventos:

```python
from dataclasses import dataclass, field
from uuid import UUID, uuid4
from datetime import datetime, timezone
from typing import Any

# Eventos de dominio de la cuenta bancaria
@dataclass(frozen=True)
class CuentaAbierta:
    cuenta_id: UUID
    titular: str
    deposito_inicial: float
    ocurrido_en: datetime

@dataclass(frozen=True)
class DepositoRealizado:
    cuenta_id: UUID
    importe: float
    ocurrido_en: datetime

@dataclass(frozen=True)
class RetiroRealizado:
    cuenta_id: UUID
    importe: float
    ocurrido_en: datetime

# Aggregate con Event Sourcing
@dataclass
class CuentaBancaria:
    id: UUID = field(default_factory=uuid4)
    titular: str = ""
    saldo: float = 0.0
    _cambios: list = field(default_factory=list, repr=False)
    _version: int = 0

    # Estado reconstruido aplicando eventos
    def apply(self, evento: Any) -> None:
        match evento:
            case CuentaAbierta():
                self.titular = evento.titular
                self.saldo = evento.deposito_inicial
            case DepositoRealizado():
                self.saldo += evento.importe
            case RetiroRealizado():
                self.saldo -= evento.importe
        self._version += 1

    # Comandos que producen eventos
    def depositar(self, importe: float) -> None:
        if importe <= 0:
            raise ValueError("El importe debe ser positivo")
        evento = DepositoRealizado(
            cuenta_id=self.id,
            importe=importe,
            ocurrido_en=datetime.now(timezone.utc)
        )
        self.apply(evento)
        self._cambios.append(evento)

    def retirar(self, importe: float) -> None:
        if importe > self.saldo:
            raise ValueError("Saldo insuficiente")
        evento = RetiroRealizado(
            cuenta_id=self.id,
            importe=importe,
            ocurrido_en=datetime.now(timezone.utc)
        )
        self.apply(evento)
        self._cambios.append(evento)

    # Reconstruir desde eventos (replay)
    @classmethod
    def desde_eventos(cls, eventos: list) -> "CuentaBancaria":
        cuenta = cls()
        for evento in eventos:
            cuenta.apply(evento)
        return cuenta

# Event Store — almacena y recupera eventos
class EventStore:
    def __init__(self) -> None:
        self._events: dict[UUID, list] = {}

    def guardar(self, aggregate_id: UUID, eventos: list, version_esperada: int) -> None:
        almacenados = self._events.get(aggregate_id, [])
        if len(almacenados) != version_esperada:
            raise ValueError("Conflict: version mismatch (optimistic concurrency)")
        self._events.setdefault(aggregate_id, []).extend(eventos)

    def cargar(self, aggregate_id: UUID) -> list:
        return list(self._events.get(aggregate_id, []))
```

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - Los **Domain Events** permiten que los Bounded Contexts se comuniquen sin acoplarse directamente. El Aggregate produce eventos; otros contextos los consumen de forma asíncrona.
> - **CQRS** separa el modelo de escritura (rico, con reglas de dominio) del modelo de lectura (proyecciones planas y rápidas). No son incompatibles con un mismo almacén de datos.
> - **Event Sourcing** guarda la historia de eventos en lugar del estado actual. Ganas auditabilidad total y la capacidad de reconstruir el estado en cualquier punto del tiempo, a costa de mayor complejidad.
> - Los tres patrones se complementan: Event Sourcing + CQRS + Domain Events es la combinación estándar en sistemas distribuidos de alta escala.

### Para llevar a la práctica
- [ ] Añade Domain Events a un Aggregate existente: identifica qué "cosas importantes ocurrieron" y modélalas como eventos
- [ ] Implementa un handler de evento que actualice un contador o tabla de resumen cuando se confirma un pedido
- [ ] Prueba Event Sourcing en un aggregate simple: cuenta bancaria o carrito de compra

### Recursos
- 📖 *Implementing Domain-Driven Design* — Vaughn Vernon, Capítulo 8 (Domain Events)
- 🌐 martinfowler.com/bliki/CQRS.html
- 🌐 eventstore.com/blog/what-is-event-sourcing
- 📄 Greg Young — CQRS Documents (el paper original de CQRS+ES)

---
`#ddd` `#domain-events` `#cqrs` `#event-sourcing`
