# 🐦 Pipecat

[[Desarrollo Profesional/Inicio Profesional|⬅️ Volver a Desarrollo Profesional]]

> [!abstract] Sobre esta sección
> Pipecat es un framework open source (de Daily) para construir **agentes de voz y multimodales en tiempo real**, y la **alternativa principal a [[Desarrollo Profesional/LiveKit/LiveKit|LiveKit Agents]]**. Su filosofía es distinta: en lugar de partir de una plataforma de transporte, Pipecat es un **pipeline de frames** —piezas de audio, texto y eventos que fluyen por una cadena de procesadores— agnóstico del transporte (funciona sobre Daily, WebSockets, teléfono, etc.). Es Python puro, muy componible y popular para prototipar y desplegar bots de voz.

---

## 📚 Páginas de esta sección

1. [[Desarrollo Profesional/Pipecat/Páginas/01 - Pipeline de Frames|01 — Pipeline de Frames]] — frames, processors, transports y el modelo de flujo
2. [[Desarrollo Profesional/Pipecat/Páginas/02 - Bot de Voz y comparación con LiveKit|02 — Bot de Voz y comparación con LiveKit]] — un bot completo y cuándo elegir Pipecat vs LiveKit

---

## ¿Qué es Pipecat en una frase?

Un pipeline componible por el que fluyen *frames* (audio, texto, eventos) a través de procesadores intercambiables (STT, LLM, TTS) para construir agentes de voz en tiempo real, sin atarte a un transporte concreto.

> [!tip] Pipecat vs LiveKit, en breve
> Ambos resuelven el mismo problema (agentes de voz STT-LLM-TTS con gestión de turnos). **LiveKit** parte de una plataforma WebRTC robusta y escalable; **Pipecat** parte de un pipeline flexible y agnóstico del transporte. La comparación detallada está en la [[Desarrollo Profesional/Pipecat/Páginas/02 - Bot de Voz y comparación con LiveKit|página 02]]. Para observabilidad de ambos, [[Desarrollo Profesional/Langfuse/Langfuse|Langfuse]].

---

## 🔗 Recursos Generales
- 🌐 docs.pipecat.ai — documentación oficial
- 🌐 github.com/pipecat-ai/pipecat — código y ejemplos
- 🌐 docs.pipecat.ai/getting-started/overview — visión general

---
`#pipecat` `#voz` `#tiempo-real` `#indice`
