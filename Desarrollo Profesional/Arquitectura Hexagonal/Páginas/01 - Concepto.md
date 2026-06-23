# 📐 Hexagonal 01 — El Concepto

[[Desarrollo Profesional/Arquitectura Hexagonal/Arquitectura Hexagonal|⬅️ Volver a Hexagonal]] | [[Desarrollo Profesional/Arquitectura Hexagonal/Páginas/02 - Puertos y Adaptadores|02 →]]

> [!abstract] Introducción
> La arquitectura hexagonal nació de una idea simple pero poderosa: la lógica de negocio no debería saber nada sobre cómo se almacenan los datos, cómo llegan las peticiones o qué framework web se usa. El hexágono no es una forma — es una metáfora para decir que el núcleo del sistema puede conectarse a cualquier agente externo a través de interfaces bien definidas.

## ¿De qué vamos a hablar?

El problema que resuelve la arquitectura hexagonal, la idea del hexágono, y por qué hace que el código sea más testable, más mantenible y más resistente al cambio tecnológico.

### Conceptos que vamos a cubrir
- El problema que resuelve: el acoplamiento con infraestructura
- La metáfora del hexágono
- Las tres capas: Dominio, Aplicación e Infraestructura
- Por qué el testing mejora radicalmente

---

## El Concepto

### El Problema: Acoplamiento con Infraestructura

Imagina una función de negocio que procesa pedidos. En una arquitectura sin capas definidas, la lógica de negocio está entrelazada con la infraestructura:

```python
# ❌ Problema: la lógica de negocio está acoplada a la infraestructura
import psycopg2
import smtplib
from flask import Flask, request, jsonify

app = Flask(__name__)

@app.route("/pedidos", methods=["POST"])
def crear_pedido():
    datos = request.json
    conn = psycopg2.connect("postgresql://...")  # infraestructura
    cursor = conn.cursor()

    # Lógica de negocio mezclada con SQL
    cursor.execute("SELECT stock FROM productos WHERE id = %s", (datos["producto_id"],))
    stock = cursor.fetchone()[0]

    if stock < datos["cantidad"]:
        return jsonify({"error": "Sin stock"}), 400  # regla de negocio enterrada en un controlador

    cursor.execute(
        "INSERT INTO pedidos (cliente_id, producto_id, cantidad) VALUES (%s, %s, %s)",
        (datos["cliente_id"], datos["producto_id"], datos["cantidad"])
    )
    conn.commit()

    # Email de confirmación acoplado al flujo
    smtp = smtplib.SMTP("smtp.gmail.com")
    smtp.sendmail("no-reply@tienda.com", datos["email"], "Pedido confirmado")

    return jsonify({"ok": True}), 201
```

Este código tiene varios problemas graves:
- **Imposible de testear** sin una base de datos real y un servidor SMTP
- **Imposible de cambiar** la base de datos sin tocar la lógica de negocio
- **Imposible de reutilizar** la lógica en otro canal (CLI, tests, worker)
- **La regla de negocio** (`stock < cantidad`) está oculta entre SQL e infraestructura

### La Solución: El Hexágono

La arquitectura hexagonal resuelve esto con una regla simple: **la lógica de negocio define interfaces; la infraestructura las implementa**.

```
                     ┌─────────────────────────────────────────┐
                     │                                         │
  HTTP Request  ─────┤►  Adaptador HTTP (FastAPI Controller)   │
  (Driver)           │                │                        │
                     │                ▼                        │
                     │  ┌─────────────────────────────────┐   │
                     │  │   Puerto Entrada (Use Case)     │   │
                     │  │                                 │   │
                     │  │   ┌─────────────────────────┐  │   │
  CLI Command  ─────────►  │      DOMINIO             │  │   │
  (Driver)           │  │  │   (Entities, VO, Rules)  │  │   │
                     │  │  └─────────────────────────┘  │   │
                     │  │                                 │   │
                     │  │   Puerto Salida (Repository)   │   │
                     │  └─────────────────────────────────┘   │
                     │                │                        │
                     │                ▼                        │
                     │  Adaptador DB (PostgresRepository) ─────┤► PostgreSQL
                     │                                         │
                     │  Adaptador Email (SmtpMailer) ──────────┤► SMTP
                     │                                         │
                     └─────────────────────────────────────────┘
```

### Las Tres Capas

**Dominio** — el núcleo. Contiene:
- Entidades, Value Objects, Aggregates
- Reglas de negocio puras
- Interfaces (Puertos) que define para hablar con el exterior
- **No depende de nada externo** — ni Flask, ni SQLAlchemy, ni nada

**Aplicación** — los casos de uso. Contiene:
- Orquestación: recibe un comando, llama al dominio, guarda, publica eventos
- Implementa los puertos de entrada (las acciones que puede invocar el mundo exterior)
- Usa los puertos de salida (repositorios, servicios externos) pero no sabe cómo están implementados

**Infraestructura** — los adaptadores. Contiene:
- Controladores HTTP (FastAPI, Flask)
- Implementaciones de repositorios (Postgres, MongoDB, memoria)
- Clientes de APIs externas
- Consumers de mensajes (Kafka, SQS)

### La Regla de las Dependencias

Las dependencias siempre apuntan hacia el centro:

```
Infraestructura → Aplicación → Dominio
```

El Dominio no sabe nada de la Aplicación. La Aplicación no sabe nada de la Infraestructura. Esto se logra con la **Inversión de Dependencias (DIP)**:

```python
# El dominio define el puerto (la interfaz)
from typing import Protocol
from uuid import UUID

class RepositorioPedido(Protocol):  # Puerto de salida — en el dominio
    def obtener(self, id: UUID) -> "Pedido | None": ...
    def guardar(self, pedido: "Pedido") -> None: ...

# La infraestructura lo implementa
class RepositorioPedidoPostgres:   # Adaptador — en la infraestructura
    def __init__(self, session) -> None:
        self._session = session

    def obtener(self, id: UUID) -> "Pedido | None":
        # implementación con SQLAlchemy
        pass

    def guardar(self, pedido: "Pedido") -> None:
        # implementación con SQLAlchemy
        pass

# El dominio/aplicación usa el puerto — no sabe qué implementación hay detrás
class ProcesarPedidoUseCase:
    def __init__(self, repo: RepositorioPedido) -> None:  # depende de la interfaz
        self._repo = repo

    def ejecutar(self, pedido_id: UUID) -> None:
        pedido = self._repo.obtener(pedido_id)  # usa el puerto
        # ...
```

### Por Qué el Testing Mejora

Con este diseño, los tests de la lógica de negocio no necesitan base de datos, ni HTTP, ni SMTP:

```python
def test_procesar_pedido_confirma_si_hay_stock():
    # Arrange — repositorio en memoria, sin BD real
    repo = RepositorioPedidoMemoria()
    pedido = crear_pedido_con_stock(producto_id=uuid4(), cantidad=5)
    repo.guardar(pedido)

    caso_de_uso = ProcesarPedidoUseCase(repo=repo)

    # Act
    caso_de_uso.ejecutar(pedido.id)

    # Assert
    pedido_guardado = repo.obtener(pedido.id)
    assert pedido_guardado.estado == "confirmado"
    # El test es rápido, determinista y no necesita infraestructura real
```

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - El problema que resuelve la arquitectura hexagonal es el **acoplamiento entre lógica de negocio e infraestructura** — cuando mezclamos SQL, HTTP y reglas de negocio en el mismo lugar.
> - La solución es definir el **hexágono** (dominio + aplicación) que vive en el centro y que define interfaces para hablar con el mundo exterior.
> - Las dependencias siempre apuntan hacia dentro: Infraestructura → Aplicación → Dominio. Nunca al revés.
> - El beneficio más inmediato es el **testing**: puedes probar toda la lógica de negocio con implementaciones en memoria, sin bases de datos ni servicios externos.

### Para llevar a la práctica
- [ ] Identifica en tu código actual dónde la lógica de negocio está mezclada con SQL o HTTP
- [ ] Extrae una regla de negocio a una función pura (sin efectos secundarios, sin dependencias externas)
- [ ] Escribe un test para esa función — ¿necesita alguna infraestructura para ejecutarse?

### Recursos
- 🌐 alistair.cockburn.us/hexagonal-architecture — el artículo original de Cockburn
- 📖 *Clean Architecture* — Robert C. Martin (generaliza el concepto)
- 🌐 netflixtechblog.com — artículos sobre arquitectura hexagonal en producción

---
`#hexagonal` `#arquitectura` `#dominio` `#testing`
