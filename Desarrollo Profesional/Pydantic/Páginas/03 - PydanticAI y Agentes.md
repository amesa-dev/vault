# 🧱 Pydantic 03 — PydanticAI y Agentes

[[Desarrollo Profesional/Pydantic/Pydantic|⬅️ Volver a Pydantic]] | [[Desarrollo Profesional/Pydantic/Páginas/02 - Salida Estructurada y Tool Calling|← 02]]

> [!abstract] Introducción
> **PydanticAI** es el framework de agentes del equipo de Pydantic: lleva el rigor del tipado a la construcción de agentes LLM. Un `Agent` tiene un tipo de resultado (validado con Pydantic), un tipo de dependencias (para inyectar lo que necesite, como un cliente de BD), y un conjunto de **tools** que son funciones Python normales con argumentos tipados. Es la evolución natural de lo visto en las páginas anteriores: del esquema individual al agente completo.

## ¿De qué vamos a hablar?

Qué aporta PydanticAI frente a usar la API a pelo, cómo definir un agente con resultado tipado, cómo registrar tools, e inyección de dependencias para tools que necesitan contexto externo.

### Conceptos que vamos a cubrir
- El `Agent`: modelo, resultado tipado y system prompt
- Tools como funciones Python tipadas
- Inyección de dependencias con `RunContext`
- Resultados validados y por qué importa en producción

---

## El Concepto

### Un agente con resultado tipado

Defines un `Agent` indicando el modelo y el **tipo de salida**. PydanticAI garantiza que `result.output` cumple ese tipo, reintentando si el LLM se desvía.

```python
from pydantic import BaseModel
from pydantic_ai import Agent

class Analisis(BaseModel):
    sentimiento: str          # "positivo" | "neutro" | "negativo"
    resumen: str
    temas: list[str]

agente = Agent(
    "anthropic:claude-sonnet-4-6",
    output_type=Analisis,
    system_prompt="Analiza reseñas de producto de forma objetiva y concisa.",
)

resultado = agente.run_sync("La batería dura poco pero la cámara es excelente.")
print(resultado.output.sentimiento)  # p. ej. "neutro"
print(resultado.output.temas)        # ["batería", "cámara"]
```

`resultado.output` es una instancia de `Analisis` **ya validada**: puedes usarla con seguridad de tipos en el resto de tu código.

### Tools: funciones Python que el agente puede llamar

Una tool es una función decorada con `@agente.tool`. PydanticAI genera su esquema a partir de los **type hints** y la **docstring**, y valida los argumentos que el LLM proponga.

```python
from pydantic_ai import Agent, RunContext

agente = Agent("anthropic:claude-sonnet-4-6")

@agente.tool_plain
def temperatura_actual(ciudad: str) -> str:
    """Devuelve la temperatura actual de una ciudad dada."""
    # llamada real a una API meteorológica...
    return f"En {ciudad} hay 22°C"

# El agente decide cuándo llamar a la tool; los argumentos llegan validados
resultado = agente.run_sync("¿Qué tiempo hace en Valencia?")
print(resultado.output)
```

### Inyección de dependencias con RunContext

Las tools del mundo real necesitan recursos: un cliente de base de datos, un usuario autenticado, una API key. PydanticAI los inyecta de forma **tipada** mediante `deps_type` y `RunContext`.

```python
from dataclasses import dataclass
from pydantic_ai import Agent, RunContext

@dataclass
class Deps:
    db: "ClienteBD"
    usuario_id: int

agente = Agent("anthropic:claude-sonnet-4-6", deps_type=Deps)

@agente.tool
def pedidos_del_usuario(ctx: RunContext[Deps]) -> list[str]:
    """Lista los pedidos del usuario actual."""
    return ctx.deps.db.buscar_pedidos(ctx.deps.usuario_id)  # acceso tipado a las deps

resultado = agente.run_sync(
    "¿Cuáles son mis últimos pedidos?",
    deps=Deps(db=mi_db, usuario_id=42),
)
```

> [!tip] Por qué tipar el agente cambia las cosas en producción
> En un prototipo da igual recibir un dict. En producción, que `result.output` sea un modelo Pydantic validado significa que los errores del LLM se detectan **en la frontera** (con reintento), no tres capas más abajo con un `KeyError`. El tipado convierte fallos silenciosos en fallos explícitos y recuperables.

### Encaje con el ecosistema

PydanticAI cubre lo mismo que harías con [[Desarrollo Profesional/Pydantic/Páginas/02 - Salida Estructurada y Tool Calling|Instructor]] + un bucle de agente manual, pero integrado: resultado tipado, tools, dependencias, streaming y observabilidad (se integra con [[Desarrollo Profesional/Langfuse/Langfuse|Langfuse]] y Logfire). Es una alternativa más "pythónica" y ligera a [[Desarrollo Profesional/LangChain/LangChain|LangChain]] cuando quieres control y tipos estrictos. Conecta con los conceptos de [[Desarrollo Profesional/IA/Páginas/04 - Agentes y MCP|Agentes y MCP]].

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - **PydanticAI** lleva el tipado de Pydantic a los agentes: un `Agent` con **resultado validado**, dependencias y tools.
> - **`output_type`** garantiza que `result.output` cumple tu modelo Pydantic, reintentando si el LLM se desvía.
> - Las **tools** son funciones Python con type hints y docstring; PydanticAI genera su esquema y valida los argumentos.
> - **`RunContext` + `deps_type`** inyectan recursos (BD, usuario, claves) de forma tipada en las tools.
> - En producción, el resultado tipado detecta errores del LLM **en la frontera**, no tres capas más abajo.

### Para llevar a la práctica
- [ ] Crea un agente con `output_type` Pydantic que analice un texto y devuelva un objeto validado
- [ ] Añade una `tool` con argumentos tipados y comprueba cómo el agente la invoca sola
- [ ] Inyecta una dependencia (un cliente simulado) con `RunContext` y úsala dentro de una tool

### Recursos
- 🌐 ai.pydantic.dev — documentación de PydanticAI
- 🌐 ai.pydantic.dev/tools — tools y RunContext
- 🌐 ai.pydantic.dev/dependencies — inyección de dependencias

---
`#pydantic` `#ia` `#agentes` `#pydantic-ai`
