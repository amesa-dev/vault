# 🐦 Pipecat 01 — Pipeline de Frames

[[Desarrollo Profesional/Pipecat/Pipecat|⬅️ Volver a Pipecat]] | [[Desarrollo Profesional/Pipecat/Páginas/02 - Bot de Voz y comparación con LiveKit|02 →]]

> [!abstract] Introducción
> La abstracción central de Pipecat no es la "sala" sino el **pipeline**: una secuencia de **procesadores** por la que fluyen **frames**. Un frame es una unidad de datos —un trozo de audio, un fragmento de texto, una transcripción, un evento de "el usuario empezó a hablar"—. Cada procesador recibe frames, hace algo y emite frames hacia el siguiente. Entender este modelo de flujo bidireccional es entender Pipecat: todo lo demás (STT, LLM, TTS) son procesadores enchufados a esa cinta transportadora.

## ¿De qué vamos a hablar?

Qué son los frames y los procesadores, cómo se ensambla un pipeline, el papel del transport como entrada/salida, y la dirección del flujo.

### Conceptos que vamos a cubrir
- Frames: las unidades que fluyen
- Processors: las piezas que transforman
- Transports: la entrada y salida del mundo real
- El pipeline y el flujo bidireccional

---

## El Concepto

### Frames y processors

Un **frame** es un dato tipado que viaja por el pipeline: `AudioRawFrame`, `TranscriptionFrame`, `TextFrame`, `LLMMessagesFrame`, frames de control... Un **processor** consume frames de un tipo y produce otros.

```
[Audio del usuario] ─AudioFrame─▶ [STT] ─TranscriptionFrame─▶ [LLM] ─TextFrame─▶ [TTS] ─AudioFrame─▶ [Audio al usuario]
```

Cada caja es un processor; cada flecha, frames de un tipo concreto fluyendo hacia el siguiente.

### El transport: entrada y salida

El **transport** conecta el pipeline con el mundo real: captura el audio entrante (entrada) y reproduce el saliente (salida). Es la pieza que Pipecat mantiene **agnóstica**: el mismo pipeline funciona sobre Daily (WebRTC), un WebSocket o telefonía, cambiando solo el transport.

```python
from pipecat.pipeline.pipeline import Pipeline

pipeline = Pipeline([
    transport.input(),    # audio entrante del usuario  ──▶ frames
    stt,                  # audio  ──▶ transcripción
    llm,                  # mensajes ──▶ respuesta
    tts,                  # texto  ──▶ audio
    transport.output(),   # frames ──▶ audio saliente al usuario
])
```

### El flujo bidireccional

Los frames no van solo "hacia adelante" (downstream). También hay frames **upstream** (hacia atrás), usados sobre todo para control: por ejemplo, cuando el usuario interrumpe, un frame upstream avisa para **cancelar** la generación de voz en curso. Esta bidireccionalidad es lo que permite gestionar interrupciones limpias.

```
downstream  ───────────────▶  (datos: audio, texto, respuesta)
◀───────────────  upstream    (control: interrupción, fin de turno, errores)
```

### Agregadores de contexto

Para que el LLM mantenga la conversación, Pipecat usa **agregadores de contexto** que acumulan el historial (mensajes del usuario y del asistente) y lo inyectan en cada turno. Así el procesador del LLM recibe siempre la conversación completa, no solo la última frase.

> [!tip] Por qué el modelo de frames es potente
> Como todo es un frame que fluye por procesadores intercambiables, añadir capacidades es enchufar un procesador más: un filtro que detecte palabras prohibidas, un logger, un procesador que dispare una animación al hablar. No tocas el resto del pipeline. Es composición pura, la misma idea que el `|` de [[Desarrollo Profesional/LangChain/Páginas/01 - Fundamentos y LCEL|LCEL en LangChain]] pero para streams en tiempo real.

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - La abstracción central de Pipecat es el **pipeline**: procesadores en secuencia por los que fluyen **frames**.
> - Un **frame** es un dato tipado (audio, transcripción, texto, eventos); un **processor** consume unos y emite otros.
> - El **transport** conecta con el mundo real (entrada/salida de audio) y es **agnóstico**: Daily, WebSocket o teléfono con el mismo pipeline.
> - El flujo es **bidireccional**: downstream lleva datos, upstream lleva control (interrupciones, fin de turno).
> - Los **agregadores de contexto** mantienen el historial para que el LLM tenga memoria de la conversación.

### Para llevar a la práctica
- [ ] Dibuja el pipeline STT→LLM→TTS identificando qué tipo de frame fluye en cada flecha
- [ ] Instala Pipecat y arranca un ejemplo oficial para ver el pipeline en marcha
- [ ] Identifica dónde insertarías un procesador propio (p. ej. un logger de transcripciones)

### Recursos
- 🌐 docs.pipecat.ai/getting-started/core-concepts — frames y pipeline
- 🌐 docs.pipecat.ai/server/base-classes/transport — transports
- 🌐 github.com/pipecat-ai/pipecat/tree/main/examples — ejemplos

---
`#pipecat` `#frames` `#pipeline` `#voz`
