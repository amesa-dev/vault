# 🎙️ LiveKit 01 — Fundamentos de Tiempo Real

[[Desarrollo Profesional/LiveKit/LiveKit|⬅️ Volver a LiveKit]] | [[Desarrollo Profesional/LiveKit/Páginas/02 - Agentes de Voz con IA|02 →]]

> [!abstract] Introducción
> Antes de construir un agente de voz hay que entender el modelo de LiveKit, que hereda de **WebRTC**. La unidad central es la **Room** (sala): un espacio donde varios **Participants** (participantes) publican y se suscriben a **Tracks** (pistas de audio, vídeo o datos). Un servidor **SFU** reparte esas pistas de forma eficiente. Estos conceptos son los mismos tanto si conectas dos personas como si conectas a una persona con un agente de IA.

## ¿De qué vamos a hablar?

El modelo Room/Participant/Track, qué es y por qué importa el SFU, el flujo de conexión con tokens, y un servidor mínimo de tokens en Python.

### Conceptos que vamos a cubrir
- WebRTC en una frase y por qué un SFU
- Rooms, Participants y Tracks
- Tokens de acceso y conexión
- El modelo aplicado a un agente de IA

---

## El Concepto

### WebRTC y el SFU

**WebRTC** permite enviar audio/vídeo entre navegadores con muy baja latencia. El problema: conectar a todos con todos (malla) no escala. LiveKit usa un **SFU** (Selective Forwarding Unit): cada participante envía su pista **una vez** al servidor, y el SFU la **reenvía** a los demás. Escala a muchos participantes sin saturar la subida de cada uno.

```
Malla (no escala)            SFU (LiveKit)
  A ── B                       A ─┐
  │ ╳ │                        B ─┤──▶ [SFU] ──▶ reparte a cada uno
  C ── D                       C ─┘
(cada uno sube a todos)     (cada uno sube una vez)
```

### Rooms, Participants y Tracks

```
Room "soporte-123"
├── Participant: usuario        (local)
│   └── Track: micrófono (audio)
└── Participant: agente-ia      (remoto)
    └── Track: voz sintetizada (audio)
```

- **Room**: el espacio lógico de la sesión. Todo ocurre dentro de una sala.
- **Participant**: cada entidad conectada (una persona, un agente, un proceso de servidor).
- **Track**: un flujo de medios que un participante **publica**; los demás se **suscriben** a él.

### Tokens y conexión

Para entrar en una sala, el cliente necesita un **access token** (JWT) firmado en tu servidor con la API key/secret. El token codifica quién es y qué permisos tiene (a qué sala, si puede publicar, etc.). Nunca pongas el secret en el cliente.

```python
from livekit import api

def crear_token(sala: str, identidad: str) -> str:
    token = (
        api.AccessToken()  # usa LIVEKIT_API_KEY y LIVEKIT_API_SECRET del entorno
        .with_identity(identidad)
        .with_name("Usuario")
        .with_grants(api.VideoGrants(room_join=True, room=sala))
    )
    return token.to_jwt()

# El cliente (web/móvil) usa este JWT para conectarse a la URL del servidor LiveKit
```

```
Cliente ──pide token──▶ Tu backend ──firma JWT──▶ Cliente ──conecta con JWT──▶ LiveKit (sala)
```

### El modelo aplicado a un agente de IA

Aquí está la clave para la siguiente página: **un agente de voz es, técnicamente, un participante más**. Se une a la sala, se **suscribe** a la pista de audio del usuario (para oírle) y **publica** su propia pista de audio (para hablarle). Todo lo demás —STT, LLM, TTS— ocurre dentro de ese participante.

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - LiveKit se basa en **WebRTC** y usa un **SFU**: cada participante sube su pista una vez y el servidor la reparte, lo que escala mucho mejor que una malla.
> - El modelo es **Room → Participants → Tracks**: una sala con participantes que publican y se suscriben a pistas de audio/vídeo/datos.
> - La conexión requiere un **access token (JWT)** firmado en tu backend con la API key/secret; nunca expongas el secret en el cliente.
> - Un **agente de IA es un participante más**: se suscribe al audio del usuario y publica su voz.

### Para llevar a la práctica
- [ ] Levanta un servidor LiveKit (cloud o local con `livekit-server`) y genera un token con el SDK de Python
- [ ] Conecta dos clientes a una misma sala y observa la publicación/suscripción de pistas
- [ ] Dibuja el modelo Room/Participant/Track de un caso de uso tuyo (p. ej. atención al cliente)

### Recursos
- 🌐 docs.livekit.io/home/get-started/intro-to-livekit — introducción
- 🌐 docs.livekit.io/home/client/tracks — pistas (publicar/suscribir)
- 🌐 docs.livekit.io/home/server/generating-tokens — generación de tokens

---
`#livekit` `#webrtc` `#tiempo-real` `#fundamentos`
