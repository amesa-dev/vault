# 🎙️ LiveKit

[[Desarrollo Profesional/Inicio Profesional|⬅️ Volver a Desarrollo Profesional]]

> [!abstract] Sobre esta sección
> LiveKit es una plataforma open source de **comunicación en tiempo real** (audio, vídeo y datos) construida sobre WebRTC. Su uso clásico es videollamadas y streaming, pero su papel más relevante hoy es ser **la infraestructura de los agentes de voz con IA**: el framework **LiveKit Agents** orquesta el pipeline voz-a-voz (escuchar → transcribir → razonar con un LLM → responder con voz) en tiempo real y con baja latencia. Esta sección cubre los fundamentos de transporte y, sobre todo, cómo construir un agente de voz.

---

## 📚 Páginas de esta sección

1. [[Desarrollo Profesional/LiveKit/Páginas/01 - Fundamentos de Tiempo Real|01 — Fundamentos de Tiempo Real]] — WebRTC, Rooms, Participants, Tracks y el SFU
2. [[Desarrollo Profesional/LiveKit/Páginas/02 - Agentes de Voz con IA|02 — Agentes de Voz con IA]] — LiveKit Agents, pipeline STT-LLM-TTS, VAD y turn detection

---

## ¿Qué es LiveKit en una frase?

La tubería en tiempo real que transporta voz y vídeo con baja latencia entre usuarios y servidores — y, con LiveKit Agents, el esqueleto sobre el que un LLM escucha y habla en una conversación natural.

> [!tip] Dónde encaja en una app de IA
> El LLM y el RAG ([[Desarrollo Profesional/LangChain/LangChain|LangChain]], [[Desarrollo Profesional/IA/Páginas/04 - Agentes y MCP|Agentes]]) aportan el "cerebro"; LiveKit aporta los "oídos y la boca" en tiempo real. Para observar y depurar estos agentes, [[Desarrollo Profesional/Langfuse/Langfuse|Langfuse]].

---

## 🔗 Recursos Generales
- 🌐 docs.livekit.io — documentación oficial
- 🌐 docs.livekit.io/agents — framework LiveKit Agents
- 🌐 github.com/livekit/agents — ejemplos y código del framework

---
`#livekit` `#tiempo-real` `#voz` `#indice`
