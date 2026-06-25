# 🔁 Loop Engineering 02 — Diseño del Bucle y Herramientas

[[Desarrollo Profesional/Loop Engineering/Loop Engineering|⬅️ Volver a Loop Engineering]] | [[Desarrollo Profesional/Loop Engineering/Páginas/01 - Qué es y el Bucle Agéntico|← 01]] | [[Desarrollo Profesional/Loop Engineering/Páginas/03 - Verificación Evals y Guardarraíles|03 →]]

> [!abstract] Introducción
> Un bucle que funciona en una demo y uno que aguanta tareas reales se diferencian en los detalles del diseño: qué herramientas le das al agente (y cómo las describes), qué información cabe en la ventana de contexto en cada vuelta, dónde vive el estado y cuándo el bucle debe parar. Estas decisiones son el grueso del loop engineering. Aquí van las que más rendimiento dan.

## ¿De qué vamos a hablar?

Del diseño de las cuatro piezas que determinan si tu bucle es fiable: el harness, las herramientas, el contexto y las condiciones de parada.

### Conceptos que vamos a cubrir
- El harness: el código alrededor del modelo que hace girar el bucle
- Diseño de herramientas: pocas, claras y con buenos errores
- Context engineering dentro del bucle: la ventana es un presupuesto
- Estado, memoria y compactación
- Condiciones de parada y control del coste

---

## El Concepto

### El Harness: el Bucle es Código, no Magia

El **harness** es todo el código que rodea al modelo: el que construye el contexto, llama al LLM, ejecuta las herramientas, mete los resultados de vuelta y decide si seguir. El modelo es el cerebro; el harness es el cuerpo y el sistema nervioso. La calidad de un agente depende tanto del harness como del modelo: el mismo LLM rinde radicalmente distinto según el bucle que lo envuelva.

```
        ┌──────────────────── HARNESS ────────────────────┐
        │                                                  │
        │  construir contexto ──▶  LLM  ──▶ parsear acción │
        │         ▲                              │         │
        │         │                              ▼         │
        │   meter resultado  ◀──  ejecutar herramienta     │
        │         │                              │         │
        │         └──────  ¿parar? ◀─────────────┘         │
        └──────────────────────────────────────────────────┘
```

### Diseño de Herramientas

Las herramientas son la API que el agente tiene del mundo. Su diseño es decisivo:

- **Pocas y ortogonales.** Un menú corto de herramientas potentes vence a una lista enorme de funciones solapadas: cuantas más opciones parecidas, más se equivoca el modelo eligiendo. Una herramienta `ejecutar_sql` bien hecha bate a cincuenta endpoints específicos.
- **Descripciones como contrato.** El modelo elige la herramienta leyendo su nombre, su descripción y su esquema de parámetros. Escríbelos como si fueran para un compañero nuevo: claros, con unidades, con ejemplos. Aquí brilla [[Desarrollo Profesional/Pydantic/Páginas/01 - Modelos como Esquema para LLMs|Pydantic como esquema]].
- **Errores que enseñan.** Cuando una tool falla, su mensaje de error vuelve al contexto como observación. Un error útil ("falta el parámetro `fecha`, formato YYYY-MM-DD") permite autocorrección; un `KeyError` pelado, no.

```python
from pydantic import BaseModel, Field

class BuscarPedidosArgs(BaseModel):
    """Busca pedidos de un cliente en un rango de fechas."""
    cliente_id: int = Field(description="ID numérico del cliente")
    desde: str = Field(description="Fecha inicio, formato YYYY-MM-DD")
    hasta: str = Field(description="Fecha fin, formato YYYY-MM-DD")

def buscar_pedidos(args: BuscarPedidosArgs) -> str:
    try:
        fecha_desde = date.fromisoformat(args.desde)
    except ValueError:
        # Error como enseñanza: el agente puede corregir en la siguiente vuelta
        return "ERROR: 'desde' debe tener formato YYYY-MM-DD (ej. 2026-01-15)"
    pedidos = db.buscar(args.cliente_id, fecha_desde, ...)
    return formatear_compacto(pedidos)   # devuelve algo legible y NO gigante
```

### Context Engineering: la Ventana es un Presupuesto

En cada vuelta del bucle, el modelo solo "sabe" lo que cabe en su ventana de contexto. Y la ventana es finita y cara. El problema central de los bucles largos: la conversación crece sin parar (cada acción y cada resultado se acumulan) hasta que satura la ventana, sube el coste y *degrada* la calidad (el modelo se pierde entre tanto ruido).

Tácticas de context engineering dentro del bucle:
- **Mete solo lo relevante.** No vuelques un fichero de 5.000 líneas si el agente necesita una función. Recupera bajo demanda (RAG, ver [[Desarrollo Profesional/IA/Páginas/03 - RAG|IA: RAG]]) en vez de precargar.
- **Comprime los resultados de las tools.** Una tool que devuelve 10.000 filas debe resumir o paginar, no inundar el contexto.
- **Compacta la historia.** Cuando el bucle se alarga, resume las vueltas viejas en una nota breve y descarta el detalle. Es la técnica de *compaction* que usan los agentes de codificación para sostener sesiones largas.

> [!tip] La regla mental
> Trata cada token de contexto como dinero y atención. La pregunta en cada vuelta no es "¿qué puedo meter?", sino "¿qué necesita ver el modelo *ahora mismo* para dar el siguiente paso, y qué puedo sacar?".

### Estado y Memoria

El bucle necesita acordarse de cosas entre vueltas y, a veces, entre sesiones:
- **Estado de la vuelta**: la lista de mensajes (lo que vimos en la página 01). Es la memoria de trabajo de corto plazo.
- **Memoria externa**: cuando la tarea no cabe en la ventana, el agente escribe a un almacén externo (un fichero de notas, una base de datos, un *scratchpad*) y lo relee cuando lo necesita. Sacar información de la ventana y traerla bajo demanda es una de las palancas más potentes para tareas largas.

### Condiciones de Parada y Coste

Un bucle sin frenos es un bucle peligroso (y caro). Define siempre cuándo para:
- **Éxito**: el agente declara la tarea terminada (deja de pedir herramientas, o llama a una tool `finalizar`).
- **Presupuesto**: máximo de pasos, de tokens, de tiempo o de dinero. Un tope duro evita bucles infinitos y facturas sorpresa.
- **Sin progreso**: si el agente repite la misma acción fallida una y otra vez, córtalo. Detectar el bucle-dentro-del-bucle ("loop" improductivo) es parte del oficio.
- **Escalada a humano**: ante una acción peligrosa o ambigua, parar y preguntar (lo vemos en la [[Desarrollo Profesional/Loop Engineering/Páginas/03 - Verificación Evals y Guardarraíles|página 03]]).

```python
# Frenos combinados en el bucle
inicio = time.monotonic()
coste_tokens = 0
acciones_recientes: list[str] = []

while True:
    if pasos >= MAX_PASOS:               break          # presupuesto de pasos
    if time.monotonic() - inicio > 300:  break          # presupuesto de tiempo
    if coste_tokens > MAX_TOKENS:        break          # presupuesto de coste
    if bucle_improductivo(acciones_recientes): break    # sin progreso
    ...
```

El coste es una restricción de diseño de primer orden: usa el [[Desarrollo Profesional/IA/Páginas/01 - LLMs y Fundamentos|modelo adecuado a cada paso]] (un modelo pequeño para tareas mecánicas, uno grande para razonar), cachea el prompt cuando el harness lo permita y mide siempre tokens por tarea.

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - El **harness** —el código que rodea al modelo— pesa tanto como el propio modelo: el mismo LLM rinde distinto según el bucle que lo envuelva.
> - **Herramientas**: pocas y ortogonales, con descripciones que son su contrato y errores que enseñan (vuelven al contexto y permiten autocorrección).
> - La **ventana de contexto es un presupuesto**: mete solo lo relevante, comprime resultados de tools y compacta la historia para sostener bucles largos.
> - Saca lo que no cabe a una **memoria externa** y tráelo bajo demanda.
> - Define siempre **condiciones de parada** (éxito, presupuesto, sin progreso, escalada) y trata el **coste** como restricción de diseño.

### Para llevar a la práctica
- [ ] Reescribe la descripción de una de tus tools como si fuera para un compañero nuevo, con unidades y ejemplo
- [ ] Haz que tus tools devuelvan errores en lenguaje natural accionable, no excepciones crudas
- [ ] Añade compactación: cuando los mensajes pasen de N, resume los viejos en una nota
- [ ] Pon un tope de pasos, de tiempo y de tokens a tu bucle, y un detector de acciones repetidas

### Recursos
- 🌐 anthropic.com/engineering/effective-context-engineering-for-ai-agents — context engineering en bucles largos
- 🌐 anthropic.com/engineering/writing-tools-for-agents — cómo escribir herramientas que el agente use bien
- 🌐 modelcontextprotocol.io — MCP, estándar para exponer herramientas a agentes (ver [[Desarrollo Profesional/IA/Páginas/04 - Agentes y MCP|IA 04]])
- 📖 *Pydantic AI* — agentes tipados con tools (ver [[Desarrollo Profesional/Pydantic/Páginas/03 - PydanticAI y Agentes|Pydantic 03]])

---
`#loop-engineering` `#harness` `#tools` `#context-engineering` `#agentes`
