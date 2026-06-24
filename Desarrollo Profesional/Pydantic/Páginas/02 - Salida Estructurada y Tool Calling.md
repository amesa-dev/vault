# 🧱 Pydantic 02 — Salida Estructurada y Tool Calling

[[Desarrollo Profesional/Pydantic/Pydantic|⬅️ Volver a Pydantic]] | [[Desarrollo Profesional/Pydantic/Páginas/01 - Modelos como Esquema para LLMs|← 01]] | [[Desarrollo Profesional/Pydantic/Páginas/03 - PydanticAI y Agentes|03 →]]

> [!abstract] Introducción
> Aquí está el uso estrella de Pydantic en IA: convertir la respuesta de un LLM —texto— en un **objeto Python validado**. Dos mecanismos lo hacen posible: la **salida estructurada** (le pides al modelo que responda con la forma de tu esquema) y el **tool/function calling** (el modelo decide llamar a una función cuyos argumentos defines con Pydantic). La librería **Instructor** une ambos en una API mínima y añade **reintentos automáticos** cuando la validación falla.

## ¿De qué vamos a hablar?

Cómo forzar salida estructurada con las APIs nativas y con Instructor, cómo definir herramientas (tools) con argumentos tipados por Pydantic, y el bucle de reintento por validación que hace todo esto fiable.

### Conceptos que vamos a cubrir
- Salida estructurada con Anthropic/OpenAI usando un esquema Pydantic
- Instructor: `response_model` y reintentos automáticos
- Tool calling: argumentos de herramienta como modelos Pydantic
- El bucle validar → reintentar

### El problema que resolvemos

```python
# SIN estructura — frágil y propenso a romperse
respuesta = llm("Extrae nombre y edad de: 'Ana tiene 30 años'")
# -> "El nombre es Ana y la edad 30."  ¿Cómo parseas esto de forma fiable? Con regex frágiles.

# CON Pydantic — el LLM devuelve directamente un objeto validado
persona = extraer(texto, modelo=Persona)  # -> Persona(nombre="Ana", edad=30)
```

---

## El Concepto

### Salida estructurada con la API nativa

Las APIs modernas aceptan un JSON Schema para forzar la forma de la respuesta. Con el modelo de Pydantic ya tienes ese schema.

```python
import anthropic
from pydantic import BaseModel, Field

class Persona(BaseModel):
    nombre: str = Field(description="Nombre de pila")
    edad: int = Field(description="Edad en años")

client = anthropic.Anthropic()
tool = {
    "name": "registrar_persona",
    "description": "Registra los datos extraídos de una persona",
    "input_schema": Persona.model_json_schema(),  # el esquema viaja al modelo
}

msg = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=512,
    tools=[tool],
    tool_choice={"type": "tool", "name": "registrar_persona"},
    messages=[{"role": "user", "content": "Ana tiene 30 años"}],
)

# El modelo devuelve los argumentos; Pydantic los valida
datos = next(b.input for b in msg.content if b.type == "tool_use")
persona = Persona.model_validate(datos)  # -> Persona(nombre="Ana", edad=30)
```

### Instructor: la vía corta + reintentos

**Instructor** parchea el cliente del proveedor y añade `response_model`. Tú pides un modelo Pydantic y recibes una instancia ya validada. Si la validación falla, **reenvía el error al LLM y reintenta** automáticamente.

```python
import instructor
from anthropic import Anthropic
from pydantic import BaseModel, Field, field_validator

class Persona(BaseModel):
    nombre: str
    edad: int = Field(description="Edad en años, debe ser positiva")

    @field_validator("edad")
    @classmethod
    def positiva(cls, v: int) -> int:
        if v <= 0:
            raise ValueError("la edad debe ser positiva")
        return v

client = instructor.from_anthropic(Anthropic())

persona = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=512,
    response_model=Persona,      # <- recibes una Persona, no texto
    max_retries=3,               # <- reintenta si la validación falla
    messages=[{"role": "user", "content": "Ana tiene 30 años"}],
)
print(persona.nombre, persona.edad)  # Ana 30
```

### Tool calling con argumentos tipados

En tool calling, el LLM **decide** llamar a una función y produce sus argumentos. Defines esos argumentos con Pydantic: el modelo genera datos conformes y tú los validas antes de ejecutar la función real.

```python
from pydantic import BaseModel, Field

class BuscarVuelosArgs(BaseModel):
    origen: str = Field(description="Código IATA del aeropuerto de origen, p. ej. MAD")
    destino: str = Field(description="Código IATA del destino, p. ej. JFK")
    fecha: str = Field(description="Fecha en formato ISO YYYY-MM-DD")

def buscar_vuelos(args: BuscarVuelosArgs) -> list[dict]:
    # args ya viene validado: origen/destino/fecha con la forma correcta
    ...

# El esquema de la tool se obtiene igual: BuscarVuelosArgs.model_json_schema()
```

### El bucle validar → reintentar

Lo que hace robusto todo esto es el ciclo: el LLM responde, Pydantic valida, y si falla, el **mensaje de error de validación** vuelve al modelo como contexto para que se corrija.

```
Prompt + JSON Schema ──▶ LLM ──▶ JSON candidato
                                     │
                          Pydantic.model_validate()
                                     │
                   ┌── válido ──▶ objeto Python ✅
                   │
                   └── inválido ──▶ error ──▶ se reenvía al LLM ──▶ reintenta
```

Sin este bucle, una sola alucinación rompe tu pipeline. Con él, el sistema se autocorrige. Instructor (y PydanticAI, en [[Desarrollo Profesional/Pydantic/Páginas/03 - PydanticAI y Agentes|03]]) lo implementan por ti.

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - Pydantic convierte la salida de un LLM (texto) en un **objeto Python validado**, vía salida estructurada o tool calling.
> - Con la **API nativa** pasas `model_json_schema()` como esquema de una tool y validas el resultado con `model_validate()`.
> - **Instructor** simplifica todo con `response_model=TuModelo` y **`max_retries`**: recibes la instancia ya validada.
> - En **tool calling**, los argumentos de cada herramienta se definen como modelos Pydantic: el LLM genera datos conformes.
> - La clave de la fiabilidad es el **bucle validar → reintentar**: el error de validación se reenvía al modelo para que se corrija.

### Para llevar a la práctica
- [ ] Extrae datos estructurados de un texto con Instructor y un `response_model`
- [ ] Añade un `field_validator` y fuerza un fallo para ver el reintento automático
- [ ] Define los argumentos de una tool con Pydantic y valida la llamada antes de ejecutarla

### Recursos
- 🌐 python.useinstructor.com — Instructor: structured outputs y reintentos
- 🌐 docs.anthropic.com/en/docs/build-with-claude/tool-use — tool use en la API de Claude
- 🌐 docs.pydantic.dev/latest/concepts/json_schema — el JSON Schema que viaja al LLM

---
`#pydantic` `#ia` `#structured-output` `#tool-calling`
