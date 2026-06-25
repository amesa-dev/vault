# 🔗 RESTful 03 — HATEOAS y Madurez REST

[[Desarrollo Profesional/RESTful/RESTful|⬅️ Volver a RESTful]] | [[Desarrollo Profesional/RESTful/Páginas/02 - Diseño de Recursos y Operaciones|← 02]]

> [!abstract] Introducción
> Llegamos a la parte de REST que casi nadie implementa y casi todos malinterpretan: **HATEOAS**. Para Fielding, una API sin hipermedia simplemente no es REST. En la práctica, el pragmatismo manda y la mayoría se queda un escalón por debajo. Esta página explica qué es HATEOAS de verdad, lo sitúa en el modelo de madurez de Richardson y cierra con lo que sí importa siempre: cómo hacer evolucionar el contrato sin romper a tus consumidores.

## ¿De qué vamos a hablar?

De HATEOAS y la hipermedia, del modelo de madurez de Richardson como mapa, y de cómo versionar y evolucionar una API RESTful sin causar dolor.

### Conceptos que vamos a cubrir
- HATEOAS: las respuestas que te dicen qué puedes hacer después
- El modelo de madurez de Richardson (niveles 0 a 3)
- Por qué HATEOAS rara vez se adopta y cuándo merece la pena
- Versionado y evolución del contrato sin romper

---

## El Concepto

### HATEOAS — Hypermedia as the Engine of Application State

La idea: las respuestas de la API no solo traen datos, traen **enlaces** que indican las acciones y transiciones posibles desde el estado actual del recurso. El cliente navega la API siguiendo esos enlaces, igual que tú navegas una web haciendo clic, sin hardcodear cada URL.

```json
// GET /pedidos/42  →  no solo datos, también qué se puede hacer ahora
{
  "id": 42,
  "estado": "pendiente_pago",
  "total": 100.0,
  "_links": {
    "self":      { "href": "/pedidos/42" },
    "pagar":     { "href": "/pedidos/42/pago",          "method": "POST" },
    "cancelar":  { "href": "/pedidos/42/cancelacion",   "method": "POST" }
  }
}
```

Cuando el pedido ya está pagado, la representación cambia: desaparece el enlace `pagar` y aparece `enviar` o `factura`. El cliente no necesita saber las reglas de negocio ("¿puedo pagar?") — le basta con mirar si el enlace está presente. Eso es la "hipermedia como motor del estado de la aplicación": el estado de lo que el cliente puede hacer lo dirigen los enlaces que el servidor incluye.

**Lo que te da:** desacoplamiento real. El servidor puede cambiar URLs o reglas de transición sin romper al cliente, porque el cliente nunca hardcodea rutas: las descubre. Formatos estándar para esto: HAL, JSON:API, Siren.

### El Modelo de Madurez de Richardson

Leonard Richardson describió cuatro niveles que sirven de mapa para saber "cómo de RESTful" es una API:

```
Nivel 0 ── El Pantano del POX
           Un único endpoint, todo por POST. HTTP como mero túnel. (RPC disfrazado)

Nivel 1 ── Recursos
           URLs distintas por recurso (/pedidos/42), pero todo sigue siendo POST.

Nivel 2 ── Verbos HTTP
           GET/POST/PUT/DELETE con su semántica + códigos de estado correctos.
           ◀── AQUÍ está la inmensa mayoría de las "APIs REST" buenas.

Nivel 3 ── HATEOAS
           Las respuestas incluyen hipermedia. Para Fielding, esto SÍ es REST.
```

La trampa: para Fielding, **solo el nivel 3 es REST de verdad**; los niveles 0-2 son grados de "HTTP bien usado". En la industria, en cambio, se llama "REST" al nivel 2, y eso es lo que casi todo el mundo construye.

### Por Qué HATEOAS Rara Vez se Usa

Si es tan elegante, ¿por qué casi nadie llega al nivel 3? Razones honestas:
- **Los clientes no lo aprovechan.** El sueño es un cliente genérico que descubre la API navegando enlaces. En la realidad, los clientes (apps móviles, frontends) se programan contra una API concreta y conocida, y siguen hardcodeando rutas aunque existan los `_links`.
- **Más complejidad en ambos lados.** Generar y consumir hipermedia añade trabajo que muchos equipos no recuperan.
- **El nivel 2 ya resuelve el 95%.** Para la mayoría de APIs, verbos + códigos + buen diseño de recursos es el punto dulce entre rigor y pragmatismo.

**Cuándo SÍ merece la pena:** APIs públicas de muy larga vida y muchos consumidores que no controlas, donde poder cambiar el servidor sin coordinar con cada cliente tiene un valor enorme (algunas APIs de pagos y gobierno lo usan). Para una API interna entre tus propios servicios, casi nunca compensa.

> [!tip] Recomendación pragmática
> Apunta a un **nivel 2 impecable**: recursos bien nombrados, verbos correctos, códigos con significado, errores consistentes y paginación. Añade hipermedia de forma selectiva (por ejemplo, enlaces de paginación `next`/`prev`, que sí aportan sin gran coste) y reserva el HATEOAS completo para cuando el desacoplamiento extremo lo justifique.

### Versionado y Evolución del Contrato

Esto sí importa siempre. Una API publicada es un **contrato**; romperlo rompe a todos sus consumidores. La distinción clave:

- **Cambios compatibles** (no rompen): añadir un campo opcional, un endpoint nuevo, un valor de enum nuevo. Los clientes viejos los ignoran → no necesitan versión nueva.
- **Cambios incompatibles** (rompen): quitar/renombrar un campo, cambiar su tipo, hacer obligatorio algo que era opcional, cambiar la semántica → requieren versión nueva.

Estrategias de versionado:
- **En la URL** (`/v1/pedidos`, `/v2/pedidos`): explícito, fácil de enrutar y cachear. El más usado.
- **En cabecera** (`Accept: application/vnd.miempresa.v2+json`): más "purista" REST (la URI del recurso no cambia), pero menos visible.

El principio rector es la **regla de robustez** (ley de Postel): *sé estricto en lo que envías y tolerante en lo que recibes*. Diseña para extender: que añadir sea fácil y barato, y que quitar sea raro y costoso. Si tienes que romper, anuncia la deprecación, mantén la versión vieja un tiempo y comunica la migración. El detalle de paginación y errores está en [[Desarrollo Profesional/Diseño de APIs/Páginas/01 - REST y Diseño de Recursos|Diseño de APIs 01]]; documenta el contrato con **OpenAPI** (ver [[Desarrollo Profesional/Diseño de APIs/Páginas/02 - gRPC GraphQL y Alternativas|Diseño de APIs 02]]).

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - **HATEOAS** hace que las respuestas incluyan enlaces a las acciones posibles desde el estado actual; el cliente navega la API en vez de hardcodear rutas. Es la hipermedia como motor del estado.
> - El **modelo de Richardson** va del nivel 0 (RPC sobre HTTP) al 3 (HATEOAS). La industria llama "REST" al **nivel 2**; para Fielding, solo el 3 lo es.
> - HATEOAS **casi no se adopta** porque los clientes reales no lo aprovechan y el nivel 2 ya cubre el 95%. Reserva el nivel 3 para APIs públicas de larga vida con consumidores que no controlas.
> - Lo que importa siempre: **versionar y evolucionar** el contrato. Añadir campos es compatible; quitar/renombrar/cambiar tipo rompe y exige versión nueva.
> - Sigue la **regla de robustez** y diseña para extender; versiona en URL (`/v2`) o en cabecera.

### Para llevar a la práctica
- [ ] Sitúa tu API en el modelo de Richardson: ¿en qué nivel estás realmente?
- [ ] Añade enlaces de paginación (`next`/`prev`) a una colección: HATEOAS ligero y útil
- [ ] Clasifica un cambio que quieras hacer como compatible o incompatible antes de tocarlo
- [ ] Asegura que tu contrato está documentado en OpenAPI y versionado de forma explícita

### Recursos
- 🌐 martinfowler.com/articles/richardsonMaturityModel.html — el modelo de madurez, explicado por Fowler
- 📖 *RESTful Web APIs* — Richardson & Amundsen (el libro de referencia sobre hipermedia)
- 🌐 stateless.group/hal_specification.html — HAL, un formato de hipermedia sencillo
- 📄 RFC 8288 — Web Linking (la cabecera `Link` estándar)

---
`#rest` `#restful` `#hateoas` `#richardson` `#versionado`
