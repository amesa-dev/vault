# 🔌 APIs 01 — REST y Diseño de Recursos

[[Desarrollo Profesional/Diseño de APIs/Diseño de APIs|⬅️ Volver a Diseño de APIs]] | [[Desarrollo Profesional/Diseño de APIs/Páginas/02 - gRPC GraphQL y Alternativas|02 →]]

> [!abstract] Introducción
> REST es el estilo de API más extendido, y también el más malentendido: la mayoría de las "APIs REST" son en realidad APIs HTTP que no siguen los principios de Roy Fielding. Esta página explica qué es REST de verdad (el modelo de madurez de Richardson), cómo diseñar recursos y URLs que se entiendan solos, y las decisiones que más impacto tienen en la vida de una API: el versionado, la paginación, el manejo de errores y la idempotencia.

## ¿De qué vamos a hablar?

Cómo diseñar una API HTTP que sea intuitiva, consistente y capaz de evolucionar sin romper a sus consumidores.

### Conceptos que vamos a cubrir
- REST: recursos, verbos y el modelo de madurez de Richardson
- Diseño de URLs y uso correcto de los métodos HTTP
- Códigos de estado y manejo de errores
- Versionado: cómo evolucionar sin romper
- Paginación, filtrado e idempotencia

---

## El Concepto

### Qué es REST (de Verdad)

REST (Representational State Transfer) modela tu sistema como **recursos** (sustantivos: un pedido, un usuario) que se manipulan con un conjunto fijo de **verbos** (los métodos HTTP). La idea central: las URLs identifican recursos, no acciones. El **modelo de madurez de Richardson** describe los niveles:

- **Nivel 0**: HTTP como mero túnel. Un solo endpoint, todo por POST. (RPC sobre HTTP, no REST.)
- **Nivel 1 — Recursos**: URLs distintas por recurso (`/pedidos/42`), pero aún sin usar bien los verbos.
- **Nivel 2 — Verbos HTTP**: usas GET/POST/PUT/DELETE con su semántica y los códigos de estado correctos. **Aquí está la mayoría de las "APIs REST" buenas en la práctica.**
- **Nivel 3 — HATEOAS**: las respuestas incluyen enlaces a las acciones posibles, de modo que el cliente navega la API sin hardcodear URLs. Elegante pero raro en la práctica.

Apunta al **nivel 2**: es el punto dulce entre rigor y pragmatismo.

### Diseño de URLs y Verbos

Las URLs nombran recursos (sustantivos en plural); los verbos HTTP expresan la acción:

```
GET    /pedidos              → lista de pedidos (colección)
POST   /pedidos              → crea un pedido nuevo
GET    /pedidos/42           → un pedido concreto
PUT    /pedidos/42           → reemplaza el pedido 42 completo
PATCH  /pedidos/42           → modifica parcialmente el pedido 42
DELETE /pedidos/42           → borra el pedido 42
GET    /pedidos/42/lineas    → las líneas de ese pedido (recurso anidado)
```

Reglas:
- **Sustantivos, no verbos en la URL**: `POST /pedidos`, no `POST /crearPedido`. La acción la da el método.
- **Propiedades correctas de los métodos**: GET es **seguro** (no muta) e **idempotente** (repetirlo no cambia nada). PUT y DELETE son idempotentes (borrar dos veces deja el mismo estado). POST **no** es idempotente (crea uno nuevo cada vez) — de ahí la importancia de las `Idempotency-Key` en POST de cobros.
- **Las acciones que no son CRUD** (ej. "confirmar un pedido") se modelan como un sub-recurso o una transición de estado: `POST /pedidos/42/confirmacion`, evitando verbos sueltos en la URL.

### Códigos de Estado y Errores

Usa los códigos HTTP con su significado, no `200` para todo:

| Código | Significado | Cuándo |
|--------|-------------|--------|
| `200 OK` | éxito | GET, PUT, PATCH correctos |
| `201 Created` | recurso creado | POST que crea (devuelve `Location`) |
| `204 No Content` | éxito sin cuerpo | DELETE correcto |
| `400 Bad Request` | petición malformada | validación fallida |
| `401 Unauthorized` | no autenticado | falta o es inválido el token |
| `403 Forbidden` | autenticado pero sin permiso | autorización fallida |
| `404 Not Found` | no existe (o no debes saberlo) | recurso inexistente |
| `409 Conflict` | conflicto de estado | versión obsoleta, duplicado |
| `422 Unprocessable` | semánticamente inválido | bien formado pero viola reglas |
| `429 Too Many Requests` | rate limit | con cabecera `Retry-After` |
| `500 / 503` | error del servidor | fallo interno / no disponible |

Los **errores deben tener un cuerpo consistente y útil**. El estándar **RFC 9457 (Problem Details)** define un formato común:

```python
# Respuesta de error consistente (Problem Details, RFC 9457)
{
    "type": "https://api.miempresa.com/errors/saldo-insuficiente",
    "title": "Saldo insuficiente",
    "status": 422,
    "detail": "La cuenta 42 tiene 50€ pero la operación requiere 100€",
    "instance": "/cuentas/42/transferencias",
    "saldo_disponible": 50.0      # campos extra específicos del error
}
```

Un buen error dice **qué pasó, por qué y qué puede hacer el cliente** — nunca filtra stack traces ni detalles internos (eso es una fuga de información, ver [[Desarrollo Profesional/Seguridad Aplicada/Páginas/01 - OWASP y Vulnerabilidades|Seguridad 01]]).

### Versionado: Evolucionar sin Romper

Una API publicada es un contrato; cambiarla de forma incompatible rompe a todos sus consumidores. La distinción clave:

- **Cambios compatibles** (no rompen): añadir un campo opcional, un endpoint nuevo, un valor de enum nuevo. Los clientes viejos los ignoran. Se pueden hacer sin versionar.
- **Cambios incompatibles** (rompen): quitar/renombrar un campo, cambiar un tipo, hacer obligatorio algo opcional, cambiar la semántica. **Requieren una versión nueva.**

Estrategias de versionado:
- **En la URL** (`/v1/pedidos`, `/v2/pedidos`): explícito, fácil de enrutar y cachear. El más usado.
- **En cabecera** (`Accept: application/vnd.miempresa.v2+json`): más "purista" REST, URLs estables, pero menos visible.

Principio rector: **diseña para la extensión**. Haz que añadir sea fácil y quitar sea raro. Un cliente nunca debe romperse porque añadiste un campo (regla de robustez: sé estricto en lo que envías, tolerante en lo que recibes).

### Paginación, Filtrado e Idempotencia

- **Paginación**: nunca devuelvas una colección sin límite (un `GET /pedidos` con 10M de filas tumba el servidor). Dos enfoques:
  - **Offset** (`?page=2&size=20`): simple, permite saltar a cualquier página, pero se vuelve lento y **inconsistente** si los datos cambian (un insert desplaza todo). 
  - **Cursor / keyset** (`?after=<id_o_token>`): pasas un puntero al último visto. Estable ante cambios y eficiente a escala. **Preferible para datasets grandes o que cambian.**
- **Filtrado y orden**: por query params (`?estado=pendiente&sort=-fecha`), documentados y validados.
- **Idempotencia en escrituras**: para POST que no deben ejecutarse dos veces (un cobro), acepta una cabecera `Idempotency-Key` y deduplica en el servidor (ver [[Desarrollo Profesional/Sistemas Distribuidos/Páginas/03 - Transacciones Distribuidas|Transacciones Distribuidas]]). Esencial porque los clientes reintentan ([[Desarrollo Profesional/Patrones de Resiliencia/Páginas/01 - Timeouts Reintentos y Backoff|Resiliencia 01]]).

```python
# Paginación por cursor con FastAPI — estable ante cambios
@app.get("/pedidos")
def listar(after: str | None = None, limit: int = 20):
    limit = min(limit, 100)               # tope: nunca dejes que el cliente pida ∞
    pedidos = db.pedidos_despues_de(after, limit + 1)
    hay_mas = len(pedidos) > limit
    pedidos = pedidos[:limit]
    return {
        "datos": pedidos,
        "siguiente_cursor": pedidos[-1].id if hay_mas else None,
    }
```

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - REST modela **recursos** (sustantivos) manipulados con **verbos HTTP**. El **modelo de Richardson** va de RPC disfrazado (nivel 0) a HATEOAS (nivel 3); apunta al **nivel 2** (verbos + códigos correctos).
> - URLs con sustantivos en plural (`POST /pedidos`, no `/crearPedido`). Respeta las propiedades: GET seguro e idempotente, PUT/DELETE idempotentes, **POST no** (de ahí las claves de idempotencia).
> - Usa los **códigos de estado** con su significado y errores con cuerpo consistente (**Problem Details, RFC 9457**) que digan qué, por qué y qué hacer — sin filtrar internals.
> - **Versionado**: añadir campos es compatible (no versiones); quitar/renombrar/cambiar tipo rompe (versiona, en URL `/v2` o en cabecera). Diseña para la extensión.
> - **Paginación** siempre (cursor/keyset > offset para datos grandes), filtrado por query params validados, e **idempotencia** en escrituras críticas porque los clientes reintentan.

### Para llevar a la práctica
- [ ] Revisa una API tuya: ¿las URLs son sustantivos? ¿usa los códigos de estado correctos o `200` para todo?
- [ ] Comprueba que tus colecciones están paginadas y con un límite máximo forzado por el servidor.
- [ ] Unifica el formato de error de tu API hacia Problem Details (RFC 9457).
- [ ] Identifica los endpoints POST que no deben ejecutarse dos veces y añádeles `Idempotency-Key`.

### Recursos
- 📖 *API Design Patterns* — JJ Geewax (Google)
- 🌐 cloud.google.com/apis/design — Google API Design Guide (excelente y pragmática)
- 🌐 martinfowler.com/articles/richardsonMaturityModel.html
- 📄 RFC 9457 — Problem Details for HTTP APIs
- 🌐 stripe.com/docs/api — referencia de una API REST ejemplar

---
`#api` `#rest` `#versionado` `#paginacion` `#http`
