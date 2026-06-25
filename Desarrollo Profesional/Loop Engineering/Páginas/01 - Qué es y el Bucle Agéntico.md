# 🔁 Loop Engineering 01 — Qué es y el Bucle Agéntico

[[Desarrollo Profesional/Loop Engineering/Loop Engineering|⬅️ Volver a Loop Engineering]] | [[Desarrollo Profesional/Loop Engineering/Páginas/02 - Diseño del Bucle y Herramientas|02 →]]

> [!abstract] Introducción
> Durante un par de años, sacarle partido a un LLM fue sobre todo cuestión de escribir el prompt perfecto. Pero un prompt es un disparo único: el modelo responde y se acabó. La idea que lo ha cambiado todo es meter al modelo en un **bucle**: dejar que actúe sobre el mundo, ver qué pasa y volver a decidir con esa información nueva. Ese bucle es lo que convierte un modelo de lenguaje en un *agente*. El loop engineering es la disciplina de diseñar ese bucle a conciencia.

## ¿De qué vamos a hablar?

De por qué el bucle es el cambio de paradigma, de su anatomía exacta y de en qué se diferencia esta disciplina del prompt engineering del que viene.

### Conceptos que vamos a cubrir
- Del prompt de un solo disparo al agente en bucle
- La anatomía del bucle: percibir → pensar → actuar → observar
- Por qué el bucle desbloquea tareas largas y abiertas
- Loop engineering vs prompt engineering vs context engineering

---

## El Concepto

### Del Disparo Único al Bucle

Un LLM "desnudo" hace una cosa: recibe un texto y predice el siguiente. Es brillante para responder, resumir o traducir, pero no puede *hacer* nada en el mundo ni corregirse a sí mismo. Si se equivoca, no se entera.

El salto agéntico consiste en darle dos cosas: **herramientas** (acciones que puede invocar: leer un fichero, ejecutar código, buscar en la web) y un **bucle** que ejecute esas acciones, le devuelva el resultado y le deje decidir de nuevo. Lo vimos en miniatura en [[Desarrollo Profesional/IA/Páginas/04 - Agentes y MCP|IA: Agentes y MCP]]; aquí lo convertimos en disciplina.

```
PROMPT (un disparo)              AGENTE (en bucle)
┌──────────┐                     ┌──────────┐
│  prompt  │                     │  objetivo │
└────┬─────┘                     └────┬─────┘
     ▼                                ▼  ┌─────────────────────┐
┌──────────┐                     ┌────┴────┐                  │
│ respuesta│                     │  LLM    │── decide acción ─▶│ herramienta
└──────────┘                     └────┬────┘                  │
   (fin)                              ▲  resultado/observación │
                                      └────────────────────────┘
                                      (repite hasta terminar)
```

### La Anatomía del Bucle

Todo bucle agéntico, por sofisticado que sea, tiene la misma forma de cuatro fases que se repiten:

1. **Percibir** — el agente recibe el estado actual: el objetivo, la conversación hasta ahora y el resultado de la última acción.
2. **Pensar** — el LLM razona sobre qué hacer a continuación (a veces de forma explícita, *chain-of-thought*).
3. **Actuar** — elige e invoca una herramienta con unos argumentos (un *tool call*).
4. **Observar** — el bucle ejecuta la herramienta y le devuelve el resultado, que pasa a formar parte de lo que percibe en la siguiente vuelta.

El bucle gira hasta que se cumple una **condición de parada**: el agente declara la tarea terminada, se agota un presupuesto (de pasos, tiempo o dinero) o salta un guardarraíl. Un esqueleto mínimo en Python:

```python
def bucle_agente(objetivo: str, herramientas: dict, max_pasos: int = 20) -> str:
    mensajes = [
        {"role": "system", "content": "Eres un agente. Usa herramientas hasta lograr el objetivo."},
        {"role": "user", "content": objetivo},
    ]
    for _ in range(max_pasos):                          # presupuesto: condición de parada dura
        respuesta = llm(mensajes, tools=esquemas(herramientas))   # PERCIBIR + PENSAR

        if respuesta.tool_calls:                         # ACTUAR
            mensajes.append(respuesta.mensaje)
            for call in respuesta.tool_calls:
                resultado = herramientas[call.nombre](**call.args)   # ejecuta la tool
                mensajes.append({                        # OBSERVAR: el resultado vuelve al contexto
                    "role": "tool",
                    "tool_call_id": call.id,
                    "content": str(resultado),
                })
            continue                                     # otra vuelta con la nueva info

        return respuesta.texto                            # sin tool_calls → el agente ha terminado
    return "Detenido: presupuesto de pasos agotado"
```

Fíjate en lo esencial: **la observación se reincorpora al contexto**. Esa realimentación es lo que permite al agente reaccionar a errores, leer salidas y ajustar el plan. Sin ese paso, no hay bucle; hay una cadena de pasos a ciegas.

### Por Qué el Bucle lo Cambia Todo

El bucle desbloquea capacidades que un disparo único no tiene:

- **Tareas largas y de muchos pasos.** "Arregla este bug" no se resuelve en una respuesta: hay que leer código, ejecutar tests, ver el fallo, editar, reejecutar. Cada vuelta del bucle es uno de esos pasos.
- **Autocorrección.** Si una acción falla (un test rojo, una excepción), el error entra como observación y el agente puede intentar otra cosa. El modelo "se entera" de sus errores.
- **Exploración.** El agente puede descubrir qué necesita sobre la marcha (qué ficheros existen, qué devuelve una API) en lugar de tenerlo todo predefinido.

Esto es exactamente lo que hacen los agentes de programación: giran un bucle de leer-editar-ejecutar-tests hasta que la tarea queda verde.

### Loop, Prompt y Context Engineering

Tres disciplinas que se solapan pero no son lo mismo:

| Disciplina | Qué optimizas | Pregunta central |
|---|---|---|
| **Prompt engineering** | El mensaje que envías | ¿Cómo le pido esto al modelo? |
| **Context engineering** | Qué información entra en la ventana en cada vuelta | ¿Qué necesita ver el modelo *ahora* y qué sobra? |
| **Loop engineering** | El bucle completo: tools, parada, verificación, operación | ¿Cómo hago que el ciclo entero sea fiable, barato y seguro? |

El loop engineering los **engloba**: dentro de un buen bucle hay buenos prompts y buen manejo del contexto, pero además hay decisiones de qué herramientas dar, cuándo parar, cómo verificar el resultado y cómo operar el agente en producción. Por eso es el marco que organiza esta sección.

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - Un LLM solo responde una vez; metido en un **bucle** con herramientas se convierte en un **agente** capaz de actuar, observar y corregirse.
> - El bucle tiene cuatro fases que se repiten: **percibir → pensar → actuar → observar**, hasta una **condición de parada** (tarea hecha, presupuesto agotado o guardarraíl).
> - La clave técnica es que **la observación se reincorpora al contexto**: esa realimentación es lo que da autocorrección y manejo de tareas largas.
> - El bucle desbloquea tareas de muchos pasos, autocorrección y exploración — justo lo que hacen los agentes de programación.
> - **Loop engineering engloba** al prompt engineering y al context engineering: ya no diseñas un mensaje, diseñas el ciclo entero.

### Para llevar a la práctica
- [ ] Implementa el esqueleto de bucle de esta página con dos herramientas tontas (sumar, leer fichero)
- [ ] Añade un `print` en cada fase para ver el ciclo percibir-pensar-actuar-observar en vivo
- [ ] Quítale al bucle el paso de "observar" (no devuelvas el resultado al contexto) y observa cómo se degrada
- [ ] Clasifica un problema tuyo con LLM: ¿necesita un prompt mejor o necesita un bucle?

### Recursos
- 🌐 anthropic.com/engineering/building-effective-agents — patrones de agentes y cuándo un bucle aporta
- 🌐 anthropic.com/engineering/effective-context-engineering-for-ai-agents — context engineering, la base del bucle
- 📖 *Documentación de la Claude API* — tool use y el bucle de agente (ver [[Desarrollo Profesional/IA/Páginas/04 - Agentes y MCP|IA 04]])

---
`#loop-engineering` `#agentes` `#bucle-agentico` `#ia` `#prompt-engineering`
