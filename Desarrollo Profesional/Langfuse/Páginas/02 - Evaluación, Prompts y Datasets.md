# 📊 Langfuse 02 — Evaluación, Prompts y Datasets

[[Desarrollo Profesional/Langfuse/Langfuse|⬅️ Volver a Langfuse]] | [[Desarrollo Profesional/Langfuse/Páginas/01 - Tracing y Observabilidad|← 01]]

> [!abstract] Introducción
> Trazar te dice *qué pasó*; evaluar te dice *si fue bueno*. El reto de los LLMs es que no hay un "correcto/incorrecto" binario: la calidad es matizada y subjetiva. Langfuse aborda esto con tres herramientas que cierran el ciclo de mejora: **scores** (puntuar las respuestas, por humanos, por código o por otro LLM), **prompt management** (versionar prompts fuera del código) y **datasets** (conjuntos de casos para probar cambios antes de desplegarlos). Juntos convierten "creo que ha mejorado" en "lo he medido".

## ¿De qué vamos a hablar?

Cómo puntuar la calidad (scores y LLM-as-a-judge), cómo gestionar prompts versionados, y cómo usar datasets para evaluar cambios de forma sistemática.

### Conceptos que vamos a cubrir
- Scores: evaluación manual, programática y LLM-as-a-judge
- Prompt management: versionar y desacoplar del código
- Datasets y experimentos (evals offline)
- El ciclo de mejora continua

---

## El Concepto

### Scores: puntuar la calidad

Un *score* es una puntuación asociada a un trace u observation. Hay tres fuentes:

- **Manual**: un humano revisa en la UI y puntúa (útil para crear un "gold standard").
- **Programática**: tu código mide algo objetivo (¿el JSON era válido?, ¿contenía la cita?).
- **LLM-as-a-judge**: otro LLM evalúa la respuesta según un criterio (relevancia, toxicidad, fidelidad al contexto).

```python
from langfuse import observe, get_client

langfuse = get_client()

@observe()
def responder(pregunta: str) -> str:
    respuesta = "..."  # llamada al LLM
    # score programático: penaliza respuestas vacías o demasiado largas
    longitud_ok = 10 < len(respuesta) < 2000
    langfuse.score_current_trace(name="longitud_ok", value=1 if longitud_ok else 0)
    return respuesta
```

> [!info] LLM-as-a-judge en una frase
> Usar un LLM para evaluar la salida de otro LLM. Se le da la respuesta (y a veces la referencia) y un criterio, y devuelve una puntuación. Escala mucho más que la revisión humana, aunque conviene calibrarlo contra juicios humanos. Langfuse permite configurarlo sobre los traces que van llegando.

### Prompt management

Tener los prompts incrustados en el código obliga a desplegar para cambiarlos y dificulta saber qué versión produjo qué. Langfuse los guarda **versionados** y los sirve por API: editas el prompt en la UI, marcas una versión como `production`, y tu código la consume.

```python
prompt = langfuse.get_prompt("soporte-clientes")  # trae la versión 'production'
texto = prompt.compile(nombre="Ana", producto="Plan Pro")  # rellena variables

# El trace queda enlazado a la versión del prompt usada → puedes comparar
# el rendimiento de la v3 frente a la v4 con datos reales.
```

Esto desacopla el prompt del despliegue y, al enlazar cada trace con su versión, permite comparar versiones con métricas reales.

### Datasets y experimentos

Un *dataset* es una colección de casos (`input` esperado → `output` de referencia opcional). Antes de cambiar un prompt o un modelo, ejecutas tu aplicación sobre el dataset y comparas resultados: es el equivalente a una **suite de tests** para IA (una *eval* offline).

```python
dataset = langfuse.get_dataset("preguntas-soporte")

for item in dataset.items:
    with item.run(run_name="prompt-v4") as root_span:
        salida = responder(item.input)       # tu función bajo prueba
        root_span.score(name="acierto", value=evaluar(salida, item.expected_output))
# En la UI comparas la run 'prompt-v4' contra 'prompt-v3' caso a caso
```

### El ciclo de mejora continua

```
Producción ──▶ Traces (qué pasó) ──▶ Scores (qué fue mal)
     ▲                                      │
     │                            casos interesantes
     │                                      ▼
 Desplegar ◀── Experimento sobre Dataset ◀── ajustar prompt/modelo
     (solo si la eval mejora)
```

Los fallos detectados en producción se añaden al dataset; cada cambio se valida contra ese dataset antes de desplegar. Así la app mejora de forma medible en lugar de por intuición. Es la [[Desarrollo Profesional/Estrategia de Testing/Estrategia de Testing|estrategia de testing]] adaptada a lo no determinista.

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - Trazar dice *qué pasó*; **evaluar** dice *si fue bueno*. La calidad en LLMs no es binaria.
> - Los **scores** puntúan respuestas de tres formas: **manual**, **programática** y **LLM-as-a-judge**.
> - El **prompt management** versiona los prompts fuera del código y enlaza cada trace con su versión, permitiendo comparar.
> - Los **datasets** son la suite de tests de la IA: ejecutas la app sobre casos y comparas versiones (evals offline) antes de desplegar.
> - El **ciclo**: producción → traces → scores → dataset → experimento → desplegar solo si la eval mejora.

### Para llevar a la práctica
- [ ] Añade un score programático a un trace (p. ej. validez del JSON de salida)
- [ ] Mueve un prompt al prompt management de Langfuse y consúmelo con `get_prompt`
- [ ] Crea un dataset de 5–10 casos y compara dos versiones de un prompt con una eval

### Recursos
- 🌐 langfuse.com/docs/scores — scores y evaluación
- 🌐 langfuse.com/docs/prompts — prompt management
- 🌐 langfuse.com/docs/datasets — datasets y experimentos

---
`#langfuse` `#evaluacion` `#prompts` `#ia`
