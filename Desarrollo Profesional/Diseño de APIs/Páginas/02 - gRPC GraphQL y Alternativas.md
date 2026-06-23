# 🔌 APIs 02 — gRPC, GraphQL y Alternativas

[[Desarrollo Profesional/Diseño de APIs/Diseño de APIs|⬅️ Volver a Diseño de APIs]] | [[Desarrollo Profesional/Diseño de APIs/Páginas/01 - REST y Diseño de Recursos|← 01]]

> [!abstract] Introducción
> REST es un gran defecto, pero no es la respuesta a todo. Cuando dos microservicios internos se llaman millones de veces, el overhead de JSON sobre HTTP/1.1 importa, y gRPC ofrece binario sobre HTTP/2 con un contrato estricto. Cuando un cliente móvil necesita datos de cinco recursos y no quiere hacer cinco llamadas ni recibir campos que no usa, GraphQL le deja pedir exactamente lo que necesita. Esta página compara los tres estilos, explica cuándo elegir cada uno, y cierra con algo transversal: la importancia del contrato (OpenAPI, Protobuf) como fuente de verdad.

## ¿De qué vamos a hablar?

Las alternativas a REST, sus trade-offs, y cómo elegir el estilo de API adecuado a cada problema.

### Conceptos que vamos a cubrir
- gRPC: contratos Protobuf, HTTP/2, streaming
- GraphQL: el cliente pide la forma de los datos
- REST vs gRPC vs GraphQL: la tabla de decisión
- El contrato como fuente de verdad: OpenAPI y Protobuf
- Contract testing entre servicios

---

## El Concepto

### gRPC — RPC de Alto Rendimiento

gRPC es un framework de llamada a procedimiento remoto (RPC) que define el contrato en **Protocol Buffers** (Protobuf): un esquema fuertemente tipado del que se generan automáticamente el cliente y el servidor en muchos lenguajes. Va sobre **HTTP/2** y serializa en **binario** (compacto y rápido de parsear).

```protobuf
// pedidos.proto — el contrato, fuente de verdad para cliente y servidor
syntax = "proto3";

service ServicioPedidos {
  rpc ObtenerPedido(PedidoRequest) returns (Pedido);
  rpc StreamPedidos(ClienteRequest) returns (stream Pedido);  // streaming nativo
}

message PedidoRequest { string pedido_id = 1; }

message Pedido {
  string id = 1;
  string estado = 2;
  double total = 3;
}
```

Ventajas: **rendimiento** (binario + HTTP/2 multiplexado), **contrato estricto** generado (menos errores de integración), **streaming bidireccional** nativo, y multilenguaje. Inconvenientes: no es legible por humanos (binario), no funciona directamente desde un navegador (necesita gRPC-Web + proxy), y el tooling es más pesado que un `curl`. **Su nicho: comunicación servicio-a-servicio interna**, donde el rendimiento y el contrato importan y nadie lo llama desde un navegador.

### GraphQL — El Cliente Decide la Forma

GraphQL invierte el control: en vez de que el servidor defina endpoints fijos con respuestas fijas, expone un **grafo de tipos** y deja que el cliente pida **exactamente los campos que necesita** en una sola petición:

```graphql
# El cliente pide justo esto — ni más campos ni menos, en una llamada
query {
  pedido(id: "42") {
    estado
    total
    cliente { nombre email }      # datos de otro "recurso", misma petición
    lineas { producto { nombre } cantidad }
  }
}
```

Resuelve dos problemas clásicos de REST:
- **Over-fetching**: REST devuelve el recurso completo aunque solo quieras dos campos. GraphQL devuelve solo lo pedido (clave para móviles con ancho de banda limitado).
- **Under-fetching / N+1 de red**: con REST, pintar una pantalla puede requerir 5 llamadas encadenadas. GraphQL las junta en una.

Pero traslada complejidad al servidor: el **caching es más difícil** (no hay URLs cacheables por recurso; todo va por POST a `/graphql`), aparece el problema **N+1 en la resolución** (hay que resolverlo con *dataloaders* que agrupan consultas), y un cliente puede pedir una query muy costosa (hay que limitar profundidad/complejidad). **Su nicho: APIs orientadas a clientes ricos y variados** (apps, webs) con necesidades de datos heterogéneas, como una *Backend-for-Frontend*.

### La Tabla de Decisión

| | REST | gRPC | GraphQL |
|--|------|------|---------|
| **Formato** | JSON/texto | Binario (Protobuf) | JSON |
| **Transporte** | HTTP/1.1+ | HTTP/2 | HTTP (POST) |
| **Contrato** | OpenAPI (opcional) | Protobuf (obligatorio) | Schema (obligatorio) |
| **Rendimiento** | Bueno | Excelente | Bueno (variable) |
| **Navegador** | Nativo | Necesita proxy | Nativo |
| **Caching HTTP** | Excelente (por URL) | Limitado | Difícil |
| **Mejor para** | APIs públicas, CRUD | Servicio↔servicio interno | Clientes ricos/variados (BFF) |

Heurística práctica:
- **API pública o CRUD estándar** → REST. Es lo que todo el mundo entiende, cachea bien, sin tooling especial.
- **Microservicios internos de alto tráfico** → gRPC. Rendimiento y contrato fuerte.
- **Agregar datos para apps/frontends con necesidades cambiantes** → GraphQL (a menudo como capa BFF sobre servicios REST/gRPC).

No es excluyente: es habitual REST de cara al público, gRPC entre servicios internos y GraphQL como gateway para los clientes. La pregunta correcta no es "¿cuál es mejor?" sino "¿qué problema tengo?".

### El Contrato como Fuente de Verdad

El hilo común a los tres: **el contrato debe ser explícito y la fuente de verdad**, no algo implícito que vive en la cabeza de quien escribió el endpoint.

- **REST → OpenAPI** (antes Swagger): describe en YAML/JSON los endpoints, parámetros, esquemas y respuestas. De ahí se generan documentación interactiva, clientes (SDKs), validación de servidor y mocks. En FastAPI, el esquema OpenAPI se genera **automáticamente** desde los type hints y modelos Pydantic — el contrato y el código no se desincronizan.
- **gRPC → Protobuf**: el `.proto` *es* el contrato; cliente y servidor se generan de él, imposible desincronizarse.
- **GraphQL → Schema**: el SDL define los tipos disponibles, y las herramientas validan queries contra él.

```python
# FastAPI genera OpenAPI desde los modelos — el contrato sale del código
from pydantic import BaseModel
from fastapi import FastAPI

app = FastAPI()

class CrearPedido(BaseModel):
    cliente_id: str
    items: list[str]

@app.post("/pedidos", status_code=201)
def crear(payload: CrearPedido) -> dict:
    ...   # /docs muestra el esquema OpenAPI interactivo automáticamente
```

### Contract Testing

Cuando un servicio (consumidor) depende de otro (proveedor), el riesgo es que el proveedor cambie y rompa al consumidor sin que nadie lo note hasta producción. El **contract testing** (con herramientas como Pact) verifica en CI que el proveedor sigue cumpliendo lo que el consumidor espera, **sin levantar ambos servicios juntos**. Es la pieza que falta entre los tests unitarios (un servicio aislado) y los e2e (todo el sistema, lentos y frágiles) — lo retomamos en [[Desarrollo Profesional/Estrategia de Testing/Páginas/02 - Tipos de Test y Estrategia|Estrategia de Testing 02]].

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - **gRPC**: contrato Protobuf (genera cliente/servidor), binario sobre HTTP/2, streaming nativo. Rápido y estricto, pero no legible ni directo desde navegador. Nicho: **servicio-a-servicio interno**.
> - **GraphQL**: el cliente pide exactamente los campos que necesita en una llamada, resolviendo over/under-fetching. A cambio: caching difícil, problema N+1 (dataloaders) y queries costosas (limitar complejidad). Nicho: **clientes ricos/variados (BFF)**.
> - **REST** sigue siendo el mejor defecto para **APIs públicas y CRUD**: universal, cacheable, sin tooling especial.
> - No es excluyente: REST público + gRPC interno + GraphQL gateway conviven. La pregunta es "¿qué problema tengo?", no "¿cuál gana?".
> - El **contrato explícito** (OpenAPI, Protobuf, Schema) es la fuente de verdad: genera docs, clientes y validación. FastAPI lo deriva del código. **Contract testing** (Pact) evita que un proveedor rompa a sus consumidores sin avisar.

### Para llevar a la práctica
- [ ] Para una comunicación interna de alto tráfico que tengas en REST/JSON, evalúa si gRPC reduciría latencia y errores de integración.
- [ ] Si tus clientes hacen muchas llamadas encadenadas o reciben datos que no usan, considera una capa GraphQL/BFF.
- [ ] Asegúrate de que tus APIs REST publican un OpenAPI actualizado (en FastAPI es automático; revísalo).
- [ ] Donde un servicio dependa de otro, introduce contract testing para no romper en producción.

### Recursos
- 🌐 grpc.io/docs — documentación de gRPC y Protobuf
- 🌐 graphql.org/learn — la guía oficial de GraphQL
- 🌐 spec.openapis.org — especificación OpenAPI
- 🌐 docs.pact.io — contract testing con Pact
- 📖 *API Design Patterns* — JJ Geewax

---
`#api` `#grpc` `#graphql` `#openapi` `#contract-testing`
