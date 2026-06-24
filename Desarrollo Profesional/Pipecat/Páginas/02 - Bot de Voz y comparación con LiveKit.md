# 🐦 Pipecat 02 — Bot de Voz y comparación con LiveKit

[[Desarrollo Profesional/Pipecat/Pipecat|⬅️ Volver a Pipecat]] | [[Desarrollo Profesional/Pipecat/Páginas/01 - Pipeline de Frames|← 01]]

> [!abstract] Introducción
> Con el modelo de pipeline claro, montar un bot de voz con Pipecat es ensamblar procesadores y arrancar la tarea. En esta página vemos un bot completo y, lo más útil, cuándo conviene Pipecat y cuándo [[Desarrollo Profesional/LiveKit/LiveKit|LiveKit]]: dos frameworks que resuelven el mismo problema con filosofías distintas. No hay un ganador absoluto; hay un encaje según tus prioridades de transporte, control y escala.

## ¿De qué vamos a hablar?

Un bot de voz mínimo con Pipecat, las piezas que lo componen, y una comparación honesta Pipecat vs LiveKit con criterios para decidir.

### Conceptos que vamos a cubrir
- Un bot de voz completo con Pipecat
- `PipelineTask` y `PipelineRunner`
- Comparación Pipecat vs LiveKit
- Cuándo elegir cada uno

---

## El Concepto

### Un bot de voz mínimo

El bot ensambla transport + STT + agregador de contexto + LLM + TTS, y lo ejecuta como una tarea. Cada servicio es intercambiable por el de otro proveedor.

```python
from pipecat.pipeline.pipeline import Pipeline
from pipecat.pipeline.task import PipelineTask
from pipecat.pipeline.runner import PipelineRunner
from pipecat.services.deepgram.stt import DeepgramSTTService
from pipecat.services.anthropic.llm import AnthropicLLMService
from pipecat.services.elevenlabs.tts import ElevenLabsTTSService

async def main(transport):
    stt = DeepgramSTTService(api_key=..., language="es")
    llm = AnthropicLLMService(api_key=..., model="claude-sonnet-4-6")
    tts = ElevenLabsTTSService(api_key=..., voice_id=...)

    # agregador que mantiene el historial de la conversación
    contexto = llm.create_context_aggregator(
        system="Eres un asistente telefónico amable y conciso."
    )

    pipeline = Pipeline([
        transport.input(),
        stt,
        contexto.user(),     # añade lo que dijo el usuario al historial
        llm,
        tts,
        transport.output(),
        contexto.assistant(),# añade la respuesta del asistente al historial
    ])

    task = PipelineTask(pipeline)
    await PipelineRunner().run(task)
```

`PipelineTask` representa la ejecución del pipeline y `PipelineRunner` la corre, gestionando el ciclo de vida, las interrupciones y el cierre. El bot escucha, transcribe, razona y responde por voz, manteniendo el contexto.

### Comparación Pipecat vs LiveKit

| Criterio | 🐦 Pipecat | 🎙️ LiveKit |
|---|---|---|
| **Filosofía** | Pipeline de frames componible | Plataforma WebRTC + framework de agentes |
| **Transporte** | Agnóstico (Daily, WS, teléfono…) | WebRTC propio, muy robusto |
| **Escalado / infra** | Tú lo montas (o Daily) | SFU y plataforma lista para escalar |
| **Control del flujo** | Muy fino (insertas procesadores) | Alto nivel, más "baterías incluidas" |
| **Curva** | Entiendes el pipeline a bajo nivel | Más rápido para el caso estándar |
| **Multimodal/vídeo** | Sí, mismo modelo de frames | Sí, fuerte en vídeo por su base WebRTC |
| **Ecosistema** | Daily, creciente | Amplio (clientes web/móvil, SDKs) |

### Cuándo elegir cada uno

- **Elige Pipecat** si quieres **control fino** del pipeline, insertar lógica propia entre etapas, ser agnóstico del transporte, o prototipar rápido en Python sin casarte con una plataforma.
- **Elige LiveKit** si necesitas una **plataforma de tiempo real robusta y escalable** de serie (SFU, clientes web/móvil maduros, telefonía gestionada), o ya usas LiveKit para vídeo/voz entre humanos y quieres añadir un agente.

> [!tip] No es una decisión irreversible
> Como ambos comparten el mismo patrón conceptual (STT-LLM-TTS + gestión de turnos), migrar de uno a otro es sobre todo reescribir el "pegamento", no la lógica del agente ni los prompts. Empieza por el que te dé velocidad hoy y reevalúa cuando la escala o el transporte lo exijan. En los dos casos, instrumenta con [[Desarrollo Profesional/Langfuse/Langfuse|Langfuse]] desde el principio.

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - Un bot de Pipecat ensambla **transport + STT + agregador de contexto + LLM + TTS** y lo corre con **`PipelineTask`** + **`PipelineRunner`**.
> - Los servicios son **intercambiables** por proveedor; el agregador de contexto mantiene el historial de la conversación.
> - **Pipecat** = pipeline de frames componible y agnóstico del transporte; **LiveKit** = plataforma WebRTC robusta y escalable con framework de agentes.
> - **Elige Pipecat** por control fino y flexibilidad de transporte; **elige LiveKit** por plataforma lista para escalar y madurez de clientes.
> - La migración entre ambos afecta al "pegamento", no a la lógica del agente: no es una decisión irreversible.

### Para llevar a la práctica
- [ ] Monta el bot de voz mínimo de Pipecat y conversa con él
- [ ] Inserta un procesador propio entre STT y LLM (p. ej. para registrar transcripciones)
- [ ] Escribe tu propia tabla de decisión Pipecat vs LiveKit para un proyecto concreto tuyo

### Recursos
- 🌐 docs.pipecat.ai/getting-started/quickstart — quickstart de un bot
- 🌐 docs.pipecat.ai/server/services — servicios STT/LLM/TTS disponibles
- 🌐 docs.livekit.io/agents — el equivalente en LiveKit para comparar

---
`#pipecat` `#voz` `#agentes` `#livekit`
