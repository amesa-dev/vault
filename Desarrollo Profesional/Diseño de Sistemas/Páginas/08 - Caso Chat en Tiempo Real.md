# 🏛️ Caso de Estudio — Chat en Tiempo Real

[[Desarrollo Profesional/Diseño de Sistemas/Diseño de Sistemas|⬅️ Volver a Diseño de Sistemas]] | [[Desarrollo Profesional/Diseño de Sistemas/Páginas/07 - Caso News Feed|← 07]] | [[Desarrollo Profesional/Diseño de Sistemas/Páginas/09 - Caso Sistema de Notificaciones|09 →]]

> [!abstract] Introducción
> Un sistema de chat (WhatsApp, Slack, Messenger) introduce un reto que los casos anteriores no tenían: la **comunicación en tiempo real bidireccional**. El servidor necesita *empujar* mensajes al receptor en el instante en que llegan, no esperar a que pregunte. Eso rompe el modelo petición-respuesta de HTTP y obliga a hablar de WebSockets, de cómo saber a qué servidor está conectado cada usuario, de la entrega y el orden de los mensajes, y de la presencia ("en línea"/"escribiendo…").

## ¿De qué vamos a hablar?

El diseño de un chat: cómo mantener conexiones en tiempo real, enrutar mensajes entre usuarios conectados a servidores distintos, y garantizar entrega, orden y presencia.

### Conceptos que vamos a cubrir
- El problema del tiempo real: polling vs WebSockets
- Servidores de chat con estado y el servicio de descubrimiento
- Flujo de un mensaje 1:1 y almacenamiento
- Entrega, orden y acuses (sent/delivered/read)
- Presencia y grupos

---

## El Diseño

### 1. Requisitos

**Funcionales**: mensajería 1:1 y de grupo en tiempo real, indicador de presencia (online/offline), acuses de recibo (enviado/entregado/leído), historial de mensajes, entrega a usuarios offline cuando vuelven.

**No funcionales**: **baja latencia** (tiempo real de verdad), entrega fiable (no perder mensajes), **orden** consistente de mensajes, y escala a cientos de millones de usuarios con muchísimas conexiones concurrentes.

### 2. El Problema del Tiempo Real

HTTP es petición-respuesta: el cliente pregunta, el servidor responde. Pero en un chat el servidor debe **iniciar** la comunicación (empujar el mensaje de Ana a Luis en cuanto llega). Opciones:

- **Short polling**: el cliente pregunta "¿hay algo nuevo?" cada X segundos. Simple pero ineficiente (mucho tráfico vacío) y con latencia (hasta X segundos de retraso).
- **Long polling**: el cliente pregunta y el servidor **mantiene la petición abierta** hasta que hay algo o expira. Mejor, pero sigue siendo un workaround.
- **WebSockets**: una conexión **persistente y bidireccional** sobre una sola conexión TCP. Tras un handshake, cliente y servidor se envían mensajes en cualquier dirección, en cualquier momento, con latencia mínima. **Es la solución estándar** para chat. La conexión queda abierta mientras el usuario está activo.

### 3. Servidores de Chat con Estado

Aquí aparece la complejidad. Con WebSockets, **cada usuario mantiene una conexión abierta con un servidor de chat concreto**. Esto significa que los servidores de chat **tienen estado** (las conexiones vivas) — lo contrario de los servidores web stateless de [[Desarrollo Profesional/Diseño de Sistemas/Páginas/02 - Escalar de Cero a Millones|la página 02]].

El problema central: **si Ana está conectada al servidor 1 y Luis al servidor 3, ¿cómo le llega a Luis el mensaje de Ana?** El servidor 1 necesita saber a qué servidor está conectado Luis. Solución: un **servicio de descubrimiento de presencia/sesiones** que mapea `usuario → servidor de chat al que está conectado`, guardado en un almacén rápido compartido (Redis).

```
Ana (conectada a Chat-Server-1) envía mensaje a Luis.
  1. Chat-Server-1 recibe el mensaje por el WebSocket de Ana.
  2. Persiste el mensaje (historial) y consulta el servicio de sesiones:
     "¿dónde está Luis?" → Chat-Server-3.
  3. Reenvía el mensaje a Chat-Server-3 (vía cola/PubSub interno).
  4. Chat-Server-3 lo empuja por el WebSocket de Luis. Entrega en tiempo real.
  5. Si Luis está offline → el mensaje queda persistido y se entrega al reconectar.
```

El reenvío entre servidores de chat se hace con un **bus de mensajes interno** (una cola o el Pub/Sub de [[Desarrollo Profesional/Redis/Páginas/02 - Patrones de Uso|Redis]], o Kafka): el servidor 1 publica "mensaje para Luis", el servidor 3 (suscrito a los mensajes de sus usuarios) lo recibe y lo entrega.

### 4. Almacenamiento de Mensajes

Volumen enorme de escrituras (cada mensaje), acceso por conversación, principalmente lecturas recientes. Características:
- Un **almacén optimizado para escritura masiva y acceso por clave de conversación**: las bases columnar-anchas (Cassandra, HBase) o clave-valor encajan mejor que SQL relacional aquí, por el volumen y el patrón de acceso ([[Desarrollo Profesional/Diseño de Sistemas/Páginas/03 - Almacenamiento Replicación y Sharding|almacenamiento]]). WhatsApp/Discord usan este tipo.
- **Shard key**: por `conversacion_id` (o `canal_id`), para que los mensajes de un chat estén juntos y se lean eficientemente.
- Los mensajes se ordenan dentro de la conversación por un **ID con orden temporal** (ver el siguiente punto).

### 5. Entrega, Orden y Acuses

- **Orden de mensajes**: como [[Desarrollo Profesional/Sistemas Distribuidos/Páginas/01 - Fundamentos y CAP|no hay reloj global fiable]], no ordenes por timestamp de cliente. Usa un **ID con orden** asignado por el servidor de la conversación (un secuenciador por chat, o IDs tipo Snowflake ordenables por tiempo, ver [[Desarrollo Profesional/Diseño de Sistemas/Páginas/11 - Caso Generador de IDs|Generador de IDs]]). Así todos ven los mensajes en el mismo orden.
- **Entrega fiable**: el mensaje se **persiste antes** de confirmarlo. Si el receptor está offline, queda almacenado y se entrega al reconectar (el cliente, al conectarse, pide "los mensajes desde el último ID que vi"). Es **at-least-once**: el cliente deduplica por ID de mensaje ([[Desarrollo Profesional/Sistemas Distribuidos/Páginas/03 - Transacciones Distribuidas|idempotencia]]).
- **Acuses (sent/delivered/read)**: son a su vez pequeños mensajes de estado. "Enviado" (el servidor lo recibió), "entregado" (llegó al dispositivo del receptor), "leído" (el receptor lo abrió). Cada transición se propaga de vuelta al emisor por el mismo canal.

### 6. Presencia y Grupos

- **Presencia (online/offline)**: cuando un usuario conecta su WebSocket, se marca online en el servicio de presencia (Redis con TTL); al desconectar (o al expirar un heartbeat), offline. Difundir el estado a todos sus contactos a cada cambio es caro a escala → se optimiza (actualizar bajo demanda, o solo a contactos activos). El "escribiendo…" es un evento efímero, fire-and-forget (Pub/Sub, sin persistir).
- **Grupos**: un mensaje a un grupo se reparte a todos los miembros (fan-out, como el [[Desarrollo Profesional/Diseño de Sistemas/Páginas/07 - Caso News Feed|news feed]]). Para grupos pequeños, push a cada miembro; para grupos enormes, las mismas consideraciones de fan-out aplican.

### 7. Cuellos de Botella y Mejoras

- **Millones de conexiones concurrentes**: cada WebSocket consume recursos; necesitas muchos servidores de chat y un balanceo consciente del estado de conexión.
- **El servicio de presencia/sesiones** es crítico y muy consultado → Redis replicado, muy disponible.
- **Reconexión**: clientes que cambian de red reconectan constantemente; el sistema debe reasignar la conexión y resincronizar mensajes perdidos por ID.
- **Cifrado extremo a extremo** (E2EE, como Signal/WhatsApp): el servidor solo enruta mensajes cifrados que no puede leer — añade gestión de claves pero no cambia la arquitectura de enrutado.

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - El chat necesita **tiempo real bidireccional**: el servidor empuja mensajes. La solución es **WebSockets** (conexión persistente bidireccional), no polling.
> - Los servidores de chat **tienen estado** (las conexiones vivas). El reto: si Ana y Luis están en servidores distintos, hace falta un **servicio de sesiones/presencia** (`usuario → servidor`) en Redis, y un **bus interno** (Pub/Sub/cola) para reenviar el mensaje al servidor del receptor.
> - **Almacenamiento**: volumen enorme y acceso por conversación → BD columnar/clave-valor sharded por `conversacion_id`, no SQL relacional.
> - **Orden**: no por timestamp de cliente (no hay reloj global) sino por un **ID con orden** asignado por el servidor (secuenciador o Snowflake). **Entrega at-least-once** con persistencia previa y deduplicación por ID; los offline reciben al reconectar.
> - **Acuses** (enviado/entregado/leído) son mini-mensajes de estado. **Presencia** vía heartbeat con TTL; "escribiendo…" es efímero (Pub/Sub sin persistir). Los **grupos** son fan-out como el feed.

### Para llevar a la práctica
- [ ] Explica por qué los servidores de chat no pueden ser stateless como los web, y qué problema crea eso.
- [ ] Diseña el servicio de sesiones: ¿qué guardas en Redis y cómo enrutas un mensaje entre dos servidores de chat?
- [ ] Razona cómo garantizarías el orden de mensajes en un grupo sin depender de relojes de cliente.
- [ ] Diseña la entrega a un usuario que estuvo offline: ¿cómo sabe qué mensajes le faltan al reconectar?

### Recursos
- 📖 *System Design Interview, Vol. 1* — Alex Xu (capítulo 12: Design a Chat System)
- 🌐 highscalability.com — la arquitectura de WhatsApp y Discord
- 📄 "How Discord Stores Trillions of Messages" — blog de Discord (Cassandra/ScyllaDB)
- 📄 Conexión con [[Desarrollo Profesional/Diseño de Sistemas/Páginas/11 - Caso Generador de IDs|IDs ordenables]] y [[Desarrollo Profesional/Sistemas Distribuidos/Páginas/03 - Transacciones Distribuidas|idempotencia]]

---
`#diseño-de-sistemas` `#chat` `#websockets` `#tiempo-real` `#caso-estudio`
