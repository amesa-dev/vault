# 🕸️ LangGraph

[[Desarrollo Profesional/Inicio Profesional|⬅️ Volver a Desarrollo Profesional]]

> [!abstract] Sobre esta sección
> LangGraph es la librería de [LangChain](https://www.langchain.com) para construir aplicaciones **agénticas con estado**, modeladas como un **grafo**: nodos (pasos) conectados por aristas (transiciones) que operan sobre un estado compartido. Frente a una cadena de flujo fijo, un grafo permite **ciclos, ramas condicionales, persistencia e intervención humana** — justo lo que separa un agente de juguete de uno apto para producción. Esta sección cubre el modelo de grafos y estado, el patrón agente con tools, y la persistencia con checkpoints.

---

## 📚 Páginas de esta sección

1. [[Desarrollo Profesional/LangGraph/Páginas/01 - Grafos Estado y Nodos|01 — Grafos, Estado y Nodos]] — `StateGraph`, el estado como `TypedDict`, reducers, nodos y compilación
2. [[Desarrollo Profesional/LangGraph/Páginas/02 - Aristas Condicionales y el Patrón Agente|02 — Aristas Condicionales y el Patrón Agente]] — `add_conditional_edges`, ciclos, `ToolNode` y el bucle ReAct
3. [[Desarrollo Profesional/LangGraph/Páginas/03 - Persistencia Memoria y Human-in-the-Loop|03 — Persistencia, Memoria y Human-in-the-Loop]] — checkpointers, `thread_id`, `interrupt` y streaming

---

## ¿Qué es LangGraph en una frase?

Un motor para describir agentes como **máquinas de estados ejecutables**: defines el estado, los pasos (nodos) y las transiciones (aristas), y LangGraph se encarga del bucle, la persistencia y las pausas.

> [!tip] LangGraph frente a LangChain
> No compiten: **LangChain** aporta los componentes (modelos, prompts, parsers, tools) y **LangGraph** los orquesta cuando el flujo necesita ciclos o estado. Si vienes de [[Desarrollo Profesional/LangChain/Páginas/03 - Agentes y LangGraph|Agentes en LangChain]], aquí profundizamos en el grafo. Para el porqué del bucle agéntico en abstracto, mira [[Desarrollo Profesional/Loop Engineering/Loop Engineering|Loop Engineering]].

---

## 🔗 Recursos Generales
- 🌐 langchain-ai.github.io/langgraph — documentación oficial de LangGraph
- 🌐 langchain-ai.github.io/langgraph/concepts — conceptos: grafos, estado, persistencia
- 🌐 langchain-ai.github.io/langgraph/tutorials — tutoriales paso a paso

---
`#langgraph` `#ia` `#agentes` `#indice`
