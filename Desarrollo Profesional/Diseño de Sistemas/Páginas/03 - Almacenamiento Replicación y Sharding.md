# 🏛️ Diseño de Sistemas 03 — Almacenamiento, Replicación y Sharding

[[Desarrollo Profesional/Diseño de Sistemas/Diseño de Sistemas|⬅️ Volver a Diseño de Sistemas]] | [[Desarrollo Profesional/Diseño de Sistemas/Páginas/02 - Escalar de Cero a Millones|← 02]] | [[Desarrollo Profesional/Diseño de Sistemas/Páginas/04 - Caché CDN Balanceo y Colas|04 →]]

> [!abstract] Introducción
> La capa de datos es casi siempre el componente más difícil de escalar, porque es donde vive el estado. Esta página profundiza en las decisiones de almacenamiento que aparecen en todos los casos de estudio: elegir entre SQL y NoSQL (y por qué la respuesta casi nunca es obvia), cómo la replicación da disponibilidad y escala de lectura, cómo el sharding reparte las escrituras, y el algoritmo que hace el sharding manejable a escala dinámica: el consistent hashing.

## ¿De qué vamos a hablar?

Cómo elegir, replicar y particionar el almacenamiento para que aguante la escala sin perder la consistencia que el negocio necesita.

### Conceptos que vamos a cubrir
- SQL vs NoSQL: el criterio real
- Replicación: maestro-réplica y sus modos
- Sharding: estrategias y la elección de la shard key
- Los problemas del sharding: hotspots, joins, rebalanceo
- Consistent hashing: repartir sin reorganizarlo todo

---

## El Concepto

### SQL vs NoSQL

La pregunta mal planteada es "¿cuál es mejor?". La correcta es "¿qué necesita mi acceso a datos?".

**SQL (relacional: PostgreSQL, MySQL)**:
- Esquema estructurado, relaciones, **JOINs**, transacciones **ACID** fuertes ([[Desarrollo Profesional/PostgreSQL/Páginas/03 - Transacciones y MVCC|MVCC]]).
- Garantías de consistencia fuerte. Ideal cuando los datos son **relacionales** y la **integridad importa** (finanzas, pedidos, cualquier cosa con invariantes transaccionales).
- Escala de lectura con réplicas; escalar escrituras (sharding) es más doloroso porque rompe los JOINs y las transacciones cross-shard.

**NoSQL** (varias familias, no es una cosa):
- **Clave-valor** (Redis, DynamoDB): acceso por clave, rapidísimo, escala horizontal trivial.
- **Documental** (MongoDB): documentos JSON flexibles, sin esquema rígido.
- **Columnar ancha** (Cassandra, Bigtable): escrituras masivas, escala enorme, consistencia ajustable.
- **Grafo** (Neo4j): relaciones como ciudadano de primera clase.
- En general: esquema flexible, **escala horizontal nativa**, alto rendimiento — a cambio de consistencia más débil (a menudo eventual) y sin JOINs ni transacciones multi-documento robustas.

Cuándo NoSQL: datos sin relaciones complejas, escala de escritura masiva, esquema cambiante, latencia baja a escala global, o tolerancia a consistencia eventual. **Heurística**: empieza con SQL (PostgreSQL es un caballo de batalla que llega lejísimos); muévete a NoSQL para necesidades específicas que SQL no cubre bien, no por moda. Es habitual usar **ambos** (polyglot persistence): PostgreSQL para lo transaccional, Redis para caché, Cassandra para series temporales de eventos.

### Replicación

Mantener copias de los datos en varios nodos, para disponibilidad y escala de lectura. Lo vimos en [[Desarrollo Profesional/Diseño de Sistemas/Páginas/02 - Escalar de Cero a Millones|la página 02]]; aquí los matices:

- **Maestro-réplica (single-leader)**: un maestro acepta escrituras, las réplicas las copian y sirven lecturas. Simple, sin conflictos de escritura. El maestro es cuello de botella de escritura y punto único (mitigado con failover automático). Es lo más común.
- **Multi-maestro (multi-leader)**: varios nodos aceptan escrituras (útil entre datacenters). Mejora disponibilidad de escritura pero introduce **conflictos** (dos escrituras simultáneas del mismo dato en maestros distintos) que hay que resolver.
- **Sin líder (leaderless: Dynamo, Cassandra)**: cualquier nodo acepta lecturas/escrituras; se usa **quórum** (escribir en W nodos, leer de R, con R+W > N para solapamiento) para la consistencia. Muy disponible, consistencia ajustable.

El modo de replicación también es síncrono (consistente pero lento: esperas confirmación de réplicas) o asíncrono (rápido pero puedes perder datos si el maestro cae antes de propagar) — el compromiso de PACELC otra vez.

### Sharding (Particionado Horizontal)

Cuando los datos o las escrituras no caben en un nodo, los repartes entre **shards** (particiones), cada uno una BD independiente con un subconjunto. La decisión central es la **shard key** (o partition key): el campo que determina en qué shard va cada dato.

Estrategias de reparto:
- **Por rango**: shard por rangos de la clave (A-F, G-M…). Permite consultas por rango eficientes, pero **propenso a hotspots** (si muchos datos caen en un rango, ese shard se sobrecarga).
- **Por hash**: `shard = hash(clave) % N`. Reparte uniformemente, evita hotspots, pero pierde la localidad (las consultas por rango deben tocar todos los shards). Es la opción más común.
- **Por directorio**: una tabla de lookup mapea clave → shard. Flexible (puedes rebalancear moviendo entradas), pero la tabla es un punto a gestionar.

Elegir bien la shard key es **la decisión más importante**: debe repartir uniformemente (evitar hotspots) y alinearse con los patrones de acceso (que las consultas frecuentes toquen un solo shard). Una mala shard key te condena a queries cross-shard lentas o a un shard caliente.

### Los Problemas del Sharding

El sharding escala, pero trae dolor que hay que conocer antes de elegirlo:

- **Hotspots / claves calientes**: un shard recibe desproporcionadamente más tráfico (la celebridad con millones de seguidores, el producto viral). Mitigación: sub-particionar la clave caliente, o cachear agresivamente.
- **JOINs entre shards**: ya no puedes hacer un JOIN SQL si las tablas están en shards distintos. Se resuelve desnormalizando (duplicar datos), haciendo el join en la aplicación, o eligiendo la shard key para co-localizar lo que se consulta junto.
- **Transacciones cross-shard**: una transacción que abarca varios shards pierde el ACID simple; necesitas Sagas o consistencia eventual ([[Desarrollo Profesional/Sistemas Distribuidos/Páginas/03 - Transacciones Distribuidas|Transacciones Distribuidas]]).
- **Rebalanceo**: añadir un shard implica mover datos. Con `hash % N`, **cambiar N reubica casi todas las claves** — un problema enorme. Eso es justo lo que resuelve el consistent hashing.

### Consistent Hashing

El problema: con `shard = hash(clave) % N`, si pasas de 4 a 5 servidores, **casi todas las claves cambian de servidor** (porque cambió el módulo), provocando una migración masiva de datos y una tormenta de cache misses. El **consistent hashing** lo resuelve.

La idea: mapeas tanto las **claves** como los **servidores** sobre un mismo **anillo** hash (de 0 a 2³²-1). Cada clave pertenece al primer servidor que encuentras girando en el sentido de las agujas del reloj desde su posición.

```
        Servidor A
       /          \
   clave1          Servidor B
     |                |
   Servidor D      clave2
       \           /
        Servidor C
   (cada clave va al siguiente servidor en el anillo, en sentido horario)
```

La propiedad mágica: **al añadir o quitar un servidor, solo se remapean las claves de su segmento del anillo (≈ K/N claves), no todas.** Añadir el servidor E entre A y B solo mueve las claves que caían entre ellos; el resto no se inmuta.

Un refinamiento esencial son los **virtual nodes** (nodos virtuales): cada servidor físico se coloca en muchos puntos del anillo, no en uno. Esto reparte la carga de forma más uniforme (sin vnodes, un servidor podría quedar con un arco enorme del anillo) y suaviza el rebalanceo. Es el algoritmo que usan DynamoDB, Cassandra y los sistemas de caché distribuida para repartir datos minimizando el movimiento al escalar — por eso aparece en tantos diseños.

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - **SQL vs NoSQL**: no es "cuál es mejor", es qué necesita tu acceso a datos. SQL para datos relacionales con integridad transaccional; NoSQL (clave-valor, documental, columnar, grafo) para escala horizontal nativa, esquema flexible o escritura masiva, a costa de consistencia más débil. Empieza con SQL; combina ambos (polyglot persistence).
> - **Replicación**: maestro-réplica (simple, común, escala lecturas), multi-maestro (escritura distribuida con conflictos) y sin líder (quórum R+W>N). Síncrona (consistente, lenta) vs asíncrona (rápida, puede perder datos).
> - **Sharding** reparte escrituras/almacenamiento. La **shard key** es la decisión clave: debe repartir uniforme y alinearse con los accesos. Estrategias: rango (hotspots), hash (uniforme, sin localidad), directorio (flexible).
> - **Problemas del sharding**: hotspots, JOINs cross-shard (desnormalizar/co-localizar), transacciones cross-shard (Sagas) y rebalanceo costoso.
> - **Consistent hashing**: mapea claves y servidores en un anillo; al cambiar el número de servidores solo se remapea ≈K/N claves (no todas, como con `hash % N`). Los **virtual nodes** reparten la carga uniformemente. Base de DynamoDB/Cassandra.

### Para llevar a la práctica
- [ ] Para un dataset tuyo, decide razonadamente SQL o NoSQL según sus relaciones, consistencia y escala — no por moda.
- [ ] Si tienes (o tendrás) sharding, define cuál sería tu shard key y comprueba que reparte uniforme y co-localiza tus consultas frecuentes.
- [ ] Implementa un consistent hash ring sencillo con virtual nodes para interiorizar por qué minimiza el movimiento de datos.
- [ ] Revisa dónde dependes de JOINs o transacciones que romperían si shardaras esa tabla.

### Recursos
- 📖 *Designing Data-Intensive Applications* — Martin Kleppmann (capítulos 5 y 6: replicación y particionado, la mejor referencia)
- 📖 *System Design Interview, Vol. 1* — Alex Xu (capítulo 5: Consistent Hashing; capítulo 6: Key-Value Store)
- 📄 Karger et al. (1997) — el paper original de consistent hashing
- 📄 *Dynamo: Amazon's Highly Available Key-value Store* (2007)

---
`#diseño-de-sistemas` `#sql-nosql` `#sharding` `#consistent-hashing` `#replicacion`
