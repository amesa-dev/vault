# 🌐 Sistemas Distribuidos 01 — Fundamentos y CAP

[[Desarrollo Profesional/Sistemas Distribuidos/Sistemas Distribuidos|⬅️ Volver a Sistemas Distribuidos]] | [[Desarrollo Profesional/Sistemas Distribuidos/Páginas/02 - Mensajería y Brokers|02 →]]

> [!abstract] Introducción
> Antes de hablar de Kafka o Sagas hay que interiorizar por qué los sistemas distribuidos son difíciles. No es complejidad accidental: es que ciertas garantías son físicamente imposibles de tener todas a la vez. Esta página establece los fundamentos —las falacias de la computación distribuida, el teorema CAP y su evolución PACELC, los niveles de consistencia y el problema del orden y el tiempo— para que el resto de la sección tenga dónde anclarse.

## ¿De qué vamos a hablar?

Las leyes que ningún sistema distribuido puede violar, y el vocabulario para razonar sobre los compromisos que sí puedes elegir.

### Conceptos que vamos a cubrir
- Las 8 falacias de la computación distribuida
- El teorema CAP y por qué "elige 2 de 3" es una simplificación
- PACELC: el compromiso que CAP omite (latencia vs consistencia)
- Niveles de consistencia: fuerte, eventual, causal, read-your-writes
- Relojes, orden y por qué no existe un "ahora" global

---

## El Concepto

### Las 8 Falacias de la Computación Distribuida

Peter Deutsch (Sun Microsystems) enumeró las suposiciones falsas que todo programador novato hace al construir sistemas distribuidos. Cada una ha causado outages reales:

1. **La red es fiable** — los paquetes se pierden, las conexiones se cortan.
2. **La latencia es cero** — una llamada de red es ~6 órdenes de magnitud más lenta que una llamada en memoria.
3. **El ancho de banda es infinito** — mover datos cuesta; un payload grande satura el enlace.
4. **La red es segura** — hay adversarios; cifra y autentica (ver [[Desarrollo Profesional/Seguridad Aplicada/Seguridad Aplicada|Seguridad]]).
5. **La topología no cambia** — los nodos entran y salen, las IPs cambian.
6. **Hay un único administrador** — atraviesas equipos, clouds y políticas distintas.
7. **El coste de transporte es cero** — serializar, mover y deserializar consume CPU y dinero.
8. **La red es homogénea** — protocolos, versiones y MTUs distintos coexisten.

La consecuencia práctica: **trata cada llamada remota como algo que puede fallar, tardar mucho o ejecutarse dos veces.** Todo lo demás en sistemas distribuidos deriva de aceptar esto.

### El Teorema CAP

Eric Brewer formuló que un sistema de datos distribuido solo puede garantizar **dos** de estas tres propiedades simultáneamente:

- **C — Consistency** (consistencia lineal): toda lectura ve la escritura más reciente. Todos los nodos coinciden.
- **A — Availability**: toda petición a un nodo no caído recibe respuesta (no error, no timeout).
- **P — Partition tolerance**: el sistema sigue funcionando aunque la red se parta y los nodos no puedan comunicarse.

El matiz que casi todo el mundo olvida: **en un sistema distribuido real las particiones de red ocurren, no son opcionales.** Por tanto P no es negociable. La elección real no es "2 de 3", es: **cuando hay una partición, ¿sacrificas C o sacrificas A?**

```
                 ¿Hay partición de red?
                          │
              ┌───────────┴───────────┐
             SÍ                       NO
              │                        │
   ¿Qué sacrifico?          (puedo tener C y A)
      │            │
   CP: rechazo   AP: respondo
   o bloqueo     con datos
   (consistente  posiblemente
   pero no       obsoletos
   disponible)   (disponible pero
                 inconsistente)
```

- **Sistema CP** (consistente bajo partición): un nodo que no puede confirmar que tiene el dato más reciente prefiere fallar. Ejemplos: bases de datos con consenso fuerte (etcd, ZooKeeper, Spanner). Lo usas cuando una respuesta incorrecta es peor que ninguna respuesta (saldos bancarios, locks distribuidos).
- **Sistema AP** (disponible bajo partición): cada nodo responde con lo que sabe, aunque esté desactualizado, y reconcilia después. Ejemplos: Cassandra, DynamoDB en modo eventual, DNS. Lo usas cuando estar disponible importa más que estar perfectamente actualizado (carritos, feeds, contadores de "me gusta").

### PACELC — Lo que CAP no dice

CAP solo habla del caso de partición, que es raro. PACELC (Daniel Abadi) completa el cuadro:

> **Si** hay **P**artición, eliges entre **A**vailability y **C**onsistency; **E**lse (en operación normal), eliges entre **L**atencia y **C**onsistencia.

Esto es lo que de verdad sientes a diario: incluso sin particiones, garantizar consistencia fuerte exige coordinar nodos (esperar confirmaciones), y eso **cuesta latencia**. Un sistema "EL" (baja latencia a costa de consistencia) responde rápido leyendo de una réplica que puede ir atrasada; un sistema "EC" espera a la confirmación del líder.

- DynamoDB, Cassandra: **PA/EL** — disponibles y rápidos, consistencia eventual.
- Spanner, etcd: **PC/EC** — consistentes siempre, pagas latencia.

### Niveles de Consistencia

"Consistente" no es binario. De más fuerte a más débil:

| Nivel | Garantía | Coste |
|-------|----------|-------|
| **Lineal / fuerte** | Toda lectura ve la última escritura confirmada, como si hubiera un solo nodo | Alto (consenso, latencia) |
| **Secuencial** | Todos ven las operaciones en el mismo orden, no necesariamente en tiempo real | Alto |
| **Causal** | Si A causó B, todos ven A antes que B; operaciones no relacionadas pueden verse en distinto orden | Medio |
| **Read-your-writes** | Ves tus propias escrituras; las de otros pueden tardar | Bajo |
| **Eventual** | Si nadie escribe, eventualmente todos convergen al mismo valor | Mínimo |

La mayoría de los sistemas a escala usan **consistencia eventual** con parches de consistencia más fuerte donde el negocio lo exige. Un patrón típico: escrituras consistentes en el dato canónico, lecturas eventuales sobre réplicas/cachés (es la base de [[Desarrollo Profesional/DDD/Páginas/03 - Patrones Avanzados|CQRS]]).

### Relojes, Orden y el "Ahora" Inexistente

En una sola máquina, el reloj te da un orden total: lo que pasó antes tiene timestamp menor. En distribuido **no puedes confiar en los timestamps de pared** para ordenar eventos entre máquinas: los relojes derivan (clock skew), NTP los ajusta hacia atrás, y dos eventos "simultáneos" en máquinas distintas no tienen un orden definido.

La herramienta conceptual es la **relación happened-before** de Lamport y los **relojes lógicos**:

```python
# Reloj de Lamport — ordena eventos sin depender del tiempo de pared.
# Regla: cada evento incrementa el contador; al recibir un mensaje,
# tomas el máximo entre tu contador y el del mensaje, y sumas 1.
class RelojLamport:
    def __init__(self) -> None:
        self.contador = 0

    def evento_local(self) -> int:
        self.contador += 1
        return self.contador

    def enviar(self) -> int:
        self.contador += 1
        return self.contador  # timestamp que viaja con el mensaje

    def recibir(self, ts_mensaje: int) -> int:
        self.contador = max(self.contador, ts_mensaje) + 1
        return self.contador

# Si evento A tiene timestamp Lamport menor que B, B NO causó A.
# (No al revés: timestamps iguales o menores no implican causalidad).
```

Los **vector clocks** van más allá y detectan si dos eventos son concurrentes (ninguno causó al otro) — la base para resolver conflictos en sistemas AP como DynamoDB. La idea clave que debes llevarte: **en distribuido, el orden es algo que construyes explícitamente, no algo que la realidad te regala.**

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - Las falacias de la red (es fiable, la latencia es cero, es segura…) son la raíz de casi todo bug distribuido. Asume que toda llamada remota puede fallar, tardar o ejecutarse dos veces.
> - **CAP**: como las particiones son inevitables, la elección real es C vs A *durante una partición*. CP rechaza para no mentir; AP responde aunque el dato esté obsoleto.
> - **PACELC** completa CAP: incluso sin partición, eliges entre **latencia y consistencia**. Consistencia fuerte = coordinación = latencia.
> - La consistencia es un espectro (lineal → causal → read-your-writes → eventual). A escala se usa eventual con islas de consistencia fuerte donde el negocio lo exige.
> - No existe un "ahora" global: el orden entre máquinas se construye con relojes lógicos (Lamport, vector clocks), no con timestamps de pared.

### Para llevar a la práctica
- [ ] Para un sistema que conozcas, clasifícalo: ¿es CP o AP? ¿Qué pasa cuando se parte la red?
- [ ] Identifica una operación de tu producto que necesite consistencia fuerte y otra que tolere eventual. Justifica cada una.
- [ ] Busca en tu código un sitio donde ordenes eventos por `timestamp` de varias máquinas — es un bug latente.

### Recursos
- 📖 *Designing Data-Intensive Applications* — Martin Kleppmann (capítulos 5, 8 y 9: el libro de referencia)
- 📄 Lamport, L. (1978). *Time, Clocks, and the Ordering of Events in a Distributed System*
- 📄 Brewer, E. (2012). *CAP Twelve Years Later: How the "Rules" Have Changed*
- 🌐 Abadi, D. — *Consistency Tradeoffs in Modern Distributed Database System Design* (PACELC)

---
`#sistemas-distribuidos` `#cap` `#consistencia` `#relojes-logicos`
