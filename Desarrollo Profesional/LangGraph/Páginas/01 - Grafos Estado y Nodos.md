# 🕸️ LangGraph 01 — Grafos, Estado y Nodos

[[Desarrollo Profesional/LangGraph/LangGraph|⬅️ Volver a LangGraph]] | [[Desarrollo Profesional/LangGraph/Páginas/02 - Aristas Condicionales y el Patrón Agente|02 →]]

> [!abstract] Introducción
> Toda aplicación de LangGraph es un grafo: un conjunto de **nodos** (funciones que hacen el trabajo) conectados por **aristas** (qué se ejecuta después) que comparten un **estado**. El estado es el corazón del modelo: es el objeto que fluye por el grafo, cada nodo lo lee y devuelve actualizaciones, y unas funciones llamadas *reducers* deciden cómo se fusionan esas actualizaciones. Si entiendes el estado, entiendes LangGraph.

## ¿De qué vamos a hablar?

Construiremos el grafo más simple posible y subiremos desde ahí: qué es el estado y cómo se declara, qué es un reducer y por qué `add_messages` es especial, cómo se escribe un nodo, y cómo se conectan con `START` y `END` antes de compilar y ejecutar.

### Conceptos que vamos a cubrir
- El estado como `TypedDict` y por qué es inmutable de hecho
- Reducers: cómo se fusionan las actualizaciones de cada nodo
- Nodos: funciones `estado -> dict`
- `StateGraph`, `START`, `END`, `compile()` e `invoke()`

---

## El Concepto

### El estado: el objeto que fluye

El estado es un esquema —normalmente un `TypedDict`— que define qué datos viajan por el grafo. Los nodos **no mutan** el estado: devuelven un diccionario con las claves que quieren actualizar, y LangGraph aplica esas actualizaciones.

```python
from typing import TypedDict

class Estado(TypedDict):
    pregunta: str
    respuesta: str
    pasos: int
```

Por defecto, cada actualización **sobrescribe** la clave anterior. Si un nodo devuelve `{"respuesta": "hola"}`, la respuesta pasa a ser `"hola"`, se pierda lo que hubiera.

### Reducers: controlar cómo se fusiona

A veces no quieres sobrescribir, sino **acumular**. Para eso anotas la clave con un *reducer*: una función `(valor_actual, actualización) -> nuevo_valor`. El caso más común es una lista a la que se va añadiendo:

```python
from typing import Annotated, TypedDict
import operator

class Estado(TypedDict):
    # operator.add concatena listas en vez de reemplazarlas
    logs: Annotated[list[str], operator.add]
    total: int  # sin reducer: se sobrescribe

# Si el estado tiene logs=["a"] y un nodo devuelve {"logs": ["b"]},
# el reducer las fusiona -> logs=["a", "b"]
```

> [!info] `add_messages`, el reducer estrella
> Para conversaciones con LLMs, LangGraph trae `add_messages`. No es un simple `+`: añade mensajes nuevos, pero además **actualiza por id** (si llega un mensaje con el mismo id, lo reemplaza) y normaliza formatos. Es el reducer que verás en casi todos los agentes.

```python
from typing import Annotated, TypedDict
from langgraph.graph.message import add_messages

class Estado(TypedDict):
    messages: Annotated[list, add_messages]
```

### Nodos: el trabajo

Un nodo es una función que **recibe el estado y devuelve un diccionario** con las actualizaciones. Nada más. Puede llamar a un LLM, consultar una BD o hacer un cálculo.

```python
def saludar(estado: Estado) -> dict:
    nombre = estado["pregunta"]
    # MAL: intentar mutar el estado directamente
    # estado["respuesta"] = f"Hola {nombre}"   # no es así como funciona

    # BIEN: devolver solo las claves a actualizar
    return {"respuesta": f"Hola {nombre}", "pasos": estado["pasos"] + 1}
```

### Montar y compilar el grafo

`START` marca dónde entra la ejecución y `END` dónde termina. Añades nodos, los conectas con aristas, y `compile()` produce una app ejecutable (un *Runnable*, igual que en LCEL).

```python
from langgraph.graph import StateGraph, START, END

grafo = StateGraph(Estado)
grafo.add_node("saludar", saludar)
grafo.add_edge(START, "saludar")   # la entrada va al nodo "saludar"
grafo.add_edge("saludar", END)     # tras "saludar", termina

app = grafo.compile()

resultado = app.invoke({"pregunta": "Andrés", "respuesta": "", "pasos": 0})
print(resultado)
# {'pregunta': 'Andrés', 'respuesta': 'Hola Andrés', 'pasos': 1}
```

El diagrama mental de este grafo mínimo:

```
  START ──▶ [saludar] ──▶ END
```

### Varios nodos en secuencia

Encadenar nodos es añadir aristas entre ellos. El estado se va enriqueciendo paso a paso:

```python
def normalizar(estado: Estado) -> dict:
    return {"pregunta": estado["pregunta"].strip().title()}

grafo = StateGraph(Estado)
grafo.add_node("normalizar", normalizar)
grafo.add_node("saludar", saludar)
grafo.add_edge(START, "normalizar")
grafo.add_edge("normalizar", "saludar")  # secuencia
grafo.add_edge("saludar", END)
app = grafo.compile()
```

```
  START ──▶ [normalizar] ──▶ [saludar] ──▶ END
```

> [!tip] Mantén los nodos pequeños y puros
> Un nodo debería hacer **una cosa** y depender solo del estado que recibe. Así el grafo se lee como un diagrama, los nodos se testean aislados (les pasas un dict, compruebas el dict que devuelven) y las ramas y ciclos de la siguiente página se vuelven triviales de razonar.

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - Una app de LangGraph es un **grafo**: nodos (trabajo) + aristas (orden) sobre un **estado** compartido.
> - El estado se declara como `TypedDict`; los nodos **no lo mutan**, devuelven un dict con las claves a actualizar.
> - Por defecto cada clave se **sobrescribe**; con un **reducer** (`Annotated[tipo, fn]`) se **fusiona** —`operator.add` acumula listas.
> - **`add_messages`** es el reducer para conversaciones: añade y actualiza mensajes por id.
> - Se monta con `StateGraph`, `add_node`, `add_edge`, `START`/`END`, y `compile()` devuelve un Runnable que ejecutas con `invoke()`.

### Para llevar a la práctica
- [ ] Define un `Estado` con una clave normal y otra con reducer `operator.add`; comprueba la diferencia al ejecutar dos nodos
- [ ] Construye un grafo de tres nodos en secuencia y dibuja su diagrama mental
- [ ] Cambia la clave de mensajes a `Annotated[list, add_messages]` y observa cómo acumula

### Recursos
- 🌐 langchain-ai.github.io/langgraph/concepts/low_level — estado, nodos, aristas y reducers
- 🌐 langchain-ai.github.io/langgraph/how-tos/state-reducers — reducers en detalle
- 📖 *LangGraph docs* — "StateGraph" en la referencia de la API

---
`#langgraph` `#estado` `#grafos` `#ia`
