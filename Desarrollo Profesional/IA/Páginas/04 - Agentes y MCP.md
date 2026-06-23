# 🤖 IA 04 — Agentes y MCP

[[Desarrollo Profesional/IA/IA|⬅️ Volver a IA]] | [[Desarrollo Profesional/IA/Páginas/03 - RAG|← 03]]

> [!abstract] Introducción
> Los agentes LLM son sistemas donde el modelo toma decisiones sobre qué acciones ejecutar para alcanzar un objetivo — no solo generan texto, sino que actúan. El Model Context Protocol (MCP) es el estándar abierto de Anthropic que define cómo los LLMs pueden acceder a herramientas, recursos y contexto de forma estandarizada. Juntos, permiten construir sistemas de IA que interactúan con el mundo real.

## ¿De qué vamos a hablar?

El patrón de agentes LLM con tool use, el Model Context Protocol (MCP) y cómo construir sistemas de agentes con Python.

### Conceptos que vamos a cubrir
- Tool use (function calling): el mecanismo base de los agentes
- El loop de agente: reason → act → observe
- Model Context Protocol (MCP): estándar de herramientas
- Patrones de orquestación: agentes en secuencia y en paralelo

---

## El Concepto

### Tool Use — El Mecanismo Base

El tool use (también llamado function calling) permite al LLM llamar a funciones externas. El modelo no ejecuta el código — elige qué función llamar y con qué argumentos, y el host ejecuta la función y devuelve el resultado:

```python
import anthropic
import json

client = anthropic.Anthropic()

# Definir herramientas disponibles para el modelo
herramientas = [
    {
        "name": "buscar_pedido",
        "description": "Busca información de un pedido por su ID",
        "input_schema": {
            "type": "object",
            "properties": {
                "pedido_id": {
                    "type": "string",
                    "description": "ID único del pedido"
                }
            },
            "required": ["pedido_id"]
        }
    },
    {
        "name": "cancelar_pedido",
        "description": "Cancela un pedido si está en estado pendiente",
        "input_schema": {
            "type": "object",
            "properties": {
                "pedido_id": {"type": "string"},
                "motivo": {"type": "string", "description": "Razón de la cancelación"}
            },
            "required": ["pedido_id", "motivo"]
        }
    }
]

# Implementaciones reales de las herramientas
def buscar_pedido(pedido_id: str) -> dict:
    # En producción, esto consultaría la BD
    return {"id": pedido_id, "estado": "pendiente", "total": 89.99, "cliente": "Ana García"}

def cancelar_pedido(pedido_id: str, motivo: str) -> dict:
    return {"ok": True, "mensaje": f"Pedido {pedido_id} cancelado. Motivo: {motivo}"}

herramienta_fn = {
    "buscar_pedido": buscar_pedido,
    "cancelar_pedido": cancelar_pedido
}
```

### El Loop de Agente

```python
def ejecutar_agente(pregunta: str, max_iteraciones: int = 10) -> str:
    mensajes = [{"role": "user", "content": pregunta}]

    for iteracion in range(max_iteraciones):
        respuesta = client.messages.create(
            model="claude-sonnet-4-6",
            max_tokens=4096,
            system="Eres un agente de soporte al cliente. Usa las herramientas disponibles para ayudar.",
            tools=herramientas,
            messages=mensajes
        )

        # Si el modelo quiere llamar a herramientas
        if respuesta.stop_reason == "tool_use":
            # Añadir la respuesta del asistente al historial
            mensajes.append({"role": "assistant", "content": respuesta.content})

            # Ejecutar cada herramienta solicitada
            resultados_herramientas = []
            for bloque in respuesta.content:
                if bloque.type == "tool_use":
                    print(f"🔧 Llamando a: {bloque.name}({bloque.input})")
                    fn = herramienta_fn[bloque.name]
                    resultado = fn(**bloque.input)
                    print(f"📦 Resultado: {resultado}")

                    resultados_herramientas.append({
                        "type": "tool_result",
                        "tool_use_id": bloque.id,
                        "content": json.dumps(resultado)
                    })

            # Añadir resultados al historial
            mensajes.append({"role": "user", "content": resultados_herramientas})

        # Si el modelo ha terminado
        elif respuesta.stop_reason == "end_turn":
            respuesta_final = next(
                (b.text for b in respuesta.content if hasattr(b, "text")), ""
            )
            return respuesta_final

    return "Límite de iteraciones alcanzado"

# Uso
respuesta = ejecutar_agente("Necesito cancelar mi pedido PED-123 porque me equivoqué")
print(respuesta)
# El agente: busca el pedido → ve que está pendiente → lo cancela → informa al cliente
```

### Model Context Protocol (MCP)

MCP es el estándar abierto de Anthropic que estandariza cómo los LLMs acceden a herramientas y contexto. Un servidor MCP expone recursos, herramientas y prompts de forma estandarizada:

```python
# Crear un servidor MCP simple
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("mi-servidor-soporte")

@mcp.tool()
def buscar_pedido(pedido_id: str) -> dict:
    """Busca un pedido por su ID y retorna su estado actual"""
    # Implementación real
    return {"id": pedido_id, "estado": "enviado", "tracking": "ES123456789"}

@mcp.tool()
def listar_pedidos_cliente(cliente_email: str) -> list[dict]:
    """Lista todos los pedidos de un cliente dado su email"""
    return [
        {"id": "PED-001", "estado": "entregado", "total": 89.99},
        {"id": "PED-002", "estado": "enviado", "total": 145.50},
    ]

@mcp.resource("pedido://{pedido_id}")
def recurso_pedido(pedido_id: str) -> str:
    """Expone los detalles de un pedido como recurso"""
    detalles = buscar_pedido(pedido_id)
    return f"Pedido {pedido_id}: {detalles}"

@mcp.prompt()
def prompt_soporte_cliente(nombre_cliente: str) -> str:
    """Prompt base para el agente de soporte"""
    return f"""Eres un agente de soporte amigable para {nombre_cliente}.
Usa las herramientas disponibles para buscar información de pedidos.
Sé conciso y directo."""

if __name__ == "__main__":
    mcp.run()
```

```bash
# Instalar y correr
pip install "mcp[cli]" fastmcp

# Ejecutar el servidor
python mi_servidor.py

# Probar con el cliente MCP
mcp dev mi_servidor.py
```

### Patrones de Orquestación

**Agentes en cadena (secuencial)** — el output de un agente es el input del siguiente:

```python
async def pipeline_analisis(texto: str) -> dict:
    # Paso 1: Clasificar intención
    clasificacion = await agente_clasificador.ejecutar(texto)

    # Paso 2: Extraer entidades según la intención
    entidades = await agente_extractor.ejecutar(texto, tipo=clasificacion["tipo"])

    # Paso 3: Generar respuesta
    respuesta = await agente_respondedor.ejecutar(
        texto=texto,
        entidades=entidades,
        tipo=clasificacion["tipo"]
    )

    return respuesta

# Agentes especializados en paralelo
import asyncio

async def analisis_completo(documento: str) -> dict:
    # Ejecutar múltiples análisis en paralelo
    resumen, sentimiento, entidades_clave = await asyncio.gather(
        agente_resumen.ejecutar(documento),
        agente_sentimiento.ejecutar(documento),
        agente_entidades.ejecutar(documento)
    )

    # Sintetizar resultados
    return {
        "resumen": resumen,
        "sentimiento": sentimiento,
        "entidades": entidades_clave
    }
```

**Patrón ReAct (Reason + Act)** — el modelo razona explícitamente antes de actuar:

```python
system_react = """Usa el siguiente formato para resolver tareas:

Pensamiento: Razona sobre qué debes hacer a continuación
Acción: La herramienta que vas a usar y con qué parámetros
Observación: El resultado de la acción
... (repite Pensamiento/Acción/Observación hasta tener suficiente información)
Respuesta Final: Tu respuesta al usuario"""
```

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - El **tool use** da a los LLMs la capacidad de actuar — el modelo elige herramientas y argumentos, el host las ejecuta. El loop reason→act→observe es el patrón base.
> - El **Model Context Protocol (MCP)** estandariza cómo los LLMs acceden a herramientas — permite que distintos modelos y aplicaciones usen los mismos servidores de herramientas.
> - Los agentes son más frágiles que los pipelines fijos — añaden error handling, límites de iteraciones y monitorización robusta.
> - Para tareas paralelas independientes, ejecuta agentes simultáneamente con `asyncio.gather`. Para tareas dependientes, usa cadenas secuenciales.

### Recursos
- 🌐 modelcontextprotocol.io — especificación oficial de MCP
- 🌐 docs.anthropic.com/en/docs/build-with-claude/tool-use — documentación de tool use
- 📄 *ReAct: Synergizing Reasoning and Acting in Language Models* — Yao et al., 2022
- 🌐 github.com/anthropics/anthropic-sdk-python — SDK de Python de Anthropic

---
`#ia` `#agentes` `#mcp` `#tool-use` `#llm`
