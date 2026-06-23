# 🤖 IA 01 — LLMs y Fundamentos

[[Desarrollo Profesional/IA/IA|⬅️ Volver a IA]] | [[Desarrollo Profesional/IA/Páginas/02 - Prompt Engineering|02 →]]

> [!abstract] Introducción
> Los LLMs (Large Language Models) son el corazón de la IA generativa. Para usarlos eficazmente como ingeniero — no solo llamar a una API — necesitas entender los conceptos que determinan su comportamiento: qué son los tokens, por qué la temperatura importa, qué limita la context window y cómo los embeddings convierten texto en vectores que las máquinas pueden comparar.

## ¿De qué vamos a hablar?

Los fundamentos conceptuales de los LLMs que todo ingeniero que trabaja con IA generativa necesita entender, sin entrar en las matemáticas del transformer pero con la profundidad suficiente para tomar decisiones técnicas.

### Conceptos que vamos a cubrir
- La arquitectura transformer en términos conceptuales
- Tokens: qué son y por qué importan
- Context window: límites y estrategias
- Temperature y top-p: controlar la aleatoriedad
- Embeddings: el puente entre texto y matemáticas

---

## El Concepto

### La Arquitectura Transformer — Intuición

El transformer (Vaswani et al., 2017 — "Attention Is All You Need") es la arquitectura que subyace a todos los LLMs modernos: GPT, Claude, Llama, Gemini. La intuición central es el **mecanismo de atención**:

Para predecir la siguiente palabra en "El gato persiguió al ___", el modelo no lee secuencialmente — presta atención a todas las palabras anteriores simultáneamente, asignando un peso a cuánto cada palabra anterior es relevante para predecir la siguiente. "gato" y "persiguió" reciben mucha atención; "El" y "al" menos.

Los LLMs son modelos **autorregresivos**: predicen un token a la vez, y cada token predicho se añade al contexto para predecir el siguiente. No "razonan" — predicen la continuación más probable de la secuencia de tokens vista hasta ahora.

```
Prompt: "La capital de Francia es"
   ↓
Modelo predice: "París" (token con probabilidad más alta)
   ↓
Nuevo contexto: "La capital de Francia es París"
   ↓
Modelo predice: "." o continúa según el contexto...
```

### Tokens — La Unidad de Procesamiento

Un token no es una palabra — es un fragmento de texto determinado por el tokenizador del modelo. La mayoría usa tokenizadores basados en BPE (Byte Pair Encoding):

```
"Tokenización" → ["Token", "ización"]  (2 tokens)
"hello"        → ["hello"]             (1 token)
"un-common"    → ["un", "-", "common"] (3 tokens)
" Python"      → [" Python"]           (el espacio es parte del token)
"2024"         → ["2024"]              (1 token — número común)
"gpt4"         → ["g", "pt", "4"]     (3 tokens — menos común)
```

**Por qué importan los tokens:**
- **Coste**: los LLMs de API se cobran por token (prompt tokens + completion tokens)
- **Context window**: el límite está en tokens, no en palabras
- **Latencia**: más tokens de salida = más tiempo de respuesta
- **Estimación rápida**: ~1 token ≈ 0.75 palabras en inglés; en español ~0.6-0.7

```python
import tiktoken  # librería de OpenAI para contar tokens

enc = tiktoken.encoding_for_model("gpt-4")
tokens = enc.encode("Hola mundo, esto es un test")
print(f"Tokens: {len(tokens)}")  # número de tokens
print(f"Token IDs: {tokens}")    # representación numérica

# Claude usa su propio tokenizador, pero la ratio es similar
```

### Context Window — El Límite de Memoria

La context window es la cantidad máxima de tokens que el modelo puede "ver" al mismo tiempo — incluye el prompt, el historial de conversación y la respuesta que está generando.

**Contextos típicos (2024-2025):**
- GPT-4o: 128k tokens (~96k palabras)
- Claude 3.5 Sonnet: 200k tokens (~150k palabras)
- Llama 3.1: 128k tokens
- Gemini 1.5 Pro: 1M tokens

**El problema de la context window:**
- El modelo no "recuerda" nada fuera de la context window
- El rendimiento puede degradarse en contextos muy largos ("lost in the middle")
- Más contexto = más cómputo cuadrático (aunque con optimizaciones)

### Temperature y Top-p — Controlar la Aleatoriedad

La **temperatura** controla cuán aleatoria es la selección de tokens:

```python
import anthropic

client = anthropic.Anthropic()

# Temperature 0 — determinista, siempre el token más probable
# Usar para: código, extracción de datos, clasificación
respuesta = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    temperature=0,
    messages=[{"role": "user", "content": "¿Cuál es la capital de España?"}]
)

# Temperature 0.7 — balance entre coherencia y creatividad
# Usar para: texto general, resúmenes, Q&A
respuesta = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    temperature=0.7,
    messages=[{"role": "user", "content": "Escribe una introducción para mi artículo"}]
)

# Temperature 1.0+ — alta creatividad, menos coherencia
# Usar para: brainstorming, contenido creativo, variedad de opciones
```

**Top-p (nucleus sampling)**: en lugar de considerar todos los tokens, solo considera los más probables cuya probabilidad acumulada llega a `p`. Con `top_p=0.9`, el modelo solo elige entre los tokens que juntos suman el 90% de probabilidad — los más improbables se excluyen. Complementa a la temperatura.

### Embeddings — Texto como Vectores

Un embedding es una representación densa de un texto como un vector de números reales (típicamente 768, 1536 o 3072 dimensiones). Textos semánticamente similares tienen vectores cercanos en el espacio vectorial:

```python
import anthropic
import numpy as np

client = anthropic.Anthropic()

def embedding(texto: str) -> list[float]:
    response = client.messages.create(
        model="claude-haiku-4-5-20251001",
        max_tokens=10,
        messages=[{"role": "user", "content": texto}]
    )
    # Para embeddings reales, usar la API de embeddings dedicada (OpenAI, Cohere, Voyage)
    pass

# Con la API de embeddings de OpenAI (ejemplo)
from openai import OpenAI

client_oai = OpenAI()

def obtener_embedding(texto: str) -> list[float]:
    response = client_oai.embeddings.create(
        input=texto,
        model="text-embedding-3-small"
    )
    return response.data[0].embedding

# Similaridad coseno — qué tan parecidos son dos textos
def similitud_coseno(a: list[float], b: list[float]) -> float:
    a_arr = np.array(a)
    b_arr = np.array(b)
    return np.dot(a_arr, b_arr) / (np.linalg.norm(a_arr) * np.linalg.norm(b_arr))

emb1 = obtener_embedding("El gato duerme en el sofá")
emb2 = obtener_embedding("El felino descansa en el sillón")
emb3 = obtener_embedding("La bolsa de valores bajó un 3%")

print(similitud_coseno(emb1, emb2))  # ~0.92 — semánticamente similares
print(similitud_coseno(emb1, emb3))  # ~0.15 — semánticamente distantes
```

Los embeddings son la base de los sistemas RAG — se almacenan en bases de datos vectoriales y se recuperan por similitud semántica.

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - Los LLMs son modelos autorregresivos que predicen el siguiente token basándose en el contexto anterior. No "razonan" — predicen continuaciones probables.
> - Un **token** no es una palabra. Estima ~0.75 palabras por token para costes. El límite de la context window está en tokens.
> - **Temperature = 0** para tareas deterministas (código, extracción). **0.5-0.7** para texto general. **>0.9** para creatividad.
> - Los **embeddings** convierten texto en vectores numéricos donde la distancia euclidiana o la similitud coseno mide la semejanza semántica.
> - El rendimiento de los LLMs puede degradarse en contextos muy largos — los documentos del medio tienden a ser "olvidados".

### Recursos
- 📄 *Attention Is All You Need* — Vaswani et al., 2017 (el paper original del transformer)
- 🌐 jalammar.github.io/illustrated-transformer — la mejor explicación visual del transformer
- 📖 *Build a Large Language Model From Scratch* — Sebastian Raschka
- 🌐 docs.anthropic.com — documentación de la API de Claude

---
`#ia` `#llm` `#transformer` `#tokens` `#embeddings`
