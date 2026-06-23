# 🏛️ Caso de Estudio — Sistema de Pagos

[[Desarrollo Profesional/Diseño de Sistemas/Diseño de Sistemas|⬅️ Volver a Diseño de Sistemas]] | [[Desarrollo Profesional/Diseño de Sistemas/Páginas/15 - Caso Proximidad Geoespacial|← 15]]

> [!abstract] Introducción
> Un sistema de pagos es el caso donde **la corrección importa más que la escala**. En un feed, perder un post es molesto; en pagos, cobrar dos veces o perder dinero es inaceptable. Reúne lo más exigente de todo lo anterior: idempotencia obligatoria, transacciones distribuidas con servicios externos poco fiables (los bancos), reconciliación, consistencia y una traza auditable perfecta. Es el broche ideal porque obliga a aplicar con rigor lo de [[Desarrollo Profesional/Sistemas Distribuidos/Páginas/03 - Transacciones Distribuidas|transacciones distribuidas]] y [[Desarrollo Profesional/Patrones de Resiliencia/Patrones de Resiliencia|resiliencia]].

## ¿De qué vamos a hablar?

El diseño de un sistema de pagos centrado en la corrección: idempotencia, el flujo de pago, la consistencia con sistemas externos y la reconciliación.

### Conceptos que vamos a cubrir
- Requisitos: corrección por encima de todo
- El flujo de un pago y los actores
- Idempotencia: no cobrar dos veces, jamás
- Consistencia: el ledger de doble entrada
- Reconciliación, reintentos y el problema del estado desconocido

---

## El Diseño

### 1. Requisitos

**Funcionales**: procesar un pago (cobrar a un comprador, abonar a un vendedor), gestionar reembolsos, mantener un registro auditable.

**No funcionales**: **corrección absoluta** (exactly-once en el efecto: nunca cobrar dos veces ni perder un cobro), **durabilidad** (un pago confirmado no se pierde nunca), consistencia, auditabilidad total, y seguridad/cumplimiento (PCI-DSS). La escala importa, pero **la corrección manda**.

### 2. El Flujo de un Pago y los Actores

```
[Usuario] → [Servicio de Pagos] → [Procesador de pagos / PSP externo (Stripe, banco)]
                  │
            [Ledger / libro mayor]  ← registro contable inmutable
                  │
            [Reconciliación]        ← cuadra lo nuestro con lo del PSP
```

- El **servicio de pagos** orquesta; **no** habla directamente con redes bancarias, usa un **PSP** (Payment Service Provider) externo (Stripe, Adyen) que es la parte poco fiable y lenta.
- El **ledger** (libro mayor) es el registro de verdad de cada movimiento de dinero.
- La **reconciliación** verifica periódicamente que nuestro registro y el del PSP coinciden.

Es esencialmente una **Saga** ([[Desarrollo Profesional/Sistemas Distribuidos/Páginas/03 - Transacciones Distribuidas|transacciones distribuidas]]): reservar, cobrar, abonar, confirmar — con compensaciones si algo falla.

### 3. Idempotencia: la Regla de Oro

El requisito que lo domina todo: **nunca cobrar dos veces.** Y los reintentos son inevitables (la red falla, la respuesta del PSP se pierde justo después de cobrar). La solución es la **clave de idempotencia** en cada operación de pago:

```python
def procesar_pago(idempotency_key, importe, conn):
    with conn.transaction():
        # ¿Ya procesamos este pago? La clave la genera el cliente y la reenvía en cada retry.
        existente = conn.execute(
            "SELECT estado, resultado FROM pagos WHERE idempotency_key = %s",
            (idempotency_key,)).fetchone()
        if existente:
            return existente["resultado"]   # devuelve el MISMO resultado, no recobra

        # Registrar la intención ANTES de llamar al PSP (estado: PENDIENTE)
        conn.execute(
            "INSERT INTO pagos (idempotency_key, importe, estado) VALUES (%s,%s,'PENDIENTE')",
            (idempotency_key, importe))
    # ... llamar al PSP fuera o dentro con cuidado (ver estado desconocido) ...
```

El cliente genera la clave (un UUID) y la **reenvía idéntica en cada reintento**; el servidor deduplica. Es el patrón de Stripe (`Idempotency-Key`). Sin esto, un timeout + reintento = doble cobro = desastre.

### 4. Consistencia: el Ledger de Doble Entrada

El dinero se modela con **contabilidad de doble entrada** (double-entry ledger), el patrón milenario de la contabilidad: **cada transacción registra dos asientos que se cancelan** (un débito en una cuenta, un crédito en otra), y la suma de todo el sistema siempre cuadra a cero.

```
Pago de 100€ del comprador al vendedor:
  - DÉBITO  cuenta_comprador  -100
  - CRÉDITO cuenta_vendedor   +100
  (la suma es 0; el dinero ni se crea ni se destruye, solo se mueve)
```

Propiedades clave del ledger:
- **Inmutable y append-only**: no se edita un asiento; un error se corrige con un asiento compensatorio nuevo (como [[Desarrollo Profesional/DDD/Páginas/03 - Patrones Avanzados|Event Sourcing]]). Esto da **auditabilidad perfecta**: toda la historia del dinero es reconstruible.
- **Consistencia fuerte**: las operaciones del ledger usan transacciones ACID ([[Desarrollo Profesional/PostgreSQL/Páginas/03 - Transacciones y MVCC|PostgreSQL]]). Aquí **no** vale la consistencia eventual del feed: el saldo debe ser correcto siempre.
- Sirve de fuente de verdad para la reconciliación.

### 5. El Problema del Estado Desconocido y la Reconciliación

El escenario más peligroso: llamas al PSP para cobrar, y **no recibes respuesta** (timeout). ¿Se cobró o no? **No lo sabes.** Reintentar a ciegas podría doblar el cobro; no reintentar podría perderlo. Soluciones:

- **Idempotencia en el PSP**: envías la misma idempotency-key al PSP; si ya procesó ese cobro, te devuelve el resultado original en vez de cobrar otra vez. Así reintentar es seguro.
- **Consultar el estado**: ante la duda, **preguntas** al PSP "¿qué pasó con el pago X?" antes de reintentar.
- **Reconciliación**: un proceso periódico que descarga el registro del PSP y lo **cuadra** con tu ledger, detectando discrepancias (cobros que tú crees pendientes pero el PSP completó, o viceversa) para corregirlas. Es la red de seguridad final que garantiza que, pase lo que pase en tiempo real, los libros acaban cuadrando.

### 6. Cuellos de Botella y Mejoras

- **PSP caído/lento**: resiliencia completa (timeouts, reintentos idempotentes, circuit breaker); los pagos pendientes se resuelven por reconciliación.
- **Reintentos seguros**: solo posibles gracias a idempotencia end-to-end (cliente → servicio → PSP).
- **Cumplimiento (PCI-DSS)**: no almacenar datos de tarjeta crudos; tokenizar vía el PSP. Cifrado y secretos según [[Desarrollo Profesional/Seguridad Aplicada/Páginas/03 - Criptografía y Secretos|Seguridad 03]].
- **Auditoría y antifraude**: el ledger inmutable + detección de patrones sospechosos.
- **La lección transferible**: en sistemas de dinero, **diseña asumiendo que cada paso puede fallar a medias**, y haz que la corrección sea recuperable (idempotencia + ledger + reconciliación), no dependiente de que el camino feliz no falle nunca.

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - En pagos, **la corrección manda sobre la escala**: nunca cobrar dos veces ni perder un cobro, durabilidad total y auditabilidad. Es una **Saga** con un PSP externo poco fiable.
> - **Idempotencia es la regla de oro**: cada pago lleva una `Idempotency-Key` que el cliente reenvía en cada reintento; el servidor (y el PSP) deduplican. Sin ella, timeout+retry = doble cobro.
> - El dinero se modela con un **ledger de doble entrada**: cada transacción son dos asientos que cuadran a cero, **inmutable y append-only** (errores se corrigen con asientos compensatorios). Da auditabilidad perfecta y exige **consistencia fuerte ACID** (no eventual).
> - El peligro máximo es el **estado desconocido** (cobraste al PSP y no recibiste respuesta): se resuelve con idempotencia en el PSP, consultando el estado antes de reintentar, y sobre todo con **reconciliación** periódica que cuadra tu ledger con el del PSP.
> - Diseña **asumiendo que cada paso falla a medias** y haz la corrección recuperable (idempotencia + ledger + reconciliación), no dependiente del camino feliz.

### Para llevar a la práctica
- [ ] Diseña el flujo de un cobro con idempotency-key y razona qué pasa exactamente en un reintento tras timeout.
- [ ] Modela un pago como dos asientos de doble entrada y verifica que la suma del sistema es cero.
- [ ] Explica el problema del "estado desconocido" y los tres mecanismos para resolverlo.
- [ ] Diseña el proceso de reconciliación: ¿qué comparas y qué haces ante una discrepancia?

### Recursos
- 📖 *System Design Interview, Vol. 2* — Alex Xu (Design a Payment System)
- 🌐 stripe.com/docs/idempotency y stripe.com/docs/payments/payment-intents
- 📄 "Building a ledger" — blogs de ingeniería de fintech (Modern Treasury, Square)
- 📄 Conexión con [[Desarrollo Profesional/Sistemas Distribuidos/Páginas/03 - Transacciones Distribuidas|Sagas e idempotencia]] y [[Desarrollo Profesional/DDD/Páginas/03 - Patrones Avanzados|Event Sourcing]]

---
`#diseño-de-sistemas` `#pagos` `#idempotencia` `#ledger` `#caso-estudio`
