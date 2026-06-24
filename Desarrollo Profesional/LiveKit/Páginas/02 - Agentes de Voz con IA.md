# 🎙️ LiveKit 02 — Agentes de Voz con IA

[[Desarrollo Profesional/LiveKit/LiveKit|⬅️ Volver a LiveKit]] | [[Desarrollo Profesional/LiveKit/Páginas/01 - Fundamentos de Tiempo Real|← 01]]

> [!abstract] Introducción
> Aquí está el motivo por el que LiveKit interesa en IA. El framework **LiveKit Agents** monta un agente de voz como un participante que escucha y habla en tiempo real. Por debajo orquesta un pipeline: **STT** (voz a texto) → **LLM** (razonamiento) → **TTS** (texto a voz), más dos piezas críticas para que la conversación sea natural: la **detección de actividad de voz** (VAD) y la **detección de turno** (saber cuándo el usuario terminó de hablar). Construir esto a mano es muy difícil; el framework lo resuelve.

## ¿De qué vamos a hablar?

La arquitectura de un agente de voz, el pipeline STT-LLM-TTS, los conceptos de VAD, turn detection e interrupciones, y un agente mínimo en Python con LiveKit Agents.

### Conceptos que vamos a cubrir
- El pipeline voz-a-voz: STT → LLM → TTS
- VAD, turn detection e interrupciones (barge-in)
- Un agente de voz mínimo con LiveKit Agents
- Tools y latencia

### La arquitectura

```
Usuario habla ─▶ [VAD detecta voz] ─▶ STT (texto) ─▶ LLM (razona) ─▶ TTS (voz) ─▶ Usuario oye
                      │                                    ▲
                      └── turn detection: ¿terminó de hablar? ┘
       (si el usuario habla mientras el agente habla → interrupción / barge-in)
```

---

## El Concepto

### El pipeline voz-a-voz

Un agente de voz encadena tres modelos. LiveKit Agents te deja elegir el proveedor de cada uno (son intercambiables):

- **STT** (Speech-to-Text): transcribe el audio del usuario. Ej.: Deepgram, AssemblyAI.
- **LLM**: genera la respuesta a partir de la transcripción. Ej.: Claude, GPT.
- **TTS** (Text-to-Speech): convierte la respuesta en voz. Ej.: ElevenLabs, Cartesia.

Existe también la vía **speech-to-speech** (un único modelo multimodal de voz, como los *realtime* de algunos proveedores) que reduce latencia a cambio de menos control sobre cada etapa.

### VAD, turn detection e interrupciones

Lo que separa un agente natural de uno torpe no son los modelos, sino la **gestión del turno**:

- **VAD** (Voice Activity Detection): detecta cuándo hay voz frente a silencio. Evita transcribir ruido.
- **Turn detection**: decide cuándo el usuario **ha terminado** de hablar para que el agente responda — sin cortarle ni dejar silencios incómodos. LiveKit usa modelos específicos para esto.
- **Interrupciones (barge-in)**: si el usuario empieza a hablar mientras el agente responde, el agente **se calla** y escucha. Imprescindible para que se sienta humano.

### Un agente de voz mínimo

LiveKit Agents reduce todo el pipeline a una `AgentSession` con los componentes elegidos. El agente corre como un *worker* que se une a las salas.

```python
from livekit.agents import Agent, AgentSession, JobContext, WorkerOptions, cli
from livekit.plugins import deepgram, anthropic, elevenlabs, silero

class Asistente(Agent):
    def __init__(self) -> None:
        super().__init__(instructions="Eres un asistente telefónico amable y conciso.")

async def entrypoint(ctx: JobContext):
    session = AgentSession(
        vad=silero.VAD.load(),                       # detección de voz
        stt=deepgram.STT(language="es"),             # voz -> texto
        llm=anthropic.LLM(model="claude-sonnet-4-6"),# razonamiento
        tts=elevenlabs.TTS(),                        # texto -> voz
    )
    await session.start(agent=Asistente(), room=ctx.room)

if __name__ == "__main__":
    cli.run_app(WorkerOptions(entrypoint_fnc=entrypoint))
```

Con eso, el agente se une a la sala, escucha al usuario, razona y responde por voz, gestionando turnos e interrupciones por defecto.

### Tools: que el agente actúe, no solo hable

Como en cualquier agente, puedes darle **herramientas** para consultar datos o ejecutar acciones (reservar, consultar un pedido). Se definen como funciones del agente con type hints —misma idea que en [[Desarrollo Profesional/LangChain/Páginas/03 - Agentes y LangGraph|LangChain]] y [[Desarrollo Profesional/Pydantic/Páginas/03 - PydanticAI y Agentes|PydanticAI]].

```python
from livekit.agents import function_tool

class Asistente(Agent):
    @function_tool
    async def estado_pedido(self, numero: str) -> str:
        """Consulta el estado de un pedido por su número."""
        return f"El pedido {numero} se entrega mañana."
```

> [!tip] La latencia es la métrica reina
> En voz, la latencia de extremo a extremo (fin del habla del usuario → primera palabra del agente) define la experiencia. Por debajo de ~800 ms se siente natural. Cada etapa suma: STT en streaming, un LLM rápido y un TTS de baja latencia. Mide cada tramo (puedes trazarlo con [[Desarrollo Profesional/Langfuse/Langfuse|Langfuse]]) y optimiza el cuello de botella.

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - **LiveKit Agents** monta un agente de voz como participante que escucha y habla en tiempo real.
> - El pipeline es **STT → LLM → TTS**, con proveedores intercambiables; la alternativa es **speech-to-speech** (menos latencia, menos control).
> - La naturalidad depende de la **gestión del turno**: **VAD**, **turn detection** e **interrupciones (barge-in)** — no solo de los modelos.
> - Una **`AgentSession`** con `vad/stt/llm/tts` basta para un agente funcional; añade **tools** (`@function_tool`) para que actúe.
> - La **latencia de extremo a extremo** es la métrica clave: busca ~800 ms o menos y optimiza el cuello de botella.

### Para llevar a la práctica
- [ ] Monta el agente de voz mínimo y habla con él en una sala de LiveKit
- [ ] Prueba a interrumpirle mientras habla y comprueba el barge-in
- [ ] Añade una `@function_tool` y pídele algo que le obligue a usarla

### Recursos
- 🌐 docs.livekit.io/agents — guía del framework de agentes
- 🌐 docs.livekit.io/agents/build/turns — turn detection e interrupciones
- 🌐 docs.livekit.io/agents/integrations — plugins de STT, LLM y TTS

---
`#livekit` `#voz` `#agentes` `#ia`
