# 🔀 CQRS 02 — Implementación en Python

[[Desarrollo Profesional/CQRS/CQRS|⬅️ Volver a CQRS]] | [[Desarrollo Profesional/CQRS/Páginas/01 - Concepto y Motivación|← 01]] | [[Desarrollo Profesional/CQRS/Páginas/03 - Read Models Eventos y Escalado|03 →]]

> [!abstract] Introducción
> Aterricemos el concepto en código. El CQRS más útil y barato es el del **nivel 2**: objetos `Command` y `Query` inmutables, un *handler* dedicado para cada uno, y un despachador (el *bus* o *mediator*) que conecta cada mensaje con su handler. Sobre esa columna vertebral se cuelgan, sin tocar el dominio, cosas como logging, validación o transacciones.

## ¿De qué vamos a hablar?

De cómo montar una infraestructura CQRS limpia en Python: los mensajes, los handlers, el bus que los une y los *decorators* (middleware) que añaden comportamiento transversal.

### Conceptos que vamos a cubrir
- Commands y queries como mensajes inmutables
- Handlers: una clase, una responsabilidad
- El Command Bus / Query Bus (patrón mediator)
- Middleware: logging, validación y transacciones sin ensuciar el dominio
- Cómo encaja con FastAPI

---

## El Concepto

### Los Mensajes: Commands y Queries

Un mensaje es un objeto de datos plano e inmutable. Sin lógica: solo describe la intención.

```python
from dataclasses import dataclass
from uuid import UUID

# --- COMMANDS: intenciones de cambio, nombre imperativo ---
@dataclass(frozen=True)
class ConfirmarPedido:
    pedido_id: UUID
    metodo_pago: str

@dataclass(frozen=True)
class CancelarPedido:
    pedido_id: UUID
    motivo: str

# --- QUERIES: peticiones de lectura, nombre interrogativo ---
@dataclass(frozen=True)
class ObtenerResumenPedido:
    pedido_id: UUID

@dataclass(frozen=True)
class ListarPedidosCliente:
    cliente_id: UUID
    pagina: int = 1
    por_pagina: int = 20
```

### Los Handlers

Cada mensaje tiene **exactamente un** handler. Un command handler orquesta el modelo de dominio; un query handler va directo a una proyección de lectura, sin pasar por los Aggregates.

```python
from typing import Protocol, TypeVar, Generic

C = TypeVar("C")   # tipo de command
Q = TypeVar("Q")   # tipo de query
R = TypeVar("R")   # tipo de resultado

class CommandHandler(Protocol[C]):
    def handle(self, command: C) -> None: ...

class QueryHandler(Protocol[Q, R]):
    def handle(self, query: Q) -> R: ...


# Command handler: usa el modelo de dominio rico
class ConfirmarPedidoHandler:
    def __init__(self, repo: "RepositorioPedido", bus_eventos: "EventBus") -> None:
        self._repo = repo
        self._eventos = bus_eventos

    def handle(self, command: ConfirmarPedido) -> None:
        pedido = self._repo.obtener(command.pedido_id)
        if pedido is None:
            raise PedidoNoEncontrado(command.pedido_id)

        pedido.confirmar(metodo_pago=command.metodo_pago)   # toda la regla vive aquí
        self._repo.guardar(pedido)

        for evento in pedido.tirar_eventos():               # ver página 03
            self._eventos.publicar(evento)


# Query handler: SQL directo a una vista desnormalizada, sin dominio
@dataclass(frozen=True)
class ResumenPedidoDTO:
    id: str
    estado: str
    total: float
    num_lineas: int

class ObtenerResumenPedidoHandler:
    def __init__(self, db_lectura) -> None:
        self._db = db_lectura

    def handle(self, query: ObtenerResumenPedido) -> ResumenPedidoDTO | None:
        fila = self._db.execute(
            "SELECT id, estado, total, num_lineas "
            "FROM vista_resumen_pedidos WHERE id = %s",
            (str(query.pedido_id),),
        ).fetchone()
        if not fila:
            return None
        return ResumenPedidoDTO(**fila)
```

Fíjate en la asimetría: el command handler navega el [[Desarrollo Profesional/DDD/Páginas/02 - Bloques Tácticos|Aggregate]] y respeta sus invariantes; el query handler hace un `SELECT` a una tabla pensada para esa pantalla. Cada lado optimizado para lo suyo.

### El Bus: un Mediator

El *bus* desacopla a quien emite el mensaje de quien lo procesa. El emisor (un endpoint, un job) solo conoce el mensaje; el bus encuentra el handler y lo ejecuta. Es el patrón **mediator**.

```python
class CommandBus:
    def __init__(self) -> None:
        self._handlers: dict[type, CommandHandler] = {}

    def registrar(self, tipo_command: type, handler: CommandHandler) -> None:
        self._handlers[tipo_command] = handler

    def enviar(self, command) -> None:
        handler = self._handlers.get(type(command))
        if handler is None:
            raise RuntimeError(f"Sin handler para {type(command).__name__}")
        handler.handle(command)


class QueryBus:
    def __init__(self) -> None:
        self._handlers: dict[type, QueryHandler] = {}

    def registrar(self, tipo_query: type, handler: QueryHandler) -> None:
        self._handlers[tipo_query] = handler

    def preguntar(self, query):
        handler = self._handlers.get(type(query))
        if handler is None:
            raise RuntimeError(f"Sin handler para {type(query).__name__}")
        return handler.handle(query)
```

### Middleware: Comportamiento Transversal

La gran ventaja del bus es que todo command pasa por un único punto. Ahí cuelgas, una sola vez, lo que aplica a todos: logging, validación, métricas, transacciones. Sin tocar ningún handler.

```python
from typing import Callable

# Un decorator que envuelve el envío en una transacción de BD
class TransaccionMiddleware:
    def __init__(self, siguiente: Callable, uow: "UnitOfWork") -> None:
        self._siguiente = siguiente
        self._uow = uow

    def __call__(self, command) -> None:
        with self._uow:                 # abre transacción
            self._siguiente(command)
            self._uow.commit()          # confirma si no hubo excepción


class LoggingMiddleware:
    def __init__(self, siguiente: Callable) -> None:
        self._siguiente = siguiente

    def __call__(self, command) -> None:
        nombre = type(command).__name__
        print(f"→ ejecutando {nombre}")
        self._siguiente(command)
        print(f"✓ completado {nombre}")
```

Cada middleware envuelve al siguiente formando una cebolla: `Logging( Transaccion( handler ) )`. La validación con [[Desarrollo Profesional/Pydantic/Pydantic|Pydantic]] o las métricas de [[Desarrollo Profesional/Prometheus/Prometheus|Prometheus]] encajan igual.

### Cableado con FastAPI

El endpoint queda finísimo: traduce HTTP a un mensaje y delega. La lógica vive en los handlers, no en el controlador.

```python
from fastapi import FastAPI, Depends

app = FastAPI()

@app.post("/pedidos/{pedido_id}/confirmacion", status_code=202)
def confirmar(pedido_id: UUID, body: dict, bus: CommandBus = Depends(get_command_bus)):
    bus.enviar(ConfirmarPedido(pedido_id=pedido_id, metodo_pago=body["metodo_pago"]))
    return {"estado": "aceptado"}

@app.get("/pedidos/{pedido_id}/resumen")
def resumen(pedido_id: UUID, bus: QueryBus = Depends(get_query_bus)):
    dto = bus.preguntar(ObtenerResumenPedido(pedido_id=pedido_id))
    if dto is None:
        raise HTTPException(404)
    return dto
```

> [!tip] No te montes el framework
> En proyectos pequeños no necesitas un bus con middleware. Empieza llamando al handler directamente desde el endpoint; introduce el bus cuando aparezca la necesidad real de comportamiento transversal o de desacoplar emisores de handlers. CQRS premia la **disciplina**, no la infraestructura.

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - Modela cada caso de uso como un mensaje inmutable: `Command` (imperativo) o `Query` (interrogativo).
> - Un mensaje, un handler. El **command handler** orquesta el modelo de dominio rico; el **query handler** va directo a una proyección de lectura.
> - El **bus/mediator** desacopla al emisor del handler y crea un punto único por donde pasa todo.
> - Ese punto único permite añadir **middleware** (logging, validación, transacciones, métricas) sin tocar el dominio: se envuelven en cebolla.
> - Con CQRS los endpoints quedan delgados: traducen el transporte a un mensaje y delegan.

### Para llevar a la práctica
- [ ] Convierte un caso de uso a un `Command` + su handler y llámalo desde el endpoint
- [ ] Añade un `CommandBus` mínimo y registra el handler en él
- [ ] Implementa un middleware de logging y otro de transacción, y compón la cebolla
- [ ] Escribe un query handler que lea de una vista desnormalizada en vez de cargar el Aggregate

### Recursos
- 📖 *Architecture Patterns with Python* — Percival & Gregory (capítulos de command bus y unit of work)
- 🌐 github.com/bobthemighty/punq — un contenedor de DI sencillo para cablear handlers
- 🌐 cosmicpython.com — la versión online del libro anterior, gratuita

---
`#cqrs` `#python` `#mediator` `#command-bus` `#fastapi`
