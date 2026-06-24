# 🦜 LangChain 03 — Agentes y LangGraph

[[Desarrollo Profesional/LangChain/LangChain|⬅️ Volver a LangChain]] | [[Desarrollo Profesional/LangChain/Páginas/02 - RAG con LangChain|← 02]]

> [!abstract] Introducción
> Una cadena (chain) tiene un flujo fijo. Un **agente** decide en tiempo de ejecución qué hacer: qué herramienta llamar, con qué argumentos, y cuándo parar. Es un bucle "razonar → actuar → observar" gobernado por el LLM. LangChain implementaba agentes con abstracciones que se quedaron cortas; hoy lo serio se hace con **LangGraph**, que modela el agente como un **grafo de estado** explícito, controlable y persistente.

## ¿De qué vamos a hablar?

Qué es un agente y su bucle, cómo definir tools, el agente prefabricado de LangGraph, y por qué un grafo de estado es superior para casos reales.

### Conceptos que vamos a cubrir
- El bucle del agente: razonar, actuar, observar
- Definir tools con el decorador `@tool`
- `create_react_agent` de LangGraph
- Grafos de estado: nodos, aristas y estado persistente

### El bucle del agente

```
        ┌──────────────────────────────────────┐
        ▼                                      │
  [LLM razona] ──▶ ¿necesita una tool? ──sí──▶ [ejecuta tool] ──▶ [observa resultado]
        │                                                              │
        └── no ──▶ [respuesta final]  ◀───────────────────────────────┘
```

---

## El Concepto

### Definir tools

Una tool es una función que el agente puede invocar. El decorador `@tool` usa los type hints y la docstring para describirla al LLM (igual que vimos en [[Desarrollo Profesional/Pydantic/Páginas/03 - PydanticAI y Agentes|PydanticAI]]).

```python
from langchain_core.tools import tool

@tool
def multiplicar(a: int, b: int) -> int:
    """Multiplica dos números enteros."""
    return a * b

@tool
def buscar_clima(ciudad: str) -> str:
    """Devuelve el clima actual de una ciudad."""
    return f"En {ciudad}: 24°C, soleado"
```

### Un agente con LangGraph

`create_react_agent` monta el bucle completo: le das modelo y tools, y él razona, llama herramientas y devuelve la respuesta.

```python
from langgraph.prebuilt import create_react_agent
from langchain_anthropic import ChatAnthropic

modelo = ChatAnthropic(model="claude-sonnet-4-6", temperature=0)
agente = create_react_agent(modelo, tools=[multiplicar, buscar_clima])

respuesta = agente.invoke({
    "messages": [("user", "¿Cuánto es 12 por 8 y qué tiempo hace en Madrid?")]
})
print(respuesta["messages"][-1].content)
# El agente llama a multiplicar(12, 8) y a buscar_clima("Madrid") por su cuenta
```

### Grafos de estado: control explícito

Para flujos reales (validación, ramas condicionales, intervención humana, reintentos) defines tú el grafo: **nodos** (pasos) conectados por **aristas** (transiciones), sobre un **estado** compartido que persiste.

```python
from typing import Annotated, TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages

class Estado(TypedDict):
    messages: Annotated[list, add_messages]  # el estado acumula mensajes

def nodo_modelo(estado: Estado) -> dict:
    respuesta = modelo.invoke(estado["messages"])
    return {"messages": [respuesta]}

grafo = StateGraph(Estado)
grafo.add_node("modelo", nodo_modelo)
grafo.add_edge(START, "modelo")
grafo.add_edge("modelo", END)

app = grafo.compile()
app.invoke({"messages": [("user", "Hola")]})
```

> [!tip] Cuándo agente, cuándo grafo, cuándo nada
> No todo necesita un agente. Si el flujo es fijo, una **cadena** (LCEL) es más barata, rápida y predecible. Usa un **agente** cuando el camino dependa de la entrada. Usa un **grafo de LangGraph** cuando necesites control fino: ramas, ciclos, persistencia del estado o un humano en el bucle. Más autonomía = más coste y menos previsibilidad.

### Persistencia y memoria

LangGraph permite añadir un *checkpointer* para que el estado sobreviva entre invocaciones: así un agente recuerda la conversación o puede pausarse y reanudarse. Es lo que lo hace apto para producción frente a los agentes "de juguete".

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - Un **agente** deja que el LLM decida qué tools llamar y cuándo parar: el bucle **razonar → actuar → observar**.
> - Las **tools** se definen con `@tool`; type hints y docstring las describen al modelo.
> - **`create_react_agent`** de LangGraph monta el bucle completo a partir de modelo + tools.
> - Para control fino usa un **grafo de estado**: nodos, aristas y un estado compartido que persiste.
> - **Elige el mínimo necesario**: cadena (flujo fijo) < agente (camino variable) < grafo (control, ciclos, persistencia).

### Para llevar a la práctica
- [ ] Define dos tools y móntalas en un `create_react_agent`; observa qué herramientas elige
- [ ] Construye un `StateGraph` mínimo con un nodo de modelo
- [ ] Añade un checkpointer para que el agente recuerde mensajes entre invocaciones

### Recursos
- 🌐 langchain-ai.github.io/langgraph — documentación de LangGraph
- 🌐 langchain-ai.github.io/langgraph/agents/agents — agentes prefabricados
- 🌐 python.langchain.com/docs/concepts/tools — tools en LangChain

---
`#langchain` `#langgraph` `#agentes` `#ia`
