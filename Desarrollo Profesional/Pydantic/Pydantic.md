# 🧱 Pydantic (enfocado en IA)

[[Desarrollo Profesional/Inicio Profesional|⬅️ Volver a Desarrollo Profesional]]

> [!abstract] Sobre esta sección
> Pydantic se conoce como librería de validación de datos, pero su papel más interesante hoy es ser **la columna vertebral del desarrollo con LLMs en Python**. Un modelo de Pydantic es a la vez un *esquema* que describes una vez y que sirve para: forzar a un LLM a devolver **salida estructurada**, definir los **argumentos de las herramientas** (function/tool calling), validar y reintentar respuestas, y tipar los datos de un **agente**. Esta sección deja de lado el uso clásico como parser y se centra en cómo Pydantic hace fiable lo que de otro modo sería texto impredecible.

---

## 📚 Páginas de esta sección

1. [[Desarrollo Profesional/Pydantic/Páginas/01 - Modelos como Esquema para LLMs|01 — Modelos como Esquema para LLMs]] — `BaseModel`, JSON Schema, `Field`, validadores y por qué importan en IA
2. [[Desarrollo Profesional/Pydantic/Páginas/02 - Salida Estructurada y Tool Calling|02 — Salida Estructurada y Tool Calling]] — structured outputs, Instructor, function calling, reintentos ante validación
3. [[Desarrollo Profesional/Pydantic/Páginas/03 - PydanticAI y Agentes|03 — PydanticAI y Agentes]] — agentes tipados, dependencias, tools y resultados validados

---

## ¿Por qué Pydantic en IA, en una frase?

Un LLM produce texto; Pydantic convierte ese texto en **objetos Python validados y tipados**, y de paso le dice al modelo —vía JSON Schema— exactamente qué forma debe tener su respuesta. Es el puente entre la imprevisibilidad del lenguaje natural y la rigidez que necesita tu código.

> [!tip] Relación con el resto del vault
> Esta sección conecta con [[Desarrollo Profesional/IA/Páginas/02 - Prompt Engineering|Prompt Engineering]] (structured output) y [[Desarrollo Profesional/IA/Páginas/04 - Agentes y MCP|Agentes y MCP]]. Pydantic es la pieza de tipado que sostiene a [[Desarrollo Profesional/LangChain/LangChain|LangChain]] y a frameworks de agentes.

---

## 🔗 Recursos Generales
- 🌐 docs.pydantic.dev — documentación oficial (v2)
- 🌐 python.useinstructor.com — Instructor, structured outputs sobre Pydantic
- 🌐 ai.pydantic.dev — PydanticAI, el framework de agentes

---
`#pydantic` `#ia` `#llm` `#indice`
