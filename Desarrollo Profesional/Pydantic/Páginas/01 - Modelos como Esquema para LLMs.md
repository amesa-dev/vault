# 🧱 Pydantic 01 — Modelos como Esquema para LLMs

[[Desarrollo Profesional/Pydantic/Pydantic|⬅️ Volver a Pydantic]] | [[Desarrollo Profesional/Pydantic/Páginas/02 - Salida Estructurada y Tool Calling|02 →]]

> [!abstract] Introducción
> Antes de usar Pydantic con LLMs hay que ver el modelo desde el ángulo correcto: no como un validador que corre *después*, sino como un **esquema que describe la forma de los datos** y que se traduce automáticamente a **JSON Schema**. Ese JSON Schema es justo el lenguaje que entienden las APIs de OpenAI, Anthropic o Google para forzar salidas estructuradas. Definir bien el modelo es, literalmente, definir el contrato con el LLM.

## ¿De qué vamos a hablar?

Los fundamentos de `BaseModel` con la mirada puesta en IA: tipos, `Field` con descripciones (que el LLM lee), validadores, y la generación de JSON Schema que conecta tu modelo con la API del LLM.

### Conceptos que vamos a cubrir
- `BaseModel` y tipado declarativo
- `Field`: descripciones, restricciones y por qué el LLM las usa
- Validadores de campo y de modelo
- `model_json_schema()`: el puente con las APIs de LLM

---

## El Concepto

### BaseModel: el esquema

Un modelo de Pydantic declara campos con tipos de Python. Pydantic valida, convierte (coerción) y expone el esquema.

```python
from pydantic import BaseModel

class Persona(BaseModel):
    nombre: str
    edad: int
    email: str

# Validación + coerción automáticas
p = Persona(nombre="Ana", edad="30", email="ana@x.com")  # "30" -> 30
print(p.edad)  # 30 (int)
```

### Field: descripciones que el LLM lee

Aquí está la diferencia clave en IA. `Field` permite añadir una **descripción** y **restricciones**. Cuando este modelo se convierte en JSON Schema y se envía al LLM, **el modelo lee esas descripciones** para saber qué poner en cada campo. Una buena descripción es, en la práctica, parte del prompt.

```python
from pydantic import BaseModel, Field

class Ticket(BaseModel):
    titulo: str = Field(description="Resumen corto del problema, máximo 8 palabras")
    prioridad: int = Field(ge=1, le=5, description="1 = trivial, 5 = crítico/caída total")
    categoria: str = Field(description="Una de: facturación, técnico, cuenta, otro")
    requiere_humano: bool = Field(description="True si el caso necesita un agente humano")
```

> [!tip] Las descripciones de `Field` son prompts
> En desarrollo con LLMs, escribir `Field(description=...)` es tan importante como el prompt principal. El modelo decide qué valor poner leyendo esa descripción. Sé específico: enumera valores válidos, da rangos, aclara el formato.

### Validadores: garantizar lo que el LLM no garantiza

Un LLM puede devolver algo plausible pero inválido. Los validadores son tu red de seguridad. `field_validator` valida un campo; `model_validator` valida relaciones entre campos.

```python
from pydantic import BaseModel, field_validator, model_validator
from typing_extensions import Self

class Reserva(BaseModel):
    dias: int
    precio_total: float

    @field_validator("dias")
    @classmethod
    def dias_positivos(cls, v: int) -> int:
        if v <= 0:
            raise ValueError("los días deben ser positivos")
        return v

    @model_validator(mode="after")
    def precio_coherente(self) -> Self:
        if self.precio_total < 0:
            raise ValueError("el precio no puede ser negativo")
        return self
```

Cuando el LLM produce algo que falla un validador, esa excepción se puede **devolver al modelo** para que reintente (lo veremos en [[Desarrollo Profesional/Pydantic/Páginas/02 - Salida Estructurada y Tool Calling|02]]).

### model_json_schema(): el puente con la API

Pydantic genera el JSON Schema de cualquier modelo. Ese documento es lo que las APIs de LLM usan para forzar la estructura.

```python
import json
print(json.dumps(Ticket.model_json_schema(), indent=2, ensure_ascii=False))
```

```json
{
  "properties": {
    "titulo": {"description": "Resumen corto del problema, máximo 8 palabras", "type": "string"},
    "prioridad": {"description": "1 = trivial, 5 = crítico/caída total", "maximum": 5, "minimum": 1, "type": "integer"},
    "categoria": {"description": "Una de: facturación, técnico, cuenta, otro", "type": "string"},
    "requiere_humano": {"description": "True si el caso necesita un agente humano", "type": "boolean"}
  },
  "required": ["titulo", "prioridad", "categoria", "requiere_humano"]
}
```

Las descripciones, los tipos y las restricciones (`minimum`, `maximum`) viajan al LLM. Por eso un modelo bien anotado produce salidas mucho más fiables.

> [!info] Enums para cerrar el espacio de respuestas
> Para campos con valores fijos, usa `Literal` o `Enum`: el JSON Schema incluirá `enum: [...]` y el modelo se ceñirá a esas opciones, eliminando variantes inventadas.
> ```python
> from typing import Literal
> categoria: Literal["facturacion", "tecnico", "cuenta", "otro"]
> ```

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - En IA, un modelo de Pydantic es un **esquema**: define el **contrato** de lo que el LLM debe devolver, no solo un validador posterior.
> - **`Field(description=...)` es parte del prompt**: el LLM lee esas descripciones para rellenar cada campo; sé específico con valores, rangos y formato.
> - Los **validadores** (`field_validator`, `model_validator`) son la red de seguridad ante salidas plausibles pero inválidas, y permiten reintentos guiados.
> - **`model_json_schema()`** traduce el modelo a JSON Schema, el lenguaje que las APIs de LLM usan para forzar la estructura.
> - Usa **`Literal`/`Enum`** para cerrar el conjunto de valores válidos.

### Para llevar a la práctica
- [ ] Diseña un modelo `Ticket` con `Field` descriptivos y míralo con `model_json_schema()`
- [ ] Añade un `field_validator` que rechace un valor que un LLM podría inventar
- [ ] Convierte un campo de texto libre en un `Literal` y observa el `enum` en el JSON Schema

### Recursos
- 🌐 docs.pydantic.dev/latest/concepts/models — modelos en Pydantic v2
- 🌐 docs.pydantic.dev/latest/concepts/json_schema — generación de JSON Schema
- 🌐 docs.pydantic.dev/latest/concepts/validators — validadores

---
`#pydantic` `#ia` `#json-schema` `#validacion`
