# 🔀 CQRS 03 — Read Models, Eventos y Escalado

[[Desarrollo Profesional/CQRS/CQRS|⬅️ Volver a CQRS]] | [[Desarrollo Profesional/CQRS/Páginas/02 - Implementación en Python|← 02]]

> [!abstract] Introducción
> Hasta aquí, lectura y escritura compartían base de datos. El nivel más ambicioso de CQRS separa también el **almacenamiento**: un modelo de escritura optimizado para reglas y un (o varios) **read models** optimizados para consultas, mantenidos al día por eventos. Esto desbloquea una escala enorme, pero a cambio te obliga a convivir con la **consistencia eventual**. Aquí está el verdadero punto donde CQRS deja de ser gratis.

## ¿De qué vamos a hablar?

De cómo se construyen y mantienen los read models con eventos, qué implica la consistencia eventual para la UI, cómo se relaciona CQRS con Event Sourcing y, sobre todo, de cuándo NO usar nada de esto.

### Conceptos que vamos a cubrir
- Read models y proyecciones: vistas materializadas que sirven una pantalla
- Proyectores: cómo los eventos actualizan los read models
- Consistencia eventual y sus consecuencias en la experiencia de usuario
- La relación (no obligatoria) entre CQRS y Event Sourcing
- Cuándo CQRS es la herramienta equivocada

---

## El Concepto

### Read Models: una Vista por Pantalla

Un **read model** (o proyección) es una representación de los datos pensada para una consulta o pantalla concreta, ya masticada: desnormalizada, sin joins, lista para servir. No tiene reglas de negocio; es caché con forma de tabla.

```
            ESCRITURA                          LECTURA
   ┌────────────────────────┐        ┌──────────────────────────┐
   │  Modelo de dominio rico │        │  Read model: vista plana  │
   │  (PostgreSQL normalizado)│ ─────▶ │  resumen_pedidos (Redis)  │
   │  pedidos, lineas, pagos │ eventos│  detalle_cliente (Mongo)  │
   └────────────────────────┘        │  busqueda_pedidos (ES)    │
        commands                      └──────────────────────────┘
        (1 modelo)                        queries (N modelos)
```

Puedes tener **varios** read models del mismo dato, cada uno en el almacén que mejor le va: una vista de resumen en [[Desarrollo Profesional/Redis/Redis|Redis]], un índice de búsqueda en [[Desarrollo Profesional/Kibana/Kibana|Elasticsearch]], un detalle en un documento. El lado de escritura no sabe ni le importa cuántos hay.

### Proyectores: Mantener el Read Model al Día

El read model se alimenta de los **Domain Events** que produce el lado de escritura (los vimos en [[Desarrollo Profesional/DDD/Páginas/03 - Patrones Avanzados|DDD: patrones avanzados]]). Un **proyector** escucha esos eventos y actualiza la proyección.

```python
from dataclasses import dataclass
from uuid import UUID

# Eventos que vienen del lado de escritura
@dataclass(frozen=True)
class PedidoConfirmado:
    pedido_id: UUID
    cliente_id: UUID
    total: float
    num_lineas: int

@dataclass(frozen=True)
class PedidoCancelado:
    pedido_id: UUID
    motivo: str

# Un proyector mantiene la tabla desnormalizada que leen las queries
class ResumenPedidosProyector:
    def __init__(self, db_lectura) -> None:
        self._db = db_lectura

    def cuando_confirmado(self, e: PedidoConfirmado) -> None:
        self._db.execute(
            "INSERT INTO vista_resumen_pedidos (id, cliente_id, estado, total, num_lineas) "
            "VALUES (%s, %s, 'confirmado', %s, %s) "
            "ON CONFLICT (id) DO UPDATE SET estado = 'confirmado'",
            (str(e.pedido_id), str(e.cliente_id), e.total, e.num_lineas),
        )

    def cuando_cancelado(self, e: PedidoCancelado) -> None:
        self._db.execute(
            "UPDATE vista_resumen_pedidos SET estado = 'cancelado' WHERE id = %s",
            (str(e.pedido_id),),
        )
```

Los proyectores se suscriben al mismo `EventBus` que vimos en CQRS 02, o consumen de un broker ([[Desarrollo Profesional/Sistemas Distribuidos/Páginas/02 - Mensajería y Brokers|colas y logs]]) cuando escritura y lectura están en procesos distintos.

### Consistencia Eventual: el Peaje

Si el read model se actualiza por eventos asíncronos, hay una ventana —milisegundos normalmente— en la que **acabas de escribir pero la lectura aún no lo refleja**. Esto es consistencia eventual, y cambia cómo diseñas la UI:

- **MAL**: confirmo el pedido (command) y, justo después, leo el listado (query) esperando verlo confirmado. Puede que aún no esté.
- **BIEN**: la UI asume el cambio de forma optimista (lo pinta confirmado) y reconcilia cuando llega la confirmación; o el command devuelve lo justo para que el cliente no tenga que releer; o se acepta el pequeño retardo.

> [!info] No siempre hay que pagar este peaje
> El nivel 2 de CQRS (misma base de datos, página 02) es **fuertemente consistente**: lees lo que acabas de escribir. La consistencia eventual solo aparece cuando separas almacenes y sincronizas por eventos asíncronos. No la heredas solo por usar commands y queries.

### CQRS y Event Sourcing: Parientes, no Gemelos

Se confunden constantemente, pero son independientes:

- **CQRS** = separar los modelos de lectura y escritura.
- **Event Sourcing** = persistir el estado como una secuencia de eventos en lugar del estado actual (ver [[Desarrollo Profesional/DDD/Páginas/03 - Patrones Avanzados|DDD 03]]).

Puedes hacer CQRS sin Event Sourcing (lo más común) y Event Sourcing sin CQRS. Ahora bien, se llevan de maravilla: si ya guardas todo como eventos, esos mismos eventos son el combustible perfecto para construir y reconstruir read models. De ahí que **Event Sourcing + CQRS** sea la combinación estrella en sistemas de alta escala.

```
Event Store  ──(stream de eventos)──▶  Proyector A ──▶ Read model SQL
   (verdad)                            Proyector B ──▶ Read model Redis
                                       Proyector C ──▶ Índice de búsqueda
```

Una ventaja preciosa: si necesitas una nueva vista, creas un proyector nuevo y **reproduces el histórico de eventos** para construirla desde cero. La verdad está en los eventos; los read models son desechables y reconstruibles.

### Cuándo NO Usar CQRS

CQRS (sobre todo en nivel 3) es una herramienta especializada. Es mala idea cuando:

- **El CRUD te basta.** Si leer y escribir usan el mismo modelo sin fricción, separar solo añade clases y ceremonia.
- **El equipo no domina la consistencia eventual.** Introducir read models asíncronos sin entender sus consecuencias genera bugs sutiles y desconfianza en los datos.
- **Lo aplicas a todo el sistema.** CQRS es una decisión **por bounded context**, no global. Habrá contextos que lo agradezcan y otros donde sea ruido.
- **Buscas una bala de plata de rendimiento.** A veces un índice bien puesto en [[Desarrollo Profesional/PostgreSQL/Páginas/02 - Indexación y Performance|PostgreSQL]] o una caché resuelven el problema sin rearquitecturar nada.

La pregunta correcta no es "¿uso CQRS?", sino "¿este contexto concreto tiene lecturas y escrituras con necesidades tan dispares que merece dos modelos?".

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - Un **read model** es una vista desnormalizada por pantalla/consulta, sin reglas: caché con forma de tabla. Puedes tener varios, cada uno en su almacén ideal.
> - Los **proyectores** escuchan los Domain Events del lado de escritura y mantienen los read models al día.
> - Separar almacenes trae **consistencia eventual**: hay una ventana donde la lectura va por detrás de la escritura, y la UI debe diseñarse para ello. El nivel 2 (misma BD) no tiene este peaje.
> - **CQRS ≠ Event Sourcing**, pero se combinan de fábula: los eventos alimentan y reconstruyen los read models; las vistas pasan a ser desechables.
> - **No abuses**: CQRS se decide por bounded context y solo cuando lectura y escritura divergen de verdad. El CRUD simple casi siempre gana.

### Para llevar a la práctica
- [ ] Coge un listado costoso de tu app y diséñalo como un read model desnormalizado
- [ ] Escribe un proyector que actualice ese read model a partir de un Domain Event
- [ ] Identifica dónde tu UI asumiría consistencia fuerte y replantéala para tolerar retardo
- [ ] Antes de adoptar CQRS en un contexto, escribe en una frase por qué leer y escribir divergen ahí

### Recursos
- 🌐 microservices.io/patterns/data/cqrs.html — el patrón con sus pros y contras bien listados
- 📄 Greg Young — charlas sobre CQRS + Event Sourcing (busca *"CQRS and Event Sourcing"*)
- 📖 *Designing Data-Intensive Applications* — Martin Kleppmann (vistas materializadas y derivación de datos)
- 🌐 martinfowler.com/bliki/CQRS.html — la advertencia de Fowler sobre no usarlo de más

---
`#cqrs` `#read-model` `#event-sourcing` `#consistencia-eventual` `#escalado`
