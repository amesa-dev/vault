# 🌐 Sistemas Distribuidos 02 — Mensajería y Brokers

[[Desarrollo Profesional/Sistemas Distribuidos/Sistemas Distribuidos|⬅️ Volver a Sistemas Distribuidos]] | [[Desarrollo Profesional/Sistemas Distribuidos/Páginas/01 - Fundamentos y CAP|← 01]] | [[Desarrollo Profesional/Sistemas Distribuidos/Páginas/03 - Transacciones Distribuidas|03 →]]

> [!abstract] Introducción
> La mensajería asíncrona es la forma de desacoplar servicios: en lugar de que A llame a B y espere, A publica un mensaje y sigue; B lo consume cuando puede. Esto absorbe picos de carga, tolera que B esté caído un rato y permite que muchos consumidores reaccionen al mismo evento. Pero introduce sus propios problemas: ¿se entrega el mensaje exactamente una vez? ¿qué pasa si el consumidor falla a medias? ¿cómo garantizas que publicas el evento solo si la transacción de base de datos tuvo éxito? Esta página cubre los modelos de broker, las garantías de entrega y el patrón Outbox.

## ¿De qué vamos a hablar?

Cómo funcionan los brokers de mensajería, qué garantías ofrecen y los patrones para no perder ni duplicar mensajes de forma destructiva.

### Conceptos que vamos a cubrir
- Cola (queue) vs log de eventos: los dos modelos fundamentales
- RabbitMQ, Kafka y Pub/Sub: cuándo cada uno
- Garantías de entrega: at-most-once, at-least-once, exactly-once
- Dead Letter Queues y reintentos
- El problema de la doble escritura y el patrón Outbox

---

## El Concepto

### Cola vs Log: los Dos Modelos

Hay dos arquitecturas mentales distintas, y confundirlas lleva a malas decisiones:

**Cola de mensajes (RabbitMQ, SQS, Pub/Sub clásico):** un mensaje se entrega a *un* consumidor de un grupo y luego **se borra**. El broker rastrea qué se ha consumido. Ideal para distribuir tareas (work queue): 10 workers compiten por mensajes, cada tarea la procesa uno.

```
Productor → [ msg3 | msg2 | msg1 ] → Consumidor (toma msg1, se borra)
                  cola                 ↘ otro consumidor toma msg2
```

**Log de eventos (Kafka, Pub/Sub con retención, Kinesis):** los mensajes se **anexan a un log inmutable y persistente**, ordenado dentro de cada partición. Los consumidores llevan su propio puntero (offset) y leen a su ritmo. El mensaje *no se borra* al consumirse — sigue ahí para otros consumidores o para reproceso.

```
Partición 0: [ev0 ev1 ev2 ev3 ev4 ev5 ...]   (inmutable, ordenado)
                       ▲            ▲
              Consumidor-A   Consumidor-B   (cada uno con su offset)
```

| | Cola (RabbitMQ) | Log (Kafka) |
|--|------------------|-------------|
| Tras consumir | Se borra | Permanece (retención por tiempo/tamaño) |
| Orden | Por cola | Por partición |
| Reproceso | No (ya no está) | Sí (rebobinas el offset) |
| Múltiples consumidores del mismo mensaje | Fan-out con exchanges | Nativo (cada grupo su offset) |
| Caso ideal | Reparto de tareas, RPC asíncrono | Event sourcing, streaming, pipelines de datos |

### Garantías de Entrega

Tres niveles, de menos a más fuerte:

- **At-most-once** ("como mucho una vez"): se entrega 0 o 1 veces. Si el consumidor falla tras recibir pero antes de procesar, el mensaje se pierde. Rápido, sin reintentos. Útil para métricas o telemetría donde perder un dato es tolerable.
- **At-least-once** ("al menos una vez"): el broker reintenta hasta recibir un ACK. Si el consumidor procesa pero muere antes de hacer ACK, el mensaje se reentrega → **duplicados**. Es el modelo por defecto de la mayoría de brokers. **Requiere consumidores idempotentes** (ver [página 03](03%20-%20Transacciones%20Distribuidas)).
- **Exactly-once** ("exactamente una vez"): cada mensaje afecta el estado una sola vez. Es el más caro y a menudo malentendido — Kafka lo ofrece dentro de su propio ecosistema (transacciones Kafka), pero **a través de un sistema externo (tu base de datos), "exactly-once" real se logra con at-least-once + idempotencia**, no por arte del broker.

> [!warning] La verdad incómoda
> "Exactly-once delivery" end-to-end con efectos secundarios externos es, en el caso general, imposible (problema de los dos generales). Lo que sí consigues es **exactly-once processing**: el mensaje puede entregarse varias veces, pero tu lógica garantiza que el efecto se aplica una sola vez. Diseña para at-least-once + idempotencia, siempre.

### ACKs, Reintentos y Dead Letter Queue

El consumidor confirma (ACK) tras procesar con éxito. Si falla o no responde a tiempo, el broker reentrega. Un mensaje "venenoso" (que siempre falla) provocaría reintentos infinitos, así que tras N intentos se manda a una **Dead Letter Queue (DLQ)** para inspección manual.

```python
# Patrón de consumidor robusto (pseudocódigo sobre cualquier broker)
MAX_REINTENTOS = 5

def consumir(mensaje) -> None:
    try:
        procesar(mensaje)          # tu lógica idempotente
        mensaje.ack()              # confirma: el broker lo da por hecho
    except ErrorTransitorio:
        if mensaje.intentos < MAX_REINTENTOS:
            mensaje.nack(requeue=True)   # reintenta con backoff
        else:
            enviar_a_dlq(mensaje)        # se rinde, lo aparta sin bloquear la cola
            mensaje.ack()
    except ErrorPermanente:
        # un mensaje malformado nunca se arreglará reintentando
        enviar_a_dlq(mensaje)
        mensaje.ack()
```

El reintento debe usar **backoff exponencial con jitter** (ver [[Desarrollo Profesional/Patrones de Resiliencia/Páginas/01 - Timeouts Reintentos y Backoff|Resiliencia 01]]) para no martillear un servicio ya caído.

### El Problema de la Doble Escritura

Este es *el* problema clásico de la mensajería, y casi todo el mundo lo implementa mal al principio:

```python
# ❌ INCORRECTO — dos sistemas, sin transacción común
def confirmar_pedido(pedido_id):
    db.update("UPDATE pedidos SET estado='confirmado' WHERE id=%s", pedido_id)
    broker.publicar(PedidoConfirmado(pedido_id))   # ← ¿y si esto falla?
```

Hay dos fallos posibles y ambos corrompen el sistema:
1. La BD commitea pero el broker se cae antes de publicar → el pedido está confirmado pero **nadie se entera** (no se envía email, no se reserva stock).
2. El broker publica pero la BD hace rollback → se reacciona a un evento que **nunca ocurrió de verdad**.

No puedes envolver "BD + broker" en una sola transacción ACID (son sistemas distintos; eso requeriría 2PC, que evitamos por frágil y lento).

### El Patrón Outbox

La solución estándar: **escribe el evento en la misma base de datos y en la misma transacción que el cambio de negocio**, en una tabla `outbox`. Un proceso aparte lee esa tabla y publica al broker.

```python
# ✅ CORRECTO — un solo commit atómico de negocio + evento
def confirmar_pedido(pedido_id, conn):
    with conn.transaction():   # UNA transacción ACID
        conn.execute("UPDATE pedidos SET estado='confirmado' WHERE id=%s", (pedido_id,))
        conn.execute(
            "INSERT INTO outbox (id, tipo, payload, creado_en, publicado) "
            "VALUES (%s, %s, %s, now(), false)",
            (uuid4(), "PedidoConfirmado", json.dumps({"pedido_id": pedido_id}), )
        )
    # Si la transacción commitea, el evento está garantizado en outbox.
    # Si hace rollback, no hay ni cambio ni evento. Atómico.

# Proceso publicador independiente (relay), corriendo en bucle:
def relay_outbox(conn, broker):
    filas = conn.execute(
        "SELECT id, tipo, payload FROM outbox "
        "WHERE publicado = false ORDER BY creado_en LIMIT 100"
    ).fetchall()
    for fila in filas:
        broker.publicar(fila["tipo"], fila["payload"])   # at-least-once
        conn.execute("UPDATE outbox SET publicado = true WHERE id = %s", (fila["id"],))
        # Si el relay muere entre publicar y marcar, republicará → duplicado.
        # Por eso el consumidor DEBE ser idempotente. At-least-once de nuevo.
```

Dos formas de leer la outbox:
- **Polling Publisher**: un proceso consulta la tabla cada X ms (simple, fácil de operar; latencia y carga de polling).
- **Change Data Capture (CDC)**: una herramienta como Debezium lee el *write-ahead log* de PostgreSQL y publica los inserts de `outbox` a Kafka automáticamente (más eficiente, sin polling, más piezas que operar).

El patrón inverso —**Inbox**— hace lo mismo en el consumidor: registra los IDs de mensajes ya procesados en una tabla para descartar duplicados. Outbox + Inbox + idempotencia es la combinación que hace la mensajería fiable de verdad.

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - Dos modelos: **cola** (el mensaje se borra al consumirse, ideal para repartir tareas) y **log** (inmutable y persistente, ideal para event sourcing y reproceso). Kafka es un log; RabbitMQ/SQS son colas.
> - Garantías: **at-most-once** (puede perder), **at-least-once** (puede duplicar — el defecto sensato), **exactly-once** (caro y limitado). Diseña siempre para at-least-once + idempotencia.
> - Mensajes que siempre fallan van a una **Dead Letter Queue** tras N reintentos con backoff, para no bloquear la cola.
> - El **problema de la doble escritura** (commitear en BD y publicar en el broker no es atómico) corrompe el estado. El **patrón Outbox** lo resuelve: escribe el evento en la misma transacción de BD y publícalo después con un relay (polling o CDC).
> - El consumidor debe ser idempotente porque el relay también es at-least-once. Outbox + Inbox + idempotencia = mensajería fiable.

### Para llevar a la práctica
- [ ] Dibuja el flujo de un evento en tu sistema: ¿hay alguna doble escritura sin Outbox? Es un bug latente.
- [ ] Revisa si tus consumidores son idempotentes. Si no, un reintento del broker corrompe datos.
- [ ] Comprueba que tus colas tienen DLQ configurada. Sin ella, un mensaje venenoso puede bloquear todo.

### Recursos
- 📖 *Designing Data-Intensive Applications* — Martin Kleppmann (capítulo 11: Stream Processing)
- 🌐 microservices.io/patterns/data/transactional-outbox.html — Chris Richardson
- 🌐 kafka.apache.org/documentation — sección de garantías de entrega y transacciones
- 🌐 debezium.io — Change Data Capture para Outbox

---
`#sistemas-distribuidos` `#mensajeria` `#kafka` `#outbox`
