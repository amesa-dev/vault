# 🌐 Sistemas Distribuidos 03 — Transacciones Distribuidas

[[Desarrollo Profesional/Sistemas Distribuidos/Sistemas Distribuidos|⬅️ Volver a Sistemas Distribuidos]] | [[Desarrollo Profesional/Sistemas Distribuidos/Páginas/02 - Mensajería y Brokers|← 02]]

> [!abstract] Introducción
> Cuando una operación de negocio abarca varios servicios —reservar stock, cobrar el pago, crear el envío— cada uno con su propia base de datos, no puedes envolverla en un `BEGIN...COMMIT` único. Necesitas coordinar pasos que pueden fallar a medias y dejar el sistema en un estado a medio cocer. Esta página explica por qué el two-phase commit no escala, cómo las Sagas resuelven el problema con compensaciones, y por qué la idempotencia es el cimiento sobre el que todo esto se sostiene.

## ¿De qué vamos a hablar?

Cómo mantener la consistencia de una operación que atraviesa varios servicios sin una transacción ACID global.

### Conceptos que vamos a cubrir
- Por qué el Two-Phase Commit (2PC) no escala
- El patrón Saga: coreografía vs orquestación
- Compensaciones: el "rollback" de negocio
- Idempotencia: la propiedad que lo hace todo posible
- Consistencia eventual y el modelo de estados

---

## El Concepto

### Por Qué el Two-Phase Commit No Sirve a Escala

El 2PC intenta dar una transacción ACID sobre varios sistemas con un coordinador:

1. **Fase de preparación**: el coordinador pregunta a todos "¿puedes commitear?". Cada uno bloquea recursos y responde sí/no.
2. **Fase de commit**: si todos dijeron sí, el coordinador ordena commitear; si alguno dijo no, ordena abortar.

El problema fatal: **si el coordinador se cae después de la fase de preparación**, todos los participantes quedan bloqueando recursos esperando una decisión que nunca llega (blocking protocol). Además mantiene locks durante toda la coordinación, lo que mata el throughput, y acopla la disponibilidad de todos los servicios (si uno está caído, nadie commitea). Por eso en microservicios **se evita el 2PC** y se adopta consistencia eventual con Sagas.

### El Patrón Saga

Una Saga descompone la transacción distribuida en una **secuencia de transacciones locales**, cada una en su propio servicio y su propia BD. Si un paso falla, se ejecutan **transacciones de compensación** que deshacen semánticamente los pasos anteriores.

La diferencia clave con ACID: una Saga **no es atómica ni aislada**. Hay momentos intermedios donde el sistema está parcialmente actualizado y otros pueden verlo. Ganas disponibilidad y desacoplamiento; pagas con consistencia eventual y la obligación de diseñar compensaciones.

```
Saga "Crear Pedido" (camino feliz):
  [Pedidos] crea pedido → [Inventario] reserva stock → [Pagos] cobra → [Envíos] programa

Si [Pagos] falla:
  [Pagos] aborta → COMPENSA [Inventario] libera stock → COMPENSA [Pedidos] cancela pedido
```

Una compensación **no es un rollback de BD** — es una acción de negocio nueva que neutraliza el efecto: si cobraste, reembolsas; si reservaste stock, lo liberas; si enviaste un email de confirmación... no puedes "des-enviarlo", mandas uno de corrección. Por eso algunas acciones deben colocarse al final (las irreversibles) o detrás de una confirmación.

### Coreografía vs Orquestación

Dos formas de coordinar la Saga:

**Coreografía** — cada servicio reacciona a eventos y publica los suyos. No hay coordinador central; la lógica está distribuida.

```
Pedido creado ──→ Inventario escucha, reserva, publica "Stock reservado"
                  ──→ Pagos escucha, cobra, publica "Pago realizado"
                      ──→ Envíos escucha, programa...
```
*Pro*: desacoplado, sin punto único. *Contra*: el flujo no está en ningún sitio; entender "qué pasa cuando se crea un pedido" exige leer N servicios. Difícil de depurar a partir de cierto tamaño.

**Orquestación** — un coordinador (el orquestador de la Saga) dice a cada servicio qué hacer y reacciona a sus respuestas.

```python
# Orquestador de Saga — la lógica del flujo vive en un solo sitio
class SagaCrearPedido:
    def __init__(self, pedidos, inventario, pagos, envios) -> None:
        self._pedidos = pedidos
        self._inventario = inventario
        self._pagos = pagos
        self._envios = envios

    def ejecutar(self, cmd) -> None:
        compensaciones = []   # pila de "deshacer" en orden inverso
        try:
            pedido_id = self._pedidos.crear(cmd)
            compensaciones.append(lambda: self._pedidos.cancelar(pedido_id))

            reserva = self._inventario.reservar(pedido_id, cmd.items)
            compensaciones.append(lambda: self._inventario.liberar(reserva))

            cobro = self._pagos.cobrar(pedido_id, cmd.importe)
            compensaciones.append(lambda: self._pagos.reembolsar(cobro))

            self._envios.programar(pedido_id)   # paso final, idealmente irreversible
            self._pedidos.marcar_completado(pedido_id)

        except Exception as e:
            # Compensar en orden inverso: deshacer lo último primero
            for compensar in reversed(compensaciones):
                try:
                    compensar()
                except Exception:
                    # Una compensación fallida va a alerta/intervención manual.
                    # Las compensaciones también deben ser idempotentes y reintentar.
                    registrar_fallo_compensacion(pedido_id)
            raise
```
*Pro*: el flujo es explícito y testeable; fácil de razonar. *Contra*: el orquestador es un componente más que mantener (aunque no es un punto único de fallo si lo despliegas con réplicas y persistes su estado). **Para flujos no triviales, prefiere orquestación.** Herramientas como Temporal, Camunda o AWS Step Functions son orquestadores de Sagas como servicio.

### Idempotencia: el Cimiento

Una operación es **idempotente** si ejecutarla N veces tiene el mismo efecto que ejecutarla una vez. Es la propiedad que hace seguro el at-least-once: como los mensajes pueden duplicarse y los pasos pueden reintentarse, cada paso debe poder repetirse sin daño.

```python
# ❌ NO idempotente — reintentar cobra dos veces
def cobrar(cuenta, importe):
    cuenta.saldo -= importe   # ejecutar dos veces resta el doble

# ✅ Idempotente — con clave de idempotencia
def cobrar(cuenta, importe, idempotency_key, conn):
    with conn.transaction():
        # ¿Ya procesamos esta operación? (tabla Inbox)
        ya_hecho = conn.execute(
            "SELECT 1 FROM cobros_procesados WHERE idempotency_key = %s",
            (idempotency_key,)
        ).fetchone()
        if ya_hecho:
            return   # no-op: el efecto ya se aplicó, reintento seguro

        conn.execute("UPDATE cuentas SET saldo = saldo - %s WHERE id = %s",
                     (importe, cuenta.id))
        conn.execute("INSERT INTO cobros_procesados (idempotency_key) VALUES (%s)",
                     (idempotency_key,))
        # El INSERT y el UPDATE en la misma transacción: o ambos o ninguno.
        # Si llega un duplicado, el SELECT lo detecta y no hace nada.
```

La **clave de idempotencia** es un identificador único de la *intención* (no de la petición HTTP): el cliente la genera (un UUID) y la reenvía en cada reintento. El servidor la usa para deduplicar. Stripe, por ejemplo, expone una cabecera `Idempotency-Key` precisamente para esto.

Formas comunes de lograr idempotencia:
- **Claves de idempotencia + tabla de procesados** (lo de arriba, lo más general).
- **Operaciones naturalmente idempotentes**: `SET estado = 'confirmado'` (no `toggle`), `PUT` en vez de `POST`, `INSERT ... ON CONFLICT DO NOTHING`.
- **Comprobación de estado**: "si el pedido ya está confirmado, no hagas nada".

### El Modelo de Estados

Como una Saga atraviesa estados intermedios visibles, modela explícitamente la máquina de estados de tu entidad y haz que las transiciones sean válidas e idempotentes:

```
borrador → pendiente_pago → pagado → en_envío → completado
                 │
                 └─→ cancelado (vía compensación)
```

Cada transición comprueba el estado actual antes de actuar (`if estado != 'pendiente_pago': return`), lo que la hace idempotente y robusta frente a eventos fuera de orden — algo inevitable en sistemas distribuidos (recuerda [[Desarrollo Profesional/Sistemas Distribuidos/Páginas/01 - Fundamentos y CAP|que no hay orden global]]).

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - El **Two-Phase Commit** da atomicidad entre sistemas pero bloquea recursos y se cuelga si el coordinador cae. No escala en microservicios.
> - Una **Saga** descompone la transacción distribuida en transacciones locales encadenadas; si un paso falla, ejecuta **compensaciones** (acciones de negocio que neutralizan los pasos previos, no rollbacks de BD).
> - **Coreografía** (servicios reaccionan a eventos, desacoplado pero difícil de seguir) vs **orquestación** (un coordinador dirige el flujo, explícito y testeable). Para flujos complejos, orquestación.
> - La **idempotencia** es el cimiento: como todo es at-least-once, cada paso debe poder repetirse sin daño. Se logra con claves de idempotencia + tabla de procesados, operaciones naturalmente idempotentes o comprobación de estado.
> - Modela la **máquina de estados** explícitamente; cada transición valida el estado actual, lo que la hace idempotente y tolerante a eventos fuera de orden.

### Para llevar a la práctica
- [ ] Toma una operación de tu sistema que toque ≥2 servicios y dibuja la Saga: pasos, compensaciones y orden.
- [ ] Identifica los pasos irreversibles (enviar email, llamar a una API externa de cobro) y colócalos donde la compensación sea posible o al final.
- [ ] Añade una `Idempotency-Key` a una operación crítica y una tabla de deduplicación.

### Recursos
- 📖 *Microservices Patterns* — Chris Richardson (capítulo 4: Saga)
- 📖 *Designing Data-Intensive Applications* — Martin Kleppmann (capítulo 9: 2PC y consenso)
- 🌐 microservices.io/patterns/data/saga.html
- 🌐 temporal.io / docs — orquestación de workflows duraderos como servicio
- 🌐 stripe.com/docs/api/idempotent_requests — idempotencia en una API real

---
`#sistemas-distribuidos` `#saga` `#idempotencia` `#transacciones`
