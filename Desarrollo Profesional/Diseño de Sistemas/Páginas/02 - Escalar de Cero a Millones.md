# 🏛️ Diseño de Sistemas 02 — Escalar de Cero a Millones

[[Desarrollo Profesional/Diseño de Sistemas/Diseño de Sistemas|⬅️ Volver a Diseño de Sistemas]] | [[Desarrollo Profesional/Diseño de Sistemas/Páginas/01 - El Método y Estimaciones|← 01]] | [[Desarrollo Profesional/Diseño de Sistemas/Páginas/03 - Almacenamiento Replicación y Sharding|03 →]]

> [!abstract] Introducción
> La mejor forma de entender las piezas de un sistema distribuido es verlas aparecer una a una, cada una resolviendo un problema concreto de escala. Esta página recorre la historia que cuenta el primer capítulo de Alex Xu: empezamos con un único servidor que lo hace todo, y conforme crece el tráfico vamos añadiendo justo la pieza que el siguiente cuello de botella exige —separar la base de datos, replicar, cachear, usar una CDN, hacer los servidores sin estado, repartir entre datacenters, desacoplar con colas y, finalmente, fragmentar la base de datos—. Al terminar tendrás el mapa mental de por qué los sistemas grandes tienen la forma que tienen.

## ¿De qué vamos a hablar?

La evolución arquitectónica de un sistema desde un servidor único hasta soportar millones de usuarios, pieza a pieza.

### Conceptos que vamos a cubrir
- Escalado vertical vs horizontal
- Separar la capa web de la base de datos
- Balanceo de carga y servidores sin estado
- Replicación de base de datos (lectura/escritura)
- Caché y CDN
- Multi-datacenter, colas de mensajes y sharding

---

## El Concepto

### Empezamos: Un Solo Servidor

Todo cabe en una máquina: la app, la base de datos, todo. El usuario pide por DNS, le llega la IP, el servidor responde. Funciona perfectamente… hasta que crece el tráfico.

```
[Usuario] ──DNS──> [Servidor único: web + app + BD]
```

La primera tentación es **escalado vertical** (scale up): una máquina más potente (más CPU, RAM). Es lo más fácil y a veces lo correcto, pero tiene un techo duro (no hay máquinas infinitas) y **ningún failover**: si esa máquina cae, todo cae. La alternativa es **escalado horizontal** (scale out): más máquinas. Más complejo pero sin techo y con redundancia. Los sistemas grandes escalan horizontalmente; la historia que sigue es cómo.

### Paso 1: Separar Base de Datos de la Capa Web

Lo primero: que web y datos puedan escalar por separado, porque tienen perfiles de recursos distintos.

```
[Usuarios] ─> [Servidor(es) web] ─> [Servidor de base de datos]
```

Ahora puedes elegir la BD adecuada ([[Desarrollo Profesional/Diseño de Sistemas/Páginas/03 - Almacenamiento Replicación y Sharding|SQL vs NoSQL, página 03]]) y dimensionar cada capa a su ritmo.

### Paso 2: Balanceador de Carga + Servidores sin Estado

Un solo servidor web es un punto único de fallo y un cuello de botella. Añadimos varios detrás de un **load balancer**, que reparte el tráfico y oculta los servidores tras una IP pública.

```
                 ┌─> [Web 1]
[Usuarios] ─> [LB] ─> [Web 2]  ─> [BD]
                 └─> [Web 3]
```

Para que esto funcione, los servidores web deben ser **sin estado (stateless)**: ninguna petición debe depender de que vuelva al *mismo* servidor. ¿Dónde va el estado de sesión entonces? Fuera de los servidores web: en un almacén compartido (Redis, BD). Así cualquier servidor atiende cualquier petición, puedes añadir/quitar servidores libremente y la caída de uno no tira sesiones. **El estado es el enemigo del escalado horizontal**; empújalo a una capa de datos compartida.

### Paso 3: Replicación de Base de Datos

Ahora la BD es el cuello de botella y el punto único de fallo. La **replicación maestro-réplica** lo ataca: el **maestro** (primary) recibe las **escrituras**; las **réplicas** (secundarios) copian sus datos y sirven **lecturas**.

```
                          ┌─> [Réplica 1] ─┐
[Web] ─escrituras─> [Maestro]              ├─ lecturas
                          └─> [Réplica 2] ─┘
```

Encaja perfecto con la realidad de que la mayoría de los sistemas leen mucho más de lo que escriben: añades réplicas para absorber lecturas. Beneficios: rendimiento de lectura, redundancia (si una réplica cae, otras sirven; si el maestro cae, una réplica se promueve) y disponibilidad. El precio: **lag de replicación** → consistencia eventual entre maestro y réplicas (recuerda [[Desarrollo Profesional/Sistemas Distribuidos/Páginas/01 - Fundamentos y CAP|PACELC]]: lo que acabas de escribir puede no verse aún en una réplica — el patrón "read-your-writes" obliga a leer del maestro tras escribir).

### Paso 4: Caché

Las lecturas siguen golpeando la BD (lento). Ponemos una **caché en memoria** ([[Desarrollo Profesional/Redis/Redis|Redis]]) delante: la app mira primero la caché (rápido, RAM) y solo va a la BD en *miss* (patrón cache-aside).

```
[Web] ─> [Caché Redis] ──miss──> [BD]
```

Una caché bien colocada absorbe la inmensa mayoría de las lecturas y reduce drásticamente la carga de la BD. Las decisiones (TTL, invalidación, eviction, stampede) están en [[Desarrollo Profesional/Redis/Páginas/02 - Patrones de Uso|Redis 02]] — y la invalidación sigue siendo uno de los dos problemas difíciles.

### Paso 5: CDN

Los recursos estáticos (imágenes, JS, CSS, vídeo) no deberían viajar desde tu datacenter hasta cada usuario del mundo (recuerda: ~150 ms intercontinentales). Una **CDN** (Content Delivery Network) es una red de servidores distribuidos geográficamente que cachean contenido estático **cerca del usuario**.

```
[Usuario en Tokio] ─> [CDN edge en Tokio] (sirve la imagen cacheada)
                        └─miss─> [Origen en Europa]
```

La CDN reduce latencia (el contenido está cerca), descarga tu origen y absorbe picos. Es, en esencia, una caché geográficamente distribuida para contenido estático.

### Paso 6: Multi-Datacenter

Para servir bien a usuarios globales y sobrevivir a la caída de un datacenter entero, replicas el sistema en **varios datacenters** y enrutas a cada usuario al más cercano (geo-DNS).

```
[Usuarios EU] ─> [Datacenter Europa] ⇄ replicación ⇄ [Datacenter US] <─ [Usuarios US]
```

Esto introduce el reto difícil de la **sincronización de datos entre datacenters** (replicación de BD a través de continentes, con su latencia y sus conflictos) y el redireccionamiento de tráfico cuando un DC cae. Es complejo, pero es lo que da disponibilidad global y resiliencia geográfica.

### Paso 7: Colas de Mensajes (Desacoplar)

Conforme el sistema crece, algunas operaciones son lentas (procesar un vídeo, enviar emails, generar miniaturas) y no deben bloquear la respuesta al usuario. Una **cola de mensajes** ([[Desarrollo Profesional/Sistemas Distribuidos/Páginas/02 - Mensajería y Brokers|Mensajería]]) desacopla productores de consumidores:

```
[Web] ─publica─> [Cola] ─> [Workers] (procesan asíncronamente)
```

El servidor web responde al instante ("tu vídeo se está procesando") y unos workers consumen la cola a su ritmo. Beneficios: la web no se bloquea, productores y consumidores escalan independientemente, y la cola absorbe picos (buffer). Es la base del procesamiento asíncrono a escala.

### Paso 8: Sharding de la Base de Datos

Llega el momento en que **una sola base de datos no aguanta el volumen de escrituras** (las réplicas escalan lecturas, no escrituras) ni los datos caben en una máquina. La solución es **sharding** (particionado horizontal): repartir los datos entre varias bases de datos, cada una con un subconjunto.

```
shard = hash(user_id) % num_shards

[Web] ─> shard 0: usuarios A...  
        shard 1: usuarios B...  
        shard 2: usuarios C...
```

Cada shard es una BD independiente que maneja su porción. Esto escala las escrituras y el almacenamiento horizontalmente, pero **introduce mucha complejidad** (lo desarrolla [[Desarrollo Profesional/Diseño de Sistemas/Páginas/03 - Almacenamiento Replicación y Sharding|la página 03]]): elegir la shard key, joins entre shards, rebalanceo, hotspots. El sharding es potente y caro; se llega a él cuando las opciones anteriores se agotan.

### El Mapa Completo

Juntando todo, la arquitectura de "millones de usuarios" tiene esta forma — y ahora entiendes *por qué* cada pieza está ahí:

```
[Usuarios] ─> [DNS/CDN] ─> [Load Balancer] ─> [Web servers stateless]
                                                    │
                          ┌─────────────────────────┼──────────────┐
                          ▼                          ▼              ▼
                   [Caché Redis]            [Cola de mensajes]  [BD: maestro
                                                    │            + réplicas,
                                              [Workers]          sharded]
              (sesión y estado fuera de los web servers, en almacén compartido)
```

La lección de fondo: **no diseñas esto de golpe.** Cada pieza resuelve un cuello de botella concreto. Empiezas simple y añades complejidad solo cuando la escala la justifica. Añadir piezas antes de tiempo es complejidad gratuita.

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - **Vertical** (máquina más potente: fácil, con techo y sin failover) vs **horizontal** (más máquinas: complejo pero sin techo y redundante). Los grandes escalan horizontal.
> - La evolución por cuellos de botella: separar BD de web → **load balancer + servidores stateless** (estado fuera, en almacén compartido) → **réplicas** de lectura → **caché** → **CDN** para estático → **multi-datacenter** → **colas** para desacoplar lo lento → **sharding** cuando una BD no aguanta las escrituras.
> - **El estado es el enemigo del escalado horizontal**: empújalo a una capa de datos compartida para que cualquier servidor atienda cualquier petición.
> - Las **réplicas escalan lecturas** (consistencia eventual por el lag); el **sharding escala escrituras y almacenamiento** (a costa de mucha complejidad).
> - **No diseñes el mapa completo de golpe**: cada pieza resuelve un problema real. Añadir antes de tiempo es complejidad gratuita.

### Para llevar a la práctica
- [ ] Dibuja la arquitectura actual de un sistema tuyo y sitúa en qué "paso" de esta evolución está.
- [ ] Identifica si tus servidores guardan estado de sesión local. Si sí, es un freno al escalado horizontal.
- [ ] Localiza tu mayor cuello de botella de lectura. ¿Lo resolverían réplicas, caché o CDN?
- [ ] Pregúntate qué operaciones lentas podrían moverse a una cola para no bloquear al usuario.

### Recursos
- 📖 *System Design Interview, Vol. 1* — Alex Xu (capítulo 1: "Scale From Zero To Millions Of Users")
- 🌐 github.com/donnemartin/system-design-primer
- 📖 *Designing Data-Intensive Applications* — Martin Kleppmann (replicación, particionado)
- 🌐 highscalability.com — arquitecturas reales de empresas a escala

---
`#diseño-de-sistemas` `#escalabilidad` `#replicacion` `#sharding`
