# 🔁 Loop Engineering — Ingeniería de Bucles Agénticos

[[Desarrollo Profesional/Inicio Profesional|⬅️ Volver a Desarrollo Profesional]]

> [!abstract] Sobre esta sección
> El *loop engineering* es la disciplina que ha emergido alrededor de una idea simple: un LLM por sí solo responde una vez, pero un LLM **dentro de un bucle** —que actúa, observa el resultado y vuelve a decidir— se convierte en un agente capaz de resolver tareas largas y abiertas. Construir esos bucles bien (qué herramientas das, qué contexto entra en cada vuelta, cuándo para, cómo verificas) es lo que separa una demo de un agente fiable. Es la evolución natural del *prompt engineering*: ya no diseñas un mensaje, diseñas el bucle entero.

---

## 📚 Páginas de esta sección

1. [[Desarrollo Profesional/Loop Engineering/Páginas/01 - Qué es y el Bucle Agéntico|01 — Qué es y el Bucle Agéntico]] — del prompt al loop, la anatomía del bucle percibir-pensar-actuar-observar
2. [[Desarrollo Profesional/Loop Engineering/Páginas/02 - Diseño del Bucle y Herramientas|02 — Diseño del Bucle y Herramientas]] — el harness, diseño de tools, context engineering y condiciones de parada
3. [[Desarrollo Profesional/Loop Engineering/Páginas/03 - Verificación Evals y Guardarraíles|03 — Verificación, Evals y Guardarraíles]] — cerrar el bucle con verificación, evals, human-in-the-loop y seguridad
4. [[Desarrollo Profesional/Loop Engineering/Páginas/04 - Operación Autonomía y Antipatrones|04 — Operación, Autonomía y Antipatrones]] — agentes autónomos, bucles en background, coste, observabilidad y errores típicos

---

## ¿Qué es Loop Engineering en una frase?

Es el arte de diseñar y operar el bucle de feedback en el que vive un agente de IA: qué ve, qué puede hacer, cómo comprueba si fue bien y cuándo se detiene — de forma que el conjunto sea fiable, barato y seguro.

---

## Ruta recomendada

Empieza por la 01 para entender el bucle. La 02 es el corazón del diseño (tools y contexto); la 03 es lo que hace al agente fiable (verificación y evals); la 04 te prepara para ponerlo a correr de verdad. Esta sección se apoya en [[Desarrollo Profesional/IA/Páginas/04 - Agentes y MCP|IA: Agentes y MCP]] y se cruza con [[Desarrollo Profesional/Pydantic/Pydantic|Pydantic]], [[Desarrollo Profesional/LangChain/LangChain|LangChain]] y [[Desarrollo Profesional/Langfuse/Langfuse|Langfuse]].

---
`#loop-engineering` `#agentes` `#ia` `#indice`
