# 🕸️ LangGraph 03 — Persistencia, Memoria y Human-in-the-Loop

[[Desarrollo Profesional/LangGraph/LangGraph|⬅️ Volver a LangGraph]] | [[Desarrollo Profesional/LangGraph/Páginas/02 - Aristas Condicionales y el Patrón Agente|← 02]]

> [!abstract] Introducción
> Un agente sin memoria empieza de cero en cada llamada: no recuerda la conversación, no puede pausarse ni reanudarse, y no admite que un humano revise antes de actuar. LangGraph resuelve esto con una pieza elegante: el **checkpointer**, que guarda el estado del grafo tras cada paso. De esa única capacidad nacen tres superpoderes —**memoria** entre invocaciones, **human-in-the-loop** (pausar para que decida una persona) y **streaming** del progreso— que son justo lo que hace falta para llevar un agente a producción.

## ¿De qué vamos a hablar?

Qué es un checkpointer y cómo se conecta, cómo el `thread_id` separa conversaciones, cómo `interrupt` pausa el grafo para meter a un humano en el bucle, y cómo emitir el progreso con streaming.

### Conceptos que vamos a cubrir
- Checkpointers: `MemorySaver` y persistencia real (SQLite/Postgres)
- `thread_id`: identidad de la conversación y memoria automática
- `interrupt` y `Command`: pausar y reanudar con decisión humana
- Streaming de pasos y tokens

---

## El Concepto

### Checkpointers: guardar el estado tras cada paso

Pasas un *checkpointer* a `compile()` y LangGraph guarda un **snapshot del estado** después de cada nodo. Para desarrollo, `MemorySaver` lo guarda en memoria; en producción usas un backend persistente (SQLite, Postgres).

```python
from langgraph.checkpoint.memory import MemorySaver

memoria = MemorySaver()
app = grafo.compile(checkpointer=memoria)
```

### `thread_id`: una conversación, una memoria

Con un checkpointer activo, cada ejecución pertenece a un **hilo** identificado por `thread_id`, que pasas en la config. Misma `thread_id` → el estado se **carga, continúa y se vuelve a guardar**. Así la memoria es automática: no gestionas tú el historial.

```python
config = {"configurable": {"thread_id": "andres-1"}}

app.invoke({"messages": [("user", "Me llamo Andrés")]}, config)
# segunda llamada, MISMO thread_id: el agente recuerda
resp = app.invoke({"messages": [("user", "¿Cómo me llamo?")]}, config)
print(resp["messages"][-1].content)   # "Te llamas Andrés"

# otro thread_id = conversación nueva, sin memoria de la anterior
otra = {"configurable": {"thread_id": "marta-1"}}
```

> [!info] Memoria a corto y a largo plazo
> El checkpointer es la **memoria a corto plazo**: el estado de *este* hilo (la conversación en curso). Para **memoria a largo plazo** —datos que persisten entre hilos y usuarios, como preferencias— LangGraph ofrece el `Store`, un almacén clave-valor aparte que consultas desde los nodos. No confundas las dos: una es el hilo, la otra es el conocimiento duradero.

### Inspeccionar el estado guardado

Como cada paso queda registrado, puedes leer el estado actual o el historial completo de un hilo:

```python
snapshot = app.get_state(config)
print(snapshot.values)        # estado actual del hilo
print(snapshot.next)          # qué nodo se ejecutaría a continuación

for paso in app.get_state_history(config):
    print(paso.config, paso.values)   # recorrido completo, útil para "time travel"
```

### Human-in-the-loop con `interrupt`

Porque el estado se persiste, el grafo puede **pausarse** en mitad de la ejecución, esperar a que un humano decida, y **reanudarse** desde el mismo punto. Dentro de un nodo, `interrupt` detiene el grafo y expone un valor a quien lo opera.

```python
from langgraph.types import interrupt, Command

def aprobar_pago(estado: Estado) -> dict:
    # el grafo se pausa aquí y devuelve este payload a quien lo invocó
    decision = interrupt({"pregunta": "¿Apruebas el pago de 500€?"})
    if decision != "sí":
        return {"messages": [("assistant", "Pago cancelado.")]}
    return {"messages": [("assistant", "Pago realizado.")]}

# 1) primera invocación: corre hasta el interrupt y se detiene
app.invoke({"messages": [("user", "Paga la factura")]}, config)

# 2) un humano revisa y reanuda con su decisión (mismo thread_id)
app.invoke(Command(resume="sí"), config)
```

Este patrón —pausar antes de una acción sensible (un pago, un borrado, enviar un correo) y pedir confirmación humana— es la guardarraíl que diferencia un agente de producción de un experimento. Conecta directo con las ideas de [[Desarrollo Profesional/Loop Engineering/Páginas/03 - Verificación Evals y Guardarraíles|verificación y human-in-the-loop]].

### Streaming: ver el progreso

Un agente puede tardar varios segundos en cerrar su bucle. En vez de esperar al final, `stream` emite resultados a medida que ocurren; el modo elegido decide qué se emite.

```python
# modo "updates": emite las actualizaciones de estado de cada nodo
for evento in app.stream({"messages": [("user", "Hola")]}, config, stream_mode="updates"):
    print(evento)

# modo "messages": emite los tokens del LLM según se generan (para UI tipo chat)
for token, meta in app.stream({"messages": [("user", "Hola")]}, config, stream_mode="messages"):
    print(token.content, end="", flush=True)
```

> [!tip] Persistencia = producción
> En cuanto añades un checkpointer, ganas las tres cosas que pide un sistema real: **memoria** sin gestionarla a mano, **resiliencia** (si un paso falla, reanudas desde el último checkpoint en vez de empezar de cero) y la posibilidad de **meter a un humano** en los puntos críticos. Empieza con `MemorySaver` para prototipar y cambia a un backend persistente (`langgraph-checkpoint-sqlite` o `-postgres`) sin tocar la lógica del grafo.

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - Un **checkpointer** guarda el estado tras cada paso; `MemorySaver` para desarrollo, SQLite/Postgres para producción.
> - El **`thread_id`** identifica la conversación: misma id → el estado se carga y continúa, dando **memoria a corto plazo** automática. El `Store` cubre la memoria a largo plazo entre hilos.
> - `get_state` y `get_state_history` permiten **inspeccionar** el estado y hacer *time travel*.
> - **`interrupt` + `Command(resume=...)`** pausan y reanudan el grafo: la base del **human-in-the-loop** para acciones sensibles.
> - **`stream`** emite el progreso (`updates`) o los tokens del LLM (`messages`) en vez de esperar al final.

### Para llevar a la práctica
- [ ] Compila un agente con `MemorySaver` y comprueba que recuerda tu nombre con el mismo `thread_id` y lo olvida con otro
- [ ] Añade un nodo con `interrupt` antes de una acción "peligrosa" y reanúdalo con `Command(resume=...)`
- [ ] Recorre `get_state_history` de un hilo para ver el rastro de checkpoints
- [ ] Cambia `invoke` por `stream` con `stream_mode="messages"` y muestra los tokens en vivo

### Recursos
- 🌐 langchain-ai.github.io/langgraph/concepts/persistence — checkpointers, threads y estado
- 🌐 langchain-ai.github.io/langgraph/concepts/human_in_the_loop — `interrupt` y aprobación humana
- 🌐 langchain-ai.github.io/langgraph/how-tos/streaming — modos de streaming

---
`#langgraph` `#persistencia` `#human-in-the-loop` `#ia`
