# 📊 Langfuse 01 — Tracing y Observabilidad

[[Desarrollo Profesional/Langfuse/Langfuse|⬅️ Volver a Langfuse]] | [[Desarrollo Profesional/Langfuse/Páginas/02 - Evaluación, Prompts y Datasets|02 →]]

> [!abstract] Introducción
> Depurar una app de IA sin observabilidad es ir a ciegas: un agente que llama a tres tools y dos LLMs es imposible de entender desde los logs. Langfuse modela cada ejecución como un **trace** (la operación completa) compuesto de **observations** anidadas: *spans* (pasos), *generations* (llamadas a LLM) y *events*. Verlo como un árbol jerárquico —con sus entradas, salidas, tokens, coste y latencia— es lo que convierte la caja negra en algo depurable.

## ¿De qué vamos a hablar?

El modelo de datos de Langfuse (trace → observations), las tres formas de instrumentar tu código, y qué métricas captura automáticamente.

### Conceptos que vamos a cubrir
- Traces y observations (spans, generations, events)
- Instrumentar con el decorador `@observe`
- Integraciones automáticas (LangChain, OpenAI/Anthropic)
- Qué se captura: tokens, coste, latencia, metadatos

---

## El Concepto

### El modelo: trace y observations

```
Trace: "responder_consulta_soporte"   (toda la petición del usuario)
├── Span: "recuperar_contexto"        (un paso lógico)
│   └── Generation: embedding query   (llamada a un modelo)
├── Generation: "llamada_llm_principal" (prompt, respuesta, tokens, coste)
└── Span: "post_procesado"
```

- **Trace**: la unidad de alto nivel, una operación completa de cara al usuario.
- **Observation**: cada paso dentro del trace. Tipos: **span** (un bloque de trabajo), **generation** (una llamada a un LLM, con su modelo, tokens y coste) y **event** (un punto puntual).

### Instrumentar con @observe

La vía más directa en Python: el decorador `@observe()`. Envuelve una función y crea automáticamente el span/generation, capturando entradas y salidas.

```python
from langfuse import observe, get_client

langfuse = get_client()  # usa LANGFUSE_PUBLIC_KEY / LANGFUSE_SECRET_KEY del entorno

@observe()
def recuperar_contexto(pregunta: str) -> list[str]:
    # ... lógica de RAG ...
    return ["fragmento 1", "fragmento 2"]

@observe()
def responder(pregunta: str) -> str:
    contexto = recuperar_contexto(pregunta)   # queda anidado como hijo en el trace
    # ... llamada al LLM ...
    return "respuesta"

responder("¿Cómo cancelo mi suscripción?")  # genera un trace con su jerarquía
```

Las funciones anidadas se enlazan solas: `recuperar_contexto` aparece como hija de `responder` en el mismo trace.

### Integraciones automáticas

No hace falta instrumentar a mano las llamadas al LLM. Langfuse se integra con los SDKs y frameworks habituales y captura las *generations* solo:

```python
# Con LangChain: un callback handler que traza toda la cadena
from langfuse.langchain import CallbackHandler

handler = CallbackHandler()
cadena.invoke({"pregunta": "..."}, config={"callbacks": [handler]})
# Cada paso de la cadena LCEL aparece como observation en el trace
```

También hay wrappers para los clientes de OpenAI/Anthropic y soporte vía OpenTelemetry, con lo que se integra con [[Desarrollo Profesional/Pydantic/Páginas/03 - PydanticAI y Agentes|PydanticAI]] y [[Desarrollo Profesional/LiveKit/LiveKit|LiveKit]].

### Qué se captura

Por cada generation, Langfuse registra automáticamente:

| Métrica | Para qué sirve |
|---|---|
| Prompt y respuesta | Depurar qué entró y salió exactamente |
| Tokens (entrada/salida) | Entender el consumo |
| **Coste** estimado | Controlar el gasto por trace/usuario/feature |
| **Latencia** | Detectar cuellos de botella (clave en voz) |
| Modelo y parámetros | Comparar configuraciones |
| Metadatos y `user_id`/`session_id` | Filtrar, agrupar y seguir conversaciones |

> [!tip] Añade contexto desde el principio
> Etiqueta los traces con `user_id`, `session_id` y `tags` desde el día uno. Cuando algo falle en producción, poder filtrar "todos los traces de este usuario en esta sesión" convierte una investigación de horas en una de minutos.

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - Langfuse modela cada ejecución como un **trace** (operación completa) con **observations** anidadas: **spans**, **generations** y **events**.
> - El decorador **`@observe()`** instrumenta funciones Python y anida automáticamente las llamadas hijas en el trace.
> - Las **integraciones** (LangChain callback, wrappers de OpenAI/Anthropic, OpenTelemetry) capturan las generations sin código manual.
> - Por cada generation se registran **prompt, respuesta, tokens, coste, latencia y modelo**.
> - Etiqueta con **`user_id`/`session_id`/`tags`** desde el principio para poder investigar producción rápido.

### Para llevar a la práctica
- [ ] Instrumenta una función con `@observe()` y mira el trace en la UI de Langfuse
- [ ] Conecta el callback handler a una cadena de LangChain y observa la jerarquía
- [ ] Añade `user_id` y `tags` a un trace y filtra por ellos en el dashboard

### Recursos
- 🌐 langfuse.com/docs/tracing — modelo de tracing
- 🌐 langfuse.com/docs/sdk/python/decorators — el decorador `@observe`
- 🌐 langfuse.com/docs/integrations — integraciones disponibles

---
`#langfuse` `#tracing` `#observabilidad` `#ia`
