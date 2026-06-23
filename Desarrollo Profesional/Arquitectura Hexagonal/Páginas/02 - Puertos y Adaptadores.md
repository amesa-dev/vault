# 📐 Hexagonal 02 — Puertos y Adaptadores

[[Desarrollo Profesional/Arquitectura Hexagonal/Arquitectura Hexagonal|⬅️ Volver a Hexagonal]] | [[Desarrollo Profesional/Arquitectura Hexagonal/Páginas/01 - Concepto|← 01]] | [[Desarrollo Profesional/Arquitectura Hexagonal/Páginas/03 - Implementación en Python|03 →]]

> [!abstract] Introducción
> Los puertos y adaptadores son los dos conceptos que dan nombre al patrón. Los **puertos** son interfaces definidas por el dominio — el contrato que dice "necesito esto para funcionar". Los **adaptadores** son las implementaciones de esos contratos para una tecnología concreta. Hay dos tipos de puertos: los de entrada (cómo el mundo llama a nuestra lógica) y los de salida (cómo nuestra lógica habla con el mundo exterior).

## ¿De qué vamos a hablar?

La distinción entre puertos primarios y secundarios, cómo se implementan como Protocols en Python, y los patrones de inyección de dependencias para conectar todo.

### Conceptos que vamos a cubrir
- Puertos primarios (entrada): los casos de uso como interfaces
- Puertos secundarios (salida): repositorios, servicios externos
- Adaptadores primarios: controladores HTTP, CLI, tests
- Adaptadores secundarios: BD, email, APIs externas
- Dependency Injection en Python

---

## El Concepto

### Puertos Primarios (Entrada) — Cómo el Mundo Entra

Los puertos primarios definen lo que el sistema puede hacer. Son las acciones del usuario (casos de uso). El mundo exterior llama a estos puertos:

```python
from typing import Protocol
from uuid import UUID
from dataclasses import dataclass

# Comandos (lo que el usuario quiere hacer)
@dataclass(frozen=True)
class CrearPedidoCommand:
    cliente_id: UUID
    producto_id: UUID
    cantidad: int

@dataclass(frozen=True)
class ConfirmarPedidoCommand:
    pedido_id: UUID

# Puerto primario — interfaz del caso de uso
class ServicioPedidos(Protocol):
    def crear_pedido(self, cmd: CrearPedidoCommand) -> UUID: ...
    def confirmar_pedido(self, cmd: ConfirmarPedidoCommand) -> None: ...
    def cancelar_pedido(self, pedido_id: UUID, motivo: str) -> None: ...
```

### Puertos Secundarios (Salida) — Cómo la Lógica Habla con el Exterior

Los puertos secundarios son las interfaces que la lógica de negocio necesita implementadas para funcionar. El dominio los define, la infraestructura los implementa:

```python
from typing import Protocol
from uuid import UUID

# Puerto secundario para persistencia de pedidos
class RepositorioPedido(Protocol):
    def obtener(self, id: UUID) -> "Pedido | None": ...
    def guardar(self, pedido: "Pedido") -> None: ...
    def listar_por_cliente(self, cliente_id: UUID) -> list["Pedido"]: ...

# Puerto secundario para notificaciones
class ServicioNotificaciones(Protocol):
    def enviar_confirmacion(self, cliente_id: UUID, pedido_id: UUID) -> None: ...
    def enviar_cancelacion(self, cliente_id: UUID, pedido_id: UUID, motivo: str) -> None: ...

# Puerto secundario para verificar stock
class ServicioInventario(Protocol):
    def hay_stock(self, producto_id: UUID, cantidad: int) -> bool: ...
    def reservar(self, producto_id: UUID, cantidad: int) -> None: ...
```

### Adaptadores Primarios — Cómo el Mundo Invoca la Lógica

Los adaptadores primarios traducen la petición del mundo exterior (HTTP, CLI, test) al lenguaje del dominio:

```python
from fastapi import FastAPI, HTTPException, Depends
from pydantic import BaseModel
from uuid import UUID

# Adaptador HTTP (FastAPI) — puerto primario en HTTP
app = FastAPI()

class CrearPedidoRequest(BaseModel):
    cliente_id: UUID
    producto_id: UUID
    cantidad: int

@app.post("/pedidos")
async def crear_pedido(
    body: CrearPedidoRequest,
    servicio: ServicioPedidos = Depends(obtener_servicio_pedidos)  # inyección
) -> dict:
    cmd = CrearPedidoCommand(
        cliente_id=body.cliente_id,
        producto_id=body.producto_id,
        cantidad=body.cantidad
    )
    try:
        pedido_id = servicio.crear_pedido(cmd)
        return {"pedido_id": str(pedido_id)}
    except ValueError as e:
        raise HTTPException(status_code=422, detail=str(e))

# Adaptador CLI
import click

@click.command()
@click.argument("cliente_id")
@click.argument("producto_id")
@click.argument("cantidad", type=int)
def crear_pedido_cli(cliente_id: str, producto_id: str, cantidad: int) -> None:
    servicio = construir_servicio_pedidos()  # composición manual
    cmd = CrearPedidoCommand(
        cliente_id=UUID(cliente_id),
        producto_id=UUID(producto_id),
        cantidad=cantidad
    )
    pedido_id = servicio.crear_pedido(cmd)
    click.echo(f"Pedido creado: {pedido_id}")
```

### Adaptadores Secundarios — Implementaciones Concretas

```python
import asyncpg
from uuid import UUID

# Adaptador de persistencia — implementa RepositorioPedido con PostgreSQL
class RepositorioPedidoPostgres:
    def __init__(self, pool: asyncpg.Pool) -> None:
        self._pool = pool

    async def obtener(self, id: UUID) -> "Pedido | None":
        async with self._pool.acquire() as conn:
            fila = await conn.fetchrow(
                "SELECT id, cliente_id, estado FROM pedidos WHERE id = $1",
                id
            )
            if not fila:
                return None
            return self._mapear_a_dominio(fila)

    async def guardar(self, pedido: "Pedido") -> None:
        async with self._pool.acquire() as conn:
            await conn.execute(
                """
                INSERT INTO pedidos (id, cliente_id, estado)
                VALUES ($1, $2, $3)
                ON CONFLICT (id) DO UPDATE SET estado = EXCLUDED.estado
                """,
                pedido.id, pedido.cliente_id, pedido.estado
            )

    def _mapear_a_dominio(self, fila) -> "Pedido":
        return Pedido(id=fila["id"], cliente_id=fila["cliente_id"], estado=fila["estado"])

# Adaptador en memoria — para tests
class RepositorioPedidoMemoria:
    def __init__(self) -> None:
        self._store: dict[UUID, "Pedido"] = {}

    def obtener(self, id: UUID) -> "Pedido | None":
        return self._store.get(id)

    def guardar(self, pedido: "Pedido") -> None:
        self._store[pedido.id] = pedido

    def listar_por_cliente(self, cliente_id: UUID) -> list["Pedido"]:
        return [p for p in self._store.values() if p.cliente_id == cliente_id]

# Adaptador de notificaciones — SendGrid
class NotificacionesSendGrid:
    def __init__(self, api_key: str) -> None:
        self._api_key = api_key

    def enviar_confirmacion(self, cliente_id: UUID, pedido_id: UUID) -> None:
        # llamada real a la API de SendGrid
        pass

# Adaptador de notificaciones — en memoria para tests
class NotificacionesMemoria:
    def __init__(self) -> None:
        self.enviadas: list[dict] = []

    def enviar_confirmacion(self, cliente_id: UUID, pedido_id: UUID) -> None:
        self.enviadas.append({"tipo": "confirmacion", "cliente": cliente_id, "pedido": pedido_id})
```

### Dependency Injection — Conectando Todo

La inyección de dependencias es el mecanismo que conecta los adaptadores con los casos de uso:

```python
# Composición manual — el enfoque más simple
def construir_servicio_pedidos() -> ServicioPedidosImpl:
    import os

    if os.getenv("ENV") == "test":
        repo = RepositorioPedidoMemoria()
        notif = NotificacionesMemoria()
        inventario = InventarioMemoria()
    else:
        pool = asyncpg.create_pool(os.getenv("DATABASE_URL"))
        repo = RepositorioPedidoPostgres(pool)
        notif = NotificacionesSendGrid(os.getenv("SENDGRID_API_KEY"))
        inventario = InventarioHttp(os.getenv("INVENTARIO_URL"))

    return ServicioPedidosImpl(
        repositorio=repo,
        notificaciones=notif,
        inventario=inventario
    )

# Con FastAPI Depends — inyección por defecto del framework
def obtener_servicio_pedidos() -> ServicioPedidos:
    return construir_servicio_pedidos()

# Con un contenedor DI (dependency-injector library)
from dependency_injector import containers, providers

class Container(containers.DeclarativeContainer):
    config = providers.Configuration()

    pool = providers.Singleton(
        asyncpg.create_pool,
        dsn=config.database.url
    )

    repositorio_pedido = providers.Factory(
        RepositorioPedidoPostgres,
        pool=pool
    )

    servicio_pedidos = providers.Factory(
        ServicioPedidosImpl,
        repositorio=repositorio_pedido
    )
```

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - Los **puertos primarios** son las acciones que el mundo exterior puede invocar en el sistema (casos de uso). Los **puertos secundarios** son las dependencias que el sistema necesita del exterior (repositorios, servicios).
> - Los **adaptadores primarios** traducen el mundo exterior (HTTP, CLI) al idioma del dominio. Los **adaptadores secundarios** implementan los contratos del dominio con tecnologías concretas.
> - En Python, `Protocol` es la herramienta perfecta para definir puertos — permite duck typing estructural sin acoplamiento explícito.
> - La inyección de dependencias es el mecanismo que conecta adaptadores con casos de uso. Puede ser composición manual, el sistema DI del framework, o una librería como `dependency-injector`.

### Para llevar a la práctica
- [ ] Define un puerto secundario para el repositorio más usado en tu proyecto
- [ ] Crea una implementación en memoria de ese repositorio y úsala en los tests
- [ ] Conecta el adaptador real y el de tests mediante composición manual en el punto de entrada

### Recursos
- 🌐 alistair.cockburn.us/hexagonal-architecture
- 📖 *Architecture Patterns with Python* — Harry Percival y Bob Gregory (Python hexagonal real)
- 🌐 python-dependency-injector.ets-labs.org — librería de DI para Python

---
`#hexagonal` `#puertos` `#adaptadores` `#dependency-injection`
