# 🔁 Loop Engineering 04 — Operación, Autonomía y Antipatrones

[[Desarrollo Profesional/Loop Engineering/Loop Engineering|⬅️ Volver a Loop Engineering]] | [[Desarrollo Profesional/Loop Engineering/Páginas/03 - Verificación Evals y Guardarraíles|← 03]]

> [!abstract] Introducción
> Diseñar el bucle es una cosa; ponerlo a correr de verdad —solo, durante horas, costando dinero y tocando sistemas reales— es otra. La última pieza del loop engineering es operacional: cómo das autonomía de forma gradual, cómo orquestas varios agentes, cómo observas lo que hace un sistema no determinista y cómo controlas el coste. Y, sobre todo, qué errores evitar, porque casi todos los fracasos de agentes repiten los mismos patrones.

## ¿De qué vamos a hablar?

De llevar el bucle a producción: niveles de autonomía, arquitecturas multi-agente, observabilidad, coste y los antipatrones que hunden la mayoría de los proyectos agénticos.

### Conceptos que vamos a cubrir
- Niveles de autonomía: del copiloto al agente en background
- Orquestación: un agente vs varios
- Observabilidad de sistemas no deterministas
- Economía del bucle: coste y latencia
- Antipatrones del loop engineering

---

## El Concepto

### Niveles de Autonomía

La autonomía no es un interruptor, es un dial que subes a medida que la confianza (medida con evals, ver [[Desarrollo Profesional/Loop Engineering/Páginas/03 - Verificación Evals y Guardarraíles|página 03]]) lo permite:

```
Copiloto ───▶ Supervisado ───▶ Semi-autónomo ───▶ Autónomo (background)
sugiere,      actúa, tú        actúa solo,         arranca solo y trabaja
tú ejecutas   apruebas cada    pides revisión      hasta terminar; revisas
              acción           al final            el resultado
```

La tendencia de moda es precisamente la del extremo derecho: agentes que arrancan, trabajan en **background** durante un rato largo (minutos u horas) sobre una tarea entera y te entregan un resultado para revisar — un PR, un informe, un refactor. Es el bucle de la página 01 corriendo solo, con buenos frenos (página 02) y buena verificación (página 03). No empieces por ahí: gánate cada nivel de autonomía con evidencia.

### Orquestación: Un Agente o Varios

Cuando una tarea se hace grande, hay dos caminos:
- **Un solo bucle.** Más simple, más barato, más fácil de depurar. **Es el punto de partida correcto casi siempre.** No metas multi-agente hasta que un agente claramente no llegue.
- **Multi-agente (orquestador-trabajadores).** Un agente coordinador reparte subtareas a agentes especializados que corren en paralelo, cada uno con su propio contexto y herramientas. Ayuda cuando la tarea se paraleliza de forma natural o cuando separar contextos evita que la ventana se sature. El precio es coordinación, coste multiplicado y depuración más difícil.

```
        ┌──────────────┐
        │ Orquestador  │  descompone el objetivo y reparte
        └──┬───┬───┬───┘
           ▼   ▼   ▼
        ┌────┐┌────┐┌────┐  cada trabajador: su propio bucle,
        │ A  ││ B  ││ C  │  su contexto y sus herramientas
        └──┬─┘└──┬─┘└──┬─┘
           └─────┴─────┘  → resultados de vuelta al orquestador
```

Regla práctica: **un buen bucle único bate a un multi-agente mediocre**. La complejidad multi-agente solo se paga sola en problemas que de verdad la necesitan.

### Observabilidad de lo No Determinista

Un agente no es determinista: la misma entrada puede dar caminos distintos. Eso rompe la depuración clásica y hace la observabilidad **imprescindible**:
- **Traza cada vuelta**: prompt, decisión, tool call, resultado, tokens y coste. Cuando algo va mal, necesitas reconstruir exactamente qué vio y qué decidió el agente en cada paso.
- **Mide en agregado**: tasa de éxito, pasos por tarea, coste por tarea, dónde se atasca. Herramientas como [[Desarrollo Profesional/Langfuse/Langfuse|Langfuse]] están hechas para esto; los principios generales de [[Desarrollo Profesional/Prometheus/Prometheus|métricas]] y [[Desarrollo Profesional/Grafana/Grafana|dashboards]] siguen aplicando.
- **Reproduce los fallos**: guarda las trazas de los casos que fallan para convertirlos en nuevos casos de eval.

### La Economía del Bucle

Cada vuelta del bucle es una llamada al modelo, y las llamadas cuestan dinero y tiempo. Un bucle que da 30 vueltas con todo el contexto cada vez puede ser sorprendentemente caro. Palancas:
- **El modelo adecuado a cada paso**: un modelo pequeño y rápido para pasos mecánicos, uno grande solo para los que requieren razonamiento (ver [[Desarrollo Profesional/IA/Páginas/01 - LLMs y Fundamentos|LLMs y fundamentos]]).
- **Caché de prompt**: si tu harness lo soporta, cachear la parte estable del contexto reduce coste y latencia drásticamente.
- **Menos vueltas, mejores**: a veces una herramienta más potente ahorra cinco vueltas. Optimiza el número de pasos, no solo el coste por paso.
- **Presupuesto por tarea**: pon un techo de coste y mídelo siempre (página 02).

### Antipatrones del Loop Engineering

Los fracasos de agentes son sorprendentemente repetitivos. Evita estos:

- **Agente para todo.** Si la tarea es determinista y conocida, escribe código normal. Un bucle de LLM para algo que es un `for` es caro, lento y menos fiable. *El agente es para lo abierto y variable, no para lo que ya sabes programar.*
- **Bucle sin frenos.** Sin tope de pasos/tiempo/coste → bucles infinitos y facturas sorpresa.
- **Bucle sin verificación.** Generar sin comprobar es producir plausibilidad (página 03).
- **Contexto que se hincha.** Acumular toda la historia sin compactar degrada calidad y dispara el coste (página 02).
- **Multi-agente prematuro.** Añadir agentes antes de exprimir uno solo: complejidad sin retorno.
- **Tools mal descritas.** Si el modelo elige mal o pasa argumentos malos, casi siempre es culpa de la descripción, no del modelo.
- **Autonomía sin evidencia.** Soltar un agente a producción sin evals que respalden la confianza.
- **Antropomorfizar el bucle.** "El agente entiende"… no: predice tokens dentro de un bucle que tú diseñaste. Razona sobre mecanismos (contexto, tools, verificación), no sobre intenciones.

> [!tip] El consejo que resume la sección
> Empieza **simple y supervisado**: un bucle, pocas tools, verificación objetiva, humano aprobando. Mide con evals. Sube la autonomía y la complejidad **solo** cuando los datos digan que el bucle es fiable. El loop engineering premia la disciplina y el escepticismo, no la sofisticación.

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - La **autonomía es un dial**: copiloto → supervisado → semi-autónomo → autónomo en background. La moda son los agentes en background, pero se gana cada nivel con evidencia.
> - **Un solo bucle es el punto de partida**; el multi-agente (orquestador-trabajadores) solo compensa en problemas que de verdad lo necesitan.
> - Los agentes son **no deterministas**: la observabilidad (trazar cada vuelta, medir en agregado, reproducir fallos) es imprescindible, no opcional.
> - La **economía del bucle** importa: modelo adecuado por paso, caché de prompt, menos vueltas mejores y presupuesto por tarea.
> - Evita los **antipatrones**: agente para lo determinista, bucle sin frenos ni verificación, contexto hinchado, multi-agente prematuro, tools mal descritas y autonomía sin evals.

### Para llevar a la práctica
- [ ] Sitúa tu agente en el dial de autonomía y define qué evidencia necesitarías para subirlo un nivel
- [ ] Antes de ir multi-agente, exprime un solo bucle y comprueba si de verdad se queda corto
- [ ] Instrumenta tu bucle para guardar cada vuelta (prompt, tool, resultado, coste) y poder reproducir fallos
- [ ] Revisa tu agente contra la lista de antipatrones y corrige el que más te pese

### Recursos
- 🌐 anthropic.com/engineering/building-effective-agents — un agente vs orquestación y cuándo cada uno
- 🌐 anthropic.com/engineering/built-multi-agent-research-system — lecciones reales de un sistema multi-agente
- 📊 [[Desarrollo Profesional/Langfuse/Langfuse|Langfuse]] — observabilidad y trazas de agentes en producción
- 🤖 [[Desarrollo Profesional/IA/Páginas/04 - Agentes y MCP|IA: Agentes y MCP]] — el bucle de agente y el protocolo de herramientas

---
`#loop-engineering` `#autonomia` `#multi-agente` `#observabilidad` `#antipatrones`
