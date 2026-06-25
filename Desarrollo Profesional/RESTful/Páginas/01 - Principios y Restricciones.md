# 🔗 RESTful 01 — Principios y Restricciones

[[Desarrollo Profesional/RESTful/RESTful|⬅️ Volver a RESTful]] | [[Desarrollo Profesional/RESTful/Páginas/02 - Diseño de Recursos y Operaciones|02 →]]

> [!abstract] Introducción
> Roy Fielding no inventó un protocolo: describió, en su tesis doctoral de 2000, **por qué la web funciona tan bien a escala planetaria** y destiló esas propiedades en un estilo arquitectónico llamado REST. La clave es que REST es un conjunto de **restricciones**: cada una te quita libertad a cambio de una propiedad deseable (escalabilidad, desacoplamiento, evolución). Entenderlas es lo que separa "hago peticiones HTTP con JSON" de "diseño una API RESTful".

## ¿De qué vamos a hablar?

De qué es REST como estilo arquitectónico y de las seis restricciones que lo definen, con foco en qué te aporta cada una y qué se rompe si la incumples.

### Conceptos que vamos a cubrir
- REST como estilo, no como tecnología
- Las 6 restricciones: cliente-servidor, stateless, cacheable, interfaz uniforme, sistema en capas, code-on-demand
- Recurso, identificador y representación: el vocabulario fundamental
- Por qué la statelessness es la restricción que más impacto tiene

---

## El Concepto

### REST es un Estilo, no una Tecnología

Un **estilo arquitectónico** es un conjunto de restricciones sobre cómo se organizan los componentes de un sistema. REST (Representational State Transfer) no es HTTP, ni JSON, ni un framework: es un patrón que *suele* implementarse sobre HTTP porque HTTP nació de las mismas ideas. Puedes usar HTTP de forma nada RESTful (un único `POST /api` que recibe un campo `action`) y, en teoría, hacer REST sobre otro protocolo.

El vocabulario básico:
- **Recurso**: cualquier cosa nombrable e interesante para el cliente (un pedido, un usuario, la colección de pedidos de un cliente). Es un concepto, no una fila de una tabla.
- **Identificador (URI)**: cada recurso tiene una URI estable que lo identifica (`/pedidos/42`).
- **Representación**: lo que el cliente recibe y envía *no es el recurso*, sino una representación de su estado en un formato concreto (un JSON, un XML). El mismo recurso puede tener varias representaciones.

Ese es el "Transfer of Representational State" del nombre: el cliente y el servidor se pasan representaciones del estado de los recursos.

### Las Seis Restricciones

REST se define por seis restricciones. Las cinco primeras son obligatorias; la sexta es opcional.

```
1. Cliente-Servidor      ── separación de responsabilidades
2. Stateless             ── cada petición se basta a sí misma
3. Cacheable             ── las respuestas dicen si se pueden cachear
4. Interfaz uniforme     ── el corazón de REST (4 sub-restricciones)
5. Sistema en capas      ── el cliente no sabe con cuántos saltos habla
6. Code-on-demand (opc.) ── el servidor puede enviar código ejecutable
```

#### 1. Cliente-Servidor
Separas la interfaz de usuario (cliente) del almacenamiento y la lógica (servidor). Pueden evolucionar de forma independiente mientras respeten el contrato. Obvio hoy, pero es la base.

#### 2. Stateless (sin estado de sesión)
**El servidor no guarda estado de la conversación entre peticiones.** Cada petición debe llevar toda la información necesaria para entenderla (incluida la autenticación). No hay "sesión" en el servidor que recuerde quién eres o por dónde ibas.

```
# MAL (con estado de sesión en el servidor):
POST /login          → el servidor guarda "usuario 42 logueado" en memoria
GET  /mi-carrito     → depende de que el servidor recuerde la sesión

# BIEN (stateless):
GET /carritos/42  Authorization: Bearer <token>
   → cada petición se autoidentifica; cualquier instancia puede atenderla
```

Esta es la restricción con más consecuencias prácticas: si ningún servidor guarda estado de sesión, **cualquier instancia puede atender cualquier petición**, lo que hace trivial escalar horizontalmente y balancear carga (ver [[Desarrollo Profesional/Diseño de Sistemas/Páginas/02 - Escalar de Cero a Millones|escalar de cero a millones]]). El precio: cada petición es más "gorda" (repite el token, el contexto) y el estado de cliente vive en el cliente o en un almacén compartido como [[Desarrollo Profesional/Redis/Redis|Redis]].

#### 3. Cacheable
Cada respuesta debe declararse, explícita o implícitamente, como cacheable o no (con cabeceras como `Cache-Control`, `ETag`). Si una respuesta es cacheable, el cliente o un intermediario pueden reutilizarla, eliminando viajes al servidor. Esto multiplica la escalabilidad y reduce latencia.

#### 4. Interfaz Uniforme — el Corazón de REST
Es la restricción que hace a REST, REST. Tiene cuatro sub-restricciones:
- **Identificación de recursos**: cada recurso se identifica por su URI.
- **Manipulación mediante representaciones**: el cliente actúa sobre el recurso enviando/recibiendo representaciones (un JSON con el nuevo estado), no llamando a métodos remotos.
- **Mensajes autodescriptivos**: cada mensaje incluye lo necesario para procesarlo (el método, el `Content-Type`, etc.).
- **HATEOAS** (Hypermedia as the Engine of Application State): las respuestas incluyen enlaces que indican qué puede hacer el cliente a continuación. Lo desarrollamos en la [[Desarrollo Profesional/RESTful/Páginas/03 - HATEOAS y Madurez REST|página 03]].

La interfaz uniforme **desacopla** cliente y servidor: como todos los recursos se manipulan igual (mismos verbos, misma semántica), el cliente no necesita conocimiento especial de cada endpoint.

#### 5. Sistema en Capas
El cliente no sabe si habla directamente con el servidor o con un intermediario (un balanceador, un proxy, un CDN, un gateway). Cada capa solo conoce la inmediata. Esto permite meter caches, balanceadores y capas de seguridad sin que el cliente se entere — la base de cualquier arquitectura web moderna.

#### 6. Code-on-Demand (opcional)
El servidor puede enviar código ejecutable al cliente (clásicamente, JavaScript) para extender su funcionalidad. Es la única restricción opcional y la menos relevante para el diseño de APIs.

### Por Qué Estas Restricciones Importan

Cada restricción es un intercambio deliberado: renuncias a flexibilidad para ganar una propiedad del sistema.

| Restricción | A qué renuncias | Qué ganas |
|---|---|---|
| Stateless | Comodidad de sesión en servidor | Escalado horizontal trivial, tolerancia a fallos |
| Cacheable | Control fino sobre cada respuesta | Latencia y carga drásticamente menores |
| Interfaz uniforme | Optimizar cada endpoint a medida | Desacoplamiento y evolución independiente |
| Sistema en capas | Visibilidad de toda la ruta | Poder meter proxies, caches y gateways |

Una API que viola la statelessness (guarda sesión en memoria del servidor) no escala bien horizontalmente, por muy "REST" que parezca su JSON. Las restricciones no son etiquetas: son las que producen las propiedades.

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - REST es un **estilo arquitectónico** (un conjunto de restricciones), no una tecnología. Suele ir sobre HTTP, pero HTTP mal usado puede no ser nada RESTful.
> - Vocabulario: un **recurso** (concepto nombrable) tiene un **identificador** (URI) y se intercambia mediante **representaciones** (JSON, XML…). El recurso no es la representación.
> - Las seis restricciones: **cliente-servidor**, **stateless**, **cacheable**, **interfaz uniforme** (el corazón, con HATEOAS dentro), **sistema en capas** y **code-on-demand** (opcional).
> - La **statelessness** es la de mayor impacto: sin sesión en el servidor, cualquier instancia atiende cualquier petición y escalar es trivial.
> - Cada restricción es un intercambio: cedes flexibilidad a cambio de escalabilidad, desacoplamiento o evolución.

### Para llevar a la práctica
- [ ] Revisa tu API: ¿guarda estado de sesión en el servidor? Si sí, no es stateless
- [ ] Comprueba qué respuestas declaran cacheabilidad (`Cache-Control`, `ETag`) y cuáles no
- [ ] Identifica los intermediarios entre tu cliente y tu servidor (proxy, LB, CDN): esa es la capa-5 en acción
- [ ] Para cada endpoint, pregúntate si manipulas un recurso mediante su representación o si estás haciendo RPC disfrazado

### Recursos
- 📄 Roy Fielding (2000) — *Architectural Styles and the Design of Network-based Software Architectures* (la tesis, capítulo 5)
- 🌐 ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm — el capítulo de REST en abierto
- 🌐 restfulapi.net — explicación accesible de las restricciones
- 📖 *RESTful Web APIs* — Leonard Richardson & Mike Amundsen

---
`#rest` `#restful` `#fielding` `#stateless` `#arquitectura`
