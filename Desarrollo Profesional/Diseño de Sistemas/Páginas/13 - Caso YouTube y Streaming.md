# 🏛️ Caso de Estudio — YouTube / Streaming de Vídeo

[[Desarrollo Profesional/Diseño de Sistemas/Diseño de Sistemas|⬅️ Volver a Diseño de Sistemas]] | [[Desarrollo Profesional/Diseño de Sistemas/Páginas/12 - Caso Web Crawler|← 12]] | [[Desarrollo Profesional/Diseño de Sistemas/Páginas/14 - Caso Google Drive|14 →]]

> [!abstract] Introducción
> Diseñar YouTube (o Netflix, o Twitch) junta varios subproblemas grandes: subir vídeos pesados, **transcodificarlos** a muchos formatos y calidades, almacenar petabytes, y —lo más difícil— entregar streaming fluido a millones de personas en todo el mundo con latencia mínima. La clave que lo hace posible es una combinación de **CDN masiva** y **streaming adaptativo**. Es el caso que mejor enseña el papel de las CDN y el procesamiento asíncrono pesado.

## ¿De qué vamos a hablar?

El diseño de una plataforma de vídeo: el pipeline de subida y transcodificación, el almacenamiento y la entrega por CDN con streaming adaptativo.

### Conceptos que vamos a cubrir
- Requisitos y la escala del vídeo
- El pipeline de subida y transcodificación
- Almacenamiento de vídeo y CDN
- Streaming adaptativo (ABR) y protocolos
- Metadatos, vistas y escalado de lectura

---

## El Diseño

### 1. Requisitos y Escala

**Funcionales**: subir vídeos, verlos en streaming (sin descargar entero), en múltiples dispositivos y calidades.

**No funcionales**: **reproducción fluida** y de baja latencia globalmente, escala masiva (miles de millones de visualizaciones/día, exabytes de almacenamiento), tolerancia a fallos, y coste contenido (el vídeo es caro de mover).

La escala del vídeo es de otro orden: un solo vídeo de 1 hora en varias calidades son varios GB; multiplica por millones de vídeos. **El ancho de banda es el recurso dominante y el coste principal.**

### 2. El Pipeline de Subida y Transcodificación

Subir un vídeo no es un simple `INSERT`. Es un **pipeline asíncrono** pesado:

```
[Usuario sube vídeo original] → [Almacenamiento temporal/original]
        → publica evento en [Cola] →
[Workers de transcodificación] (procesamiento masivo en paralelo):
   - transcodifica a múltiples resoluciones (240p, 480p, 720p, 1080p, 4K)
   - a múltiples codecs/formatos (H.264, VP9, AV1...)
   - segmenta en trozos pequeños (chunks de ~segundos) para streaming adaptativo
   - genera miniaturas
        → guarda todas las versiones en [Almacenamiento de vídeo]
        → distribuye a la [CDN]
        → marca el vídeo como "listo" y notifica al usuario
```

- **Transcodificación**: el vídeo original se convierte a muchas combinaciones de resolución/codec/bitrate, porque cada dispositivo y cada ancho de banda necesitan una distinta. Es **carísimo en CPU/GPU** → se hace con una flota de workers consumiendo una **cola** ([[Desarrollo Profesional/Sistemas Distribuidos/Páginas/02 - Mensajería y Brokers|Mensajería]]), totalmente fuera del camino de la subida. El usuario sube y recibe "tu vídeo se está procesando".
- Se modela como un **DAG de tareas** (segmentar → transcodificar cada segmento → empaquetar) que se pueden paralelizar masivamente: distintos segmentos del mismo vídeo se transcodifican en máquinas distintas a la vez.

### 3. Almacenamiento y CDN

- **Almacenamiento**: el vídeo (original + todas las versiones transcodificadas) va a **almacenamiento de objetos** (tipo GCS/S3), barato y escalable a exabytes. Los vídeos populares y los recientes se replican más; los muy antiguos y poco vistos pueden ir a almacenamiento frío más barato (cold storage).
- **CDN — la pieza crítica**: el vídeo se sirve desde la **CDN** ([[Desarrollo Profesional/Diseño de Sistemas/Páginas/04 - Caché CDN Balanceo y Colas|página 04]]), no desde tu datacenter. Los segmentos de vídeo se cachean en edges cercanos al usuario, porque mover vídeo intercontinentalmente es lento y caro. **El 90%+ del tráfico de visualización lo sirve la CDN.** Aquí se aplica la regla 80/20 con fuerza: un puñado de vídeos virales concentran la mayoría de las vistas → se cachean agresivamente en todos los edges; la *long tail* de vídeos poco vistos se sirve del origen o de edges regionales.

### 4. Streaming Adaptativo (ABR)

La razón de transcodificar a tantas calidades y segmentar en chunks: el **Adaptive Bitrate Streaming** (ABR, con protocolos como **HLS** o **DASH**). El vídeo se parte en segmentos cortos (2-10s), cada uno disponible en varias calidades. El reproductor del cliente **mide el ancho de banda en tiempo real** y pide cada segmento en la calidad que la red aguante en ese momento:

```
Red rápida → pide segmentos en 1080p.
Red se degrada → el siguiente segmento lo pide en 480p (sin cortar la reproducción).
Red mejora → vuelve a subir calidad.
```

Por eso un vídeo "se ve borroso un momento" y luego se aclara en vez de pararse a cargar: el cliente está adaptando la calidad segmento a segmento. Esto da reproducción fluida en redes variables, que es justo el requisito difícil.

### 5. Metadatos, Vistas y Lectura

- **Metadatos** (título, descripción, autor, relaciones): en una BD aparte del vídeo en sí. Lectura-intensiva → caché y réplicas.
- **Contador de vistas**: un contador muy escrito y muy leído; se usa la idea de contadores distribuidos/agregación aproximada (no necesitas el número exacto al instante).
- **Recomendaciones, búsqueda, comentarios**: subsistemas propios, cada uno un mini-caso (la búsqueda y el feed se parecen a casos anteriores).

### 6. Cuellos de Botella y Mejoras

- **Coste de ancho de banda**: dominado por la CDN; optimizado con codecs más eficientes (AV1 ahorra bits) y cacheo inteligente.
- **Coste de transcodificación**: enorme; se optimiza priorizando (no transcodificar a 4K un vídeo que nadie verá) y con hardware especializado.
- **Subidas grandes y poco fiables**: subida por partes (multipart/resumable) para reanudar si se corta.
- **Vídeos virales**: la CDN los absorbe; el reto es predecir y precachear lo que se va a hacer viral.

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - El vídeo es de **escala extrema** y el **ancho de banda es el recurso y coste dominante**. La subida no es un insert: es un **pipeline asíncrono** pesado.
> - **Transcodificación**: el original se convierte a muchas resoluciones/codecs/bitrates y se **segmenta en chunks**. Es carísimo (CPU/GPU) → flota de workers sobre una **cola**, fuera del camino de subida, modelado como un DAG paralelizable.
> - El **almacenamiento** es de objetos (exabytes; frío para lo viejo). La **CDN es la pieza crítica**: sirve el 90%+ del tráfico cacheando segmentos cerca del usuario (mover vídeo lejos es caro). 80/20: virales cacheados en todos los edges.
> - **Streaming adaptativo (ABR: HLS/DASH)**: el cliente mide su ancho de banda y pide cada segmento en la calidad que la red aguanta → reproducción fluida en redes variables (por eso se ve borroso y luego nítido, sin pararse).
> - **Metadatos** y **contador de vistas** en almacenes aparte, lectura-intensivos (caché/réplicas, conteo aproximado). Subidas resumibles por partes.

### Para llevar a la práctica
- [ ] Dibuja el pipeline de subida→transcodificación→CDN y marca qué es síncrono y qué asíncrono.
- [ ] Explica por qué se transcodifica a tantas versiones y cómo lo aprovecha el ABR.
- [ ] Razona el papel de la CDN: ¿qué porcentaje del tráfico evita tu origen y por qué importa tanto?
- [ ] Diseña la estrategia de almacenamiento caliente/frío según popularidad del vídeo.

### Recursos
- 📖 *System Design Interview, Vol. 1* — Alex Xu (capítulo 14: Design YouTube)
- 🌐 developer.apple.com/streaming — HLS; dashif.org — DASH
- 📄 "Netflix's Encoding Pipeline" y blogs de ingeniería de Netflix/YouTube
- 📄 Conexión con [[Desarrollo Profesional/Diseño de Sistemas/Páginas/04 - Caché CDN Balanceo y Colas|CDN]] y [[Desarrollo Profesional/Sistemas Distribuidos/Páginas/02 - Mensajería y Brokers|colas]]

---
`#diseño-de-sistemas` `#video` `#streaming` `#cdn` `#caso-estudio`
