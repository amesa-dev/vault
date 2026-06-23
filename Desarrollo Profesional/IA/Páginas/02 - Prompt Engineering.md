# 🤖 IA 02 — Prompt Engineering

[[Desarrollo Profesional/IA/IA|⬅️ Volver a IA]] | [[Desarrollo Profesional/IA/Páginas/01 - LLMs y Fundamentos|← 01]] | [[Desarrollo Profesional/IA/Páginas/03 - RAG|03 →]]

> [!abstract] Introducción
> El prompt engineering es el arte de comunicarse con un LLM de forma que produzca la respuesta más útil, precisa y reproducible. No es "trucar" al modelo — es dar suficiente contexto para que el modelo pueda hacer bien su trabajo. La diferencia entre un prompt mediocre y uno bien diseñado puede ser la diferencia entre un resultado inútil y uno que va directo a producción.

## ¿De qué vamos a hablar?

Las técnicas de prompt engineering desde las más básicas hasta las avanzadas: zero-shot, few-shot, chain-of-thought, system prompts y técnicas de control de formato.

### Conceptos que vamos a cubrir
- Zero-shot y few-shot: cuándo usar ejemplos
- Chain-of-thought: hacer que el modelo razone paso a paso
- System prompts: definir el rol y las restricciones
- Control de formato: outputs estructurados y JSON
- Técnicas avanzadas: self-consistency, ReAct

---

## El Concepto

### Zero-Shot vs Few-Shot

**Zero-shot**: el modelo realiza la tarea sin ejemplos. Funciona bien para tareas comunes y bien definidas:

```python
import anthropic

client = anthropic.Anthropic()

# Zero-shot — sin ejemplos
respuesta = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    messages=[{
        "role": "user",
        "content": "Clasifica el sentimiento del siguiente texto como POSITIVO, NEGATIVO o NEUTRO:\n\nTexto: 'El servicio al cliente fue horrible pero el producto llegó rápido.'"
    }]
)
```

**Few-shot**: incluyes ejemplos de entrada/salida para mostrar el formato y el comportamiento esperado. Crucial para tareas no estándar o cuando necesitas un formato específico:

```python
prompt_few_shot = """Clasifica la urgencia de los tickets de soporte como CRÍTICO, ALTO, MEDIO o BAJO.

Ejemplos:
Ticket: "El sistema de pagos está completamente caído en producción"
Urgencia: CRÍTICO

Ticket: "El gráfico de ventas del dashboard no carga en Safari"  
Urgencia: MEDIO

Ticket: "¿Se puede cambiar el color del botón de exportar?"
Urgencia: BAJO

Ticket: "Los usuarios premium no pueden acceder a sus informes desde esta mañana"
Urgencia:"""

respuesta = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=50,
    messages=[{"role": "user", "content": prompt_few_shot}]
)
# Respuesta: ALTO
```

### Chain-of-Thought (CoT)

Los LLMs cometen menos errores en tareas que requieren razonamiento cuando se les pide que muestren los pasos intermedios. "Piensa paso a paso" (think step by step) es la técnica más simple:

```python
# Sin CoT — propenso a errores en razonamiento complejo
prompt_sin_cot = "¿Cuántos días han pasado desde el 15 de marzo hasta el 22 de junio?"

# Con CoT — el modelo razona explícitamente
prompt_con_cot = """¿Cuántos días han pasado desde el 15 de marzo hasta el 22 de junio?
Piensa paso a paso antes de dar la respuesta final."""

# Zero-shot CoT — solo añadir "Piensa paso a paso"
# Few-shot CoT — incluir ejemplos que muestran el razonamiento

prompt_cot_fewshot = """Resuelve estos problemas mostrando tu razonamiento:

Problema: Si un tren va a 80 km/h durante 2.5 horas, ¿cuántos km recorre?
Razonamiento: distancia = velocidad × tiempo = 80 × 2.5 = 200 km
Respuesta: 200 km

Problema: Si Ana tiene 3 veces más dinero que Luis, y entre los dos tienen 120€, ¿cuánto tiene cada uno?
Razonamiento: Si Luis tiene X, Ana tiene 3X. X + 3X = 120, 4X = 120, X = 30.
Luis tiene 30€, Ana tiene 90€.
Respuesta: Luis 30€, Ana 90€

Problema: Un proyecto dura 6 semanas. Si ya llevamos 18 días, ¿qué porcentaje hemos completado?
Razonamiento:"""
```

### System Prompts — Definir el Rol y las Restricciones

El system prompt es el lugar donde defines quién es el modelo, qué debe hacer y qué no debe hacer. Se procesa antes que la conversación del usuario y tiene más "peso" que el mensaje de usuario:

```python
respuesta = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=2048,
    system="""Eres un revisor de código Python experto con 15 años de experiencia.
Tu objetivo es revisar código Python y dar feedback constructivo.

FORMATO DE TU RESPUESTA:
1. Resumen (1-2 líneas de lo más importante)
2. Problemas críticos (si los hay)
3. Sugerencias de mejora
4. Puntos positivos

RESTRICCIONES:
- Siempre explica el porqué de cada sugerencia
- Usa ejemplos de código cortos para ilustrar
- Si el código es correcto, dilo explícitamente — no inventes problemas
- Sé conciso: máximo 400 palabras en total""",
    messages=[{
        "role": "user",
        "content": f"Revisa este código:\n\n```python\n{codigo_usuario}\n```"
    }]
)
```

### Control de Formato — Outputs Estructurados

Para integrar la salida de un LLM en un sistema, necesitas que el formato sea predecible:

```python
# Forzar JSON — usa XML tags para delimitar la respuesta
prompt_json = """Analiza este texto de reseña de producto y extrae la información en JSON.

Responde SOLO con el JSON, sin texto adicional, dentro de tags <json>...</json>.

Formato esperado:
<json>
{
  "sentimiento": "positivo|negativo|neutro",
  "puntuacion": 1-5,
  "aspectos_positivos": ["lista", "de", "aspectos"],
  "aspectos_negativos": ["lista", "de", "aspectos"],
  "producto_mencionado": "nombre del producto o null"
}
</json>

Reseña: "El teclado mecánico es fantástico, la respuesta de las teclas es increíble y el sonido es satisfactorio. Lleva funcionando 2 años sin ningún problema. El único inconveniente es que el cable USB se podría haber hecho más largo."
"""

import json
import re

respuesta_texto = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=512,
    temperature=0,  # temperatura 0 para outputs deterministas
    messages=[{"role": "user", "content": prompt_json}]
).content[0].text

# Extraer el JSON de los tags
match = re.search(r'<json>(.*?)</json>', respuesta_texto, re.DOTALL)
if match:
    datos = json.loads(match.group(1))
    print(datos["sentimiento"])  # "positivo"
```

### Técnicas Avanzadas

**Self-consistency**: genera múltiples respuestas y elige por votación mayoritaria. Mejora la precisión en tareas de razonamiento:

```python
import asyncio

async def respuesta_con_consistencia(pregunta: str, n_muestras: int = 5) -> str:
    respuestas = await asyncio.gather(*[
        asyncio.to_thread(
            lambda: client.messages.create(
                model="claude-sonnet-4-6",
                max_tokens=200,
                temperature=0.7,
                messages=[{"role": "user", "content": pregunta}]
            ).content[0].text
        )
        for _ in range(n_muestras)
    ])

    # Extraer las respuestas finales y elegir la más común
    from collections import Counter
    respuestas_finales = [r.split("Respuesta:")[-1].strip() for r in respuestas]
    return Counter(respuestas_finales).most_common(1)[0][0]
```

**Prompting con XML para Claude**: Claude está entrenado para entender bien la estructura XML:

```python
prompt_estructurado = """<task>
  <description>Analizar el siguiente artículo técnico y extraer información clave</description>
  <article>
    {articulo}
  </article>
  <requirements>
    <item>Resumen de 3 puntos máximo</item>
    <item>Lista de tecnologías mencionadas</item>
    <item>Nivel de dificultad: básico/intermedio/avanzado</item>
  </requirements>
</task>"""
```

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - **Few-shot** mejora los resultados cuando el modelo no sabe exactamente el formato o el estilo que necesitas. 3-5 ejemplos suelen ser suficientes.
> - **Chain-of-thought** ("piensa paso a paso") reduce significativamente los errores en tareas de razonamiento aritmético y lógico.
> - El **system prompt** define el rol, el formato y las restricciones. Es la base de cualquier asistente o agente especializado.
> - Para outputs estructurados, usa `temperature=0` y delimita el JSON/XML con tags — facilita la extracción con regex.
> - La **self-consistency** mejora la precisión en tareas complejas a costa de más tokens — solo merece la pena cuando la exactitud es crítica.

### Recursos
- 📄 *Chain-of-Thought Prompting Elicits Reasoning in Large Language Models* — Wei et al., 2022
- 🌐 docs.anthropic.com/en/docs/build-with-claude/prompt-engineering
- 🌐 learnprompting.org — guía interactiva de técnicas de prompting
- 📖 *The Prompt Report* — Schulhoff et al., 2024 (survey exhaustivo de técnicas)

---
`#ia` `#prompt-engineering` `#llm` `#chain-of-thought` `#few-shot`
