# 🔗 RESTful 02 — Diseño de Recursos y Operaciones

[[Desarrollo Profesional/RESTful/RESTful|⬅️ Volver a RESTful]] | [[Desarrollo Profesional/RESTful/Páginas/01 - Principios y Restricciones|← 01]] | [[Desarrollo Profesional/RESTful/Páginas/03 - HATEOAS y Madurez REST|03 →]]

> [!abstract] Introducción
> Con la teoría de Fielding en la cabeza, toca diseñar. Las decisiones que más marcan la vida diaria de una API son mundanas: cómo nombras los recursos, qué verbo usas para cada cosa, qué código de estado devuelves y cómo manejas las peticiones que llegan dos veces. Aquí están las reglas prácticas que hacen que una API se entienda sola y aguante el paso del tiempo.

## ¿De qué vamos a hablar?

Del diseño concreto y operativo de una API RESTful: recursos, URIs, métodos HTTP con su semántica exacta, códigos de estado, negociación de contenido e idempotencia.

### Conceptos que vamos a cubrir
- Recursos como sustantivos y diseño de URIs jerárquicas
- Los métodos HTTP y sus propiedades: seguro vs idempotente
- Acciones que no son CRUD sin ensuciar las URLs
- Negociación de contenido y representaciones
- Idempotencia en la práctica con claves de idempotencia

---

## El Concepto

### Recursos y URIs

Las URIs nombran **recursos** (sustantivos, normalmente en plural); la acción la pone el método HTTP, nunca la URL.

```
GET    /pedidos                  → colección de pedidos
POST   /pedidos                  → crear un pedido
GET    /pedidos/42               → un pedido concreto
PATCH  /pedidos/42               → modificar parcialmente el pedido 42
DELETE /pedidos/42               → eliminar el pedido 42
GET    /clientes/7/pedidos       → pedidos del cliente 7 (relación jerárquica)
```

Reglas que evitan el 90% de las APIs feas:
- **Sustantivos, no verbos**: `POST /pedidos`, nunca `POST /crearPedido`. El verbo lo da HTTP.
- **Plural y coherente**: elige plural (`/pedidos/42`) y mantenlo en toda la API.
- **Jerarquía para relaciones**: `/clientes/7/pedidos` expresa "los pedidos de ese cliente". No anides más de 1-2 niveles: a partir de ahí, mejor un filtro (`/pedidos?cliente_id=7`).
- **Minúsculas y guiones**: `/ordenes-de-compra`, no `/OrdenesDeCompra`.

### Los Métodos HTTP y sus Propiedades

Dos propiedades gobiernan el uso correcto de los verbos:
- **Seguro**: no modifica el estado del servidor (solo lee).
- **Idempotente**: ejecutarlo N veces deja el sistema en el mismo estado que ejecutarlo una vez.

| Método | Uso | Seguro | Idempotente |
|---|---|:---:|:---:|
| `GET` | Leer un recurso | ✅ | ✅ |
| `HEAD` | Como GET pero solo cabeceras | ✅ | ✅ |
| `POST` | Crear / acción no idempotente | ❌ | ❌ |
| `PUT` | Reemplazar por completo | ❌ | ✅ |
| `PATCH` | Modificar parcialmente | ❌ | ❌* |
| `DELETE` | Eliminar | ❌ | ✅ |

`PUT` reemplaza el recurso entero, así que repetirlo deja el mismo resultado (idempotente). `DELETE` también: borrar dos veces deja el recurso igual de borrado. `POST` crea uno nuevo cada vez: **no** es idempotente, y de ahí nacen los cobros duplicados si el cliente reintenta. (`PATCH` puede diseñarse idempotente, pero no se garantiza.)

```python
# La diferencia PUT vs PATCH en FastAPI
@app.put("/pedidos/{pid}")          # reemplazo completo: el cuerpo es el recurso entero
def reemplazar(pid: int, pedido: PedidoCompleto):
    db.reemplazar(pid, pedido)
    return pedido

@app.patch("/pedidos/{pid}")        # modificación parcial: solo los campos enviados
def modificar(pid: int, cambios: PedidoParcial):
    actual = db.obtener(pid)
    actualizado = actual.copy(update=cambios.dict(exclude_unset=True))
    db.guardar(actualizado)
    return actualizado
```

### Acciones que no son CRUD

No todo encaja en crear/leer/actualizar/borrar. "Confirmar un pedido", "publicar un artículo", "reembolsar un pago" son transiciones de estado. La forma RESTful de modelarlas es **reificar la acción como un sub-recurso** en lugar de meter un verbo en la URL:

```
# MAL: verbos sueltos en la URL
POST /pedidos/42/confirmar
POST /pedidos/42/cancelar

# BIEN: la acción se modela como un sub-recurso/transición
POST /pedidos/42/confirmacion      → crea la confirmación del pedido 42
POST /pedidos/42/cancelacion       { "motivo": "..." }
PUT  /articulos/9/estado           { "estado": "publicado" }
```

Encaja de maravilla con [[Desarrollo Profesional/CQRS/Páginas/02 - Implementación en Python|los commands de CQRS]]: el endpoint traduce ese `POST` a un command `ConfirmarPedido` y delega.

### Códigos de Estado

Usa los códigos HTTP con su significado, no `200` para todo. Hay una tabla completa con el manejo de errores (Problem Details, RFC 9457) en [[Desarrollo Profesional/Diseño de APIs/Páginas/01 - REST y Diseño de Recursos|Diseño de APIs 01]]; lo esencial:

- **2xx éxito**: `200 OK`, `201 Created` (con cabecera `Location` al nuevo recurso), `202 Accepted` (aceptado pero aún procesando — típico en CQRS asíncrono), `204 No Content` (DELETE correcto).
- **4xx error del cliente**: `400` (malformada), `401` (no autenticado), `403` (sin permiso), `404` (no existe), `409` (conflicto de estado), `422` (semánticamente inválido), `429` (rate limit).
- **5xx error del servidor**: `500` (fallo interno), `503` (no disponible).

### Negociación de Contenido

Como un recurso puede tener varias representaciones, el cliente pide la que quiere con la cabecera `Accept`, y el servidor responde con `Content-Type`:

```
GET /pedidos/42
Accept: application/json          → el servidor devuelve JSON
Accept: application/xml           → el mismo recurso, en XML
Accept-Language: es               → la representación en español
```

Esto es la sub-restricción de "manipulación mediante representaciones" en acción: el recurso `/pedidos/42` es uno solo; sus representaciones, varias.

### Idempotencia en la Práctica

Los clientes reintentan ante fallos de red (ver [[Desarrollo Profesional/Patrones de Resiliencia/Páginas/01 - Timeouts Reintentos y Backoff|timeouts y reintentos]]). Con `POST` no idempotente, un reintento puede crear un cobro duplicado. La solución estándar es la cabecera **`Idempotency-Key`**: el cliente genera una clave única por operación; el servidor recuerda el resultado de cada clave y, si la ve repetida, devuelve el resultado anterior sin volver a ejecutar.

```python
@app.post("/pagos", status_code=201)
def crear_pago(pago: PagoIn, idempotency_key: str = Header(...)):
    previo = store.obtener(idempotency_key)
    if previo is not None:
        return previo                      # reintento: devolvemos el mismo resultado
    resultado = procesar_pago(pago)        # se ejecuta UNA sola vez por clave
    store.guardar(idempotency_key, resultado)
    return resultado
```

Es exactamente el mecanismo que usan Stripe y similares. La deduplicación de escrituras críticas se cruza con [[Desarrollo Profesional/Sistemas Distribuidos/Páginas/03 - Transacciones Distribuidas|idempotencia en sistemas distribuidos]].

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - Las **URIs nombran recursos** (sustantivos en plural); el verbo lo pone HTTP. Usa jerarquía para relaciones y filtros para lo demás.
> - Domina **seguro vs idempotente**: `GET` seguro e idempotente; `PUT`/`DELETE` idempotentes; `POST` ni lo uno ni lo otro (de ahí los duplicados).
> - Las **acciones no-CRUD** se modelan como sub-recursos (`POST /pedidos/42/confirmacion`), no como verbos en la URL; encajan con los commands de CQRS.
> - Usa los **códigos de estado** con su semántica (incluido `202 Accepted` para procesos asíncronos) y errores con cuerpo consistente.
> - La **negociación de contenido** (`Accept`/`Content-Type`) sirve distintas representaciones del mismo recurso.
> - La **idempotencia** en escrituras críticas se resuelve con `Idempotency-Key` porque los clientes reintentan.

### Para llevar a la práctica
- [ ] Audita tus URLs: ¿hay verbos donde debería haber sustantivos? Renómbralos
- [ ] Revisa que cada endpoint use el método correcto según seguro/idempotente
- [ ] Convierte una acción tipo "confirmar/publicar" a un sub-recurso RESTful
- [ ] Añade `Idempotency-Key` a tu endpoint de escritura más crítico

### Recursos
- 🌐 cloud.google.com/apis/design — Google API Design Guide (recursos, métodos, nombres)
- 🌐 stripe.com/docs/api/idempotent_requests — idempotencia bien explicada por quien la popularizó
- 📄 RFC 9110 — HTTP Semantics (la referencia de métodos y códigos)
- 🌐 restfulapi.net/http-methods — métodos HTTP y sus propiedades

---
`#rest` `#restful` `#http` `#recursos` `#idempotencia`
