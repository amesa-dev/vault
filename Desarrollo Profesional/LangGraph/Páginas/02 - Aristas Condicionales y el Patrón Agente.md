# 🕸️ LangGraph 02 — Aristas Condicionales y el Patrón Agente

[[Desarrollo Profesional/LangGraph/LangGraph|⬅️ Volver a LangGraph]] | [[Desarrollo Profesional/LangGraph/Páginas/01 - Grafos Estado y Nodos|← 01]] | [[Desarrollo Profesional/LangGraph/Páginas/03 - Persistencia Memoria y Human-in-the-Loop|03 →]]

> [!abstract] Introducción
> Un grafo en secuencia no es más que una cadena. La potencia de LangGraph aparece cuando el **camino depende del estado**: una arista condicional decide a qué nodo ir según lo que haya pasado, y eso permite **ramas y ciclos**. Sobre esa idea se construye el patrón más importante del ecosistema —el **agente**: un bucle en el que el LLM decide si llamar a una herramienta o terminar, una y otra vez, hasta resolver la tarea.

## ¿De qué vamos a hablar?

Cómo crear aristas condicionales con `add_conditional_edges`, cómo eso introduce ciclos en el grafo, y cómo ensamblar el patrón ReAct (razonar–actuar–observar) tanto a mano como con el agente prefabricado de LangGraph.

### Conceptos que vamos a cubrir
- Aristas condicionales: una función de enrutado que devuelve el nombre del siguiente nodo
- Ciclos: volver a un nodo anterior para repetir el bucle
- `ToolNode` y `tools_condition`: las piezas listas para agentes
- `create_react_agent`: el atajo de producción

---

## El Concepto

### Aristas condicionales

Una arista normal es fija: "de A siempre a B". Una **arista condicional** delega la decisión en una función de enrutado que lee el estado y devuelve **el nombre del siguiente nodo**.

```python
from langgraph.graph import StateGraph, START, END

def enrutar(estado: Estado) -> str:
    # devuelve la CLAVE del próximo nodo según el estado
    if estado["pasos"] >= 3:
        return "fin"
    return "seguir"

grafo.add_conditional_edges(
    "trabajo",            # nodo de origen
    enrutar,              # función que decide
    {"seguir": "trabajo", "fin": END},  # mapa: valor devuelto -> nodo destino
)
```

Fíjate en `"seguir": "trabajo"`: la arista vuelve al mismo nodo. Eso es un **ciclo**, algo imposible en una cadena lineal y la base de todo agente.

```
        ┌─────────────── seguir ───────────────┐
        ▼                                       │
  START ──▶ [trabajo] ──▶ enrutar ──┤
                                     └── fin ──▶ END
```

> [!info] Los ciclos necesitan un freno
> Un grafo con ciclo puede no terminar nunca. LangGraph aplica un **límite de recursión** (`recursion_limit`, 25 por defecto) que aborta si se superan demasiados pasos. Aun así, tu lógica de enrutado **debe** tener una condición de salida real —como el `pasos >= 3` de arriba— y no confiar solo en el límite.

### El patrón agente, a mano

Un agente es exactamente este esquema: un nodo que llama al LLM y una arista condicional que mira si el LLM pidió usar una herramienta. Si la pidió, vamos al nodo de tools y **volvemos** al LLM con el resultado; si no, terminamos.

```python
from typing import Annotated, TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langgraph.prebuilt import ToolNode, tools_condition
from langchain_anthropic import ChatAnthropic
from langchain_core.tools import tool

@tool
def buscar_clima(ciudad: str) -> str:
    """Devuelve el clima actual de una ciudad."""
    return f"En {ciudad}: 24°C, soleado"

tools = [buscar_clima]
modelo = ChatAnthropic(model="claude-sonnet-4-6", temperature=0).bind_tools(tools)

class Estado(TypedDict):
    messages: Annotated[list, add_messages]

def nodo_modelo(estado: Estado) -> dict:
    return {"messages": [modelo.invoke(estado["messages"])]}

grafo = StateGraph(Estado)
grafo.add_node("modelo", nodo_modelo)
grafo.add_node("tools", ToolNode(tools))   # ejecuta las tools que pidió el LLM

grafo.add_edge(START, "modelo")
# tools_condition: si el último mensaje pide tools -> "tools"; si no -> END
grafo.add_conditional_edges("modelo", tools_condition)
grafo.add_edge("tools", "modelo")          # tras ejecutar tools, vuelve al LLM

app = grafo.compile()
resp = app.invoke({"messages": [("user", "¿Qué tiempo hace en Madrid?")]})
print(resp["messages"][-1].content)
```

Las dos piezas prefabricadas hacen el trabajo pesado:
- **`ToolNode(tools)`**: un nodo que lee las llamadas a herramientas del último mensaje del LLM, las ejecuta y devuelve los resultados como mensajes.
- **`tools_condition`**: la función de enrutado lista para usar; comprueba si el LLM pidió herramientas.

El bucle resultante es el clásico **ReAct**:

```
  START ──▶ [modelo] ──▶ ¿pidió tool? ──no──▶ END
               ▲                │
               │               sí
               │                ▼
               └────────────[tools]
```

### El atajo: `create_react_agent`

Montar ese grafo a mano está bien para entenderlo, pero para el caso estándar LangGraph lo encapsula en una sola llamada. Devuelve un grafo ya compilado, equivalente al de arriba:

```python
from langgraph.prebuilt import create_react_agent

agente = create_react_agent(modelo, tools=tools)
resp = agente.invoke({"messages": [("user", "¿Qué tiempo hace en Madrid?")]})
```

> [!tip] Empieza por el prefabricado, baja al grafo cuando duela
> `create_react_agent` cubre la mayoría de los agentes de tools. Construye el grafo a mano cuando necesites algo que el prefabricado no te da: un nodo de validación antes de responder, varias ramas según el tipo de petición, un paso de aprobación humana o reintentos con lógica propia. Ese control fino —y la persistencia de la siguiente página— es la razón de existir de LangGraph.

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - Una **arista condicional** (`add_conditional_edges`) enruta al siguiente nodo según una función que lee el estado y devuelve un nombre de nodo.
> - Hacer que una arista vuelva a un nodo anterior crea un **ciclo**: la base de cualquier agente. Protégelo con una condición de salida y el `recursion_limit`.
> - El **patrón agente** es: nodo de modelo + `ToolNode` + `tools_condition`, con la arista de tools volviendo al modelo. Es el bucle **ReAct**.
> - **`create_react_agent`** monta ese grafo por ti; baja al grafo manual cuando necesites ramas, validación o human-in-the-loop.

### Para llevar a la práctica
- [ ] Crea un grafo con un ciclo controlado por contador y comprueba que para en la condición, no en el `recursion_limit`
- [ ] Monta el agente ReAct a mano con `ToolNode` y `tools_condition` y dale dos tools
- [ ] Reescríbelo con `create_react_agent` y verifica que el comportamiento es el mismo

### Recursos
- 🌐 langchain-ai.github.io/langgraph/how-tos/branching — aristas condicionales y ramas
- 🌐 langchain-ai.github.io/langgraph/reference/prebuilt — `ToolNode`, `tools_condition`, `create_react_agent`
- 🌐 react-lm.github.io — el paper original de ReAct (Yao et al.)

---
`#langgraph` `#agentes` `#react` `#ia`
