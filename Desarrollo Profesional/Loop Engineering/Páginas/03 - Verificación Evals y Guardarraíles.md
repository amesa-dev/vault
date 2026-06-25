# 🔁 Loop Engineering 03 — Verificación, Evals y Guardarraíles

[[Desarrollo Profesional/Loop Engineering/Loop Engineering|⬅️ Volver a Loop Engineering]] | [[Desarrollo Profesional/Loop Engineering/Páginas/02 - Diseño del Bucle y Herramientas|← 02]] | [[Desarrollo Profesional/Loop Engineering/Páginas/04 - Operación Autonomía y Antipatrones|04 →]]

> [!abstract] Introducción
> Un bucle que actúa pero no comprueba si acertó es un generador de plausibilidad, no de resultados. Lo que convierte un agente en algo fiable es **cerrar el bucle con verificación**: darle una forma objetiva de saber si lo que hizo está bien, para que pueda corregirse. Y como un agente actúa sobre el mundo, necesita además **guardarraíles** que limiten lo que puede hacer y **evals** que midan si el conjunto mejora o empeora cuando lo tocas. Esta es la página que separa los juguetes de los sistemas en los que confías.

## ¿De qué vamos a hablar?

De cómo darle al bucle un criterio de verdad (verificación), cómo medir su calidad de forma sistemática (evals), cómo mantener al humano en el bucle cuando hace falta y cómo ponerle límites de seguridad.

### Conceptos que vamos a cubrir
- Cerrar el bucle: verificadores y el bucle generar-verificar
- Tests, compiladores y validadores como señal objetiva
- Evals: medir agentes sin engañarte
- Human-in-the-loop: aprobación y escalada
- Guardarraíles y seguridad del bucle

---

## El Concepto

### Cerrar el Bucle con Verificación

La diferencia entre un agente que impresiona y uno que sirve es si **puede comprobar su propio trabajo**. Un bucle ciego genera una respuesta plausible y la da por buena. Un bucle cerrado genera, verifica con una señal objetiva, y si la verificación falla, vuelve a intentarlo con ese feedback. Es el patrón **generar → verificar → corregir**.

```
        ┌─────────────────────────────────────────┐
        ▼                                          │
   ┌─────────┐    ┌──────────────┐   ¿pasa?  no    │
   │ generar │──▶ │  verificador  │───────────────┘  (corrige con el error)
   └─────────┘    └──────────────┘
                         │ sí
                         ▼
                      resultado
```

La clave es que la verificación sea **objetiva y barata**. Cuanto más fuerte y automática sea la señal, mejor se autocorrige el agente.

### Señales de Verificación Objetivas

Las mejores señales no las da otro LLM, las da la realidad:

- **Tests** (la reina). En un agente de programación, ejecutar la suite de tests es el verificador perfecto: verde o rojo, sin ambigüedad. El agente edita, corre los tests, lee el fallo y reitera. Por eso TDD y agentes encajan tan bien (ver [[Desarrollo Profesional/Estrategia de Testing/Páginas/01 - Pirámide y TDD|Estrategia de Testing]]).
- **Compiladores y type checkers.** `mypy`, `tsc`, el compilador: te dicen objetivamente si el código es válido.
- **Validadores de esquema.** Si la salida debe ser un JSON con cierta forma, validarlo con [[Desarrollo Profesional/Pydantic/Páginas/02 - Salida Estructurada y Tool Calling|Pydantic]] y devolver el error de validación al bucle fuerza una salida correcta.
- **Ejecución real.** ¿La query SQL devuelve filas? ¿El endpoint responde 200? La ejecución es verdad terreno.

```python
# Bucle generar-verificar con tests como señal objetiva
for intento in range(MAX_INTENTOS):
    codigo = agente.generar(tarea, feedback=ultimo_error)
    escribir(codigo)
    resultado = ejecutar_tests()              # señal objetiva: pasa o no
    if resultado.ok:
        return codigo                          # verificado, terminamos
    ultimo_error = resultado.salida            # el fallo entra como feedback
return "No se logró pasar los tests en el presupuesto dado"
```

Cuando no hay señal automática, puedes usar **LLM-as-a-judge** (un modelo evalúa la salida de otro), pero es más débil y sesgado: úsalo para criterios difusos (tono, relevancia), no como única red de seguridad para cosas verificables (ver [[Desarrollo Profesional/Langfuse/Páginas/02 - Evaluación, Prompts y Datasets|Langfuse: evaluación]]).

### Evals: Medir el Bucle sin Engañarte

Si tocas el prompt, cambias una tool o subes de modelo, ¿el agente mejora o empeora? Sin medirlo, vas a ciegas. Los **evals** son la suite de tests del comportamiento del agente: un conjunto de tareas representativas con un criterio de éxito, que ejecutas de forma sistemática.

- **Construye un dataset** de casos reales (incluidos los que fallaron en producción): entrada + resultado esperado o criterio de éxito.
- **Mide lo que importa**: tasa de éxito de la tarea (no "suena bien"), número de pasos, coste por tarea, latencia.
- **Evita el overfitting al eval**: si optimizas solo contra tus 20 casos, mejoras esos 20 y nada más. Renueva y amplía el dataset.

Los evals son a los agentes lo que los tests al código: la red que te deja iterar con confianza. La instrumentación para capturarlos se ve en [[Desarrollo Profesional/Langfuse/Páginas/01 - Tracing y Observabilidad|Langfuse]].

### Human-in-the-Loop

No todo el bucle debe ser autónomo. Mantener a una persona en puntos concretos es diseño, no fracaso:
- **Aprobación previa** para acciones irreversibles o caras: borrar datos, enviar dinero, publicar de cara al público. El agente propone, el humano confirma.
- **Escalada ante incertidumbre**: cuando el agente no está seguro o detecta ambigüedad, que pare y pregunte en vez de inventar.
- **Niveles de autonomía**: empieza con el humano aprobando cada acción y ve soltando lazo a medida que la confianza (medida con evals) lo justifica.

### Guardarraíles y Seguridad

Un agente actúa sobre el mundo, así que sus permisos son superficie de ataque y de error:

- **Mínimo privilegio.** Dale solo las herramientas y permisos que la tarea necesita. Un agente que solo lee no debería poder borrar (ver [[Desarrollo Profesional/Seguridad Aplicada/Páginas/02 - Autenticación y Autorización|Seguridad Aplicada]]).
- **Sandboxing.** Ejecuta el código y los comandos del agente en un entorno aislado ([[Desarrollo Profesional/Docker y Contenedores/Docker y Contenedores|contenedores]]), no en producción ni en tu máquina sin red de seguridad.
- **Validación de salidas y entradas.** Trata lo que el agente produce (y lo que le llega de fuentes externas) como no confiable. La **inyección de prompts** —datos externos que intentan secuestrar las instrucciones del agente— es la vulnerabilidad estrella de los sistemas agénticos.
- **Confirmación para lo irreversible.** Cualquier acción difícil de deshacer pasa por aprobación humana o por una comprobación extra.

> [!info] La asimetría que hay que recordar
> Generar es fácil y barato; verificar bien suele ser lo difícil. La mayor parte del esfuerzo de loop engineering de calidad se va en construir buenas señales de verificación, buenos evals y buenos guardarraíles — no en el prompt del agente.

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - Un agente fiable **cierra el bucle**: genera, **verifica con una señal objetiva** y corrige con ese feedback. Sin verificación solo hay plausibilidad.
> - Las mejores señales las da la realidad: **tests, compiladores, validadores de esquema, ejecución real**. LLM-as-a-judge solo para criterios difusos.
> - Los **evals** son los tests del comportamiento del agente: un dataset de tareas con criterio de éxito, midiendo tasa de éxito, pasos y coste — sin sobreajustar al propio eval.
> - **Human-in-the-loop** por diseño: aprobación de acciones irreversibles, escalada ante incertidumbre y autonomía creciente según la confianza medida.
> - **Guardarraíles**: mínimo privilegio, sandboxing, tratar entradas/salidas como no confiables (ojo a la inyección de prompts) y confirmar lo irreversible.

### Para llevar a la práctica
- [ ] Identifica la señal de verificación objetiva de tu tarea (tests, schema, ejecución) y mete su resultado en el bucle
- [ ] Monta un eval mínimo: 10 tareas con criterio de éxito y mide la tasa de acierto de tu agente
- [ ] Marca qué acciones de tu agente son irreversibles y ponles aprobación humana
- [ ] Ejecuta las acciones del agente en un sandbox y aplica mínimo privilegio a sus herramientas

### Recursos
- 🌐 anthropic.com/engineering/building-effective-agents — verificación, human-in-the-loop y patrones fiables
- 🌐 owasp.org/www-project-top-10-for-large-language-model-applications — riesgos de LLM, incluida inyección de prompts
- 📊 [[Desarrollo Profesional/Langfuse/Langfuse|Langfuse]] — instrumentar trazas y evals de agentes
- 🧪 [[Desarrollo Profesional/Estrategia de Testing/Estrategia de Testing|Estrategia de Testing]] — los tests como verificador del bucle

---
`#loop-engineering` `#verificacion` `#evals` `#guardarrailes` `#seguridad`
