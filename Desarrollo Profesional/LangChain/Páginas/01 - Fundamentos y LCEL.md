# 🦜 LangChain 01 — Fundamentos y LCEL

[[Desarrollo Profesional/LangChain/LangChain|⬅️ Volver a LangChain]] | [[Desarrollo Profesional/LangChain/Páginas/02 - RAG con LangChain|02 →]]

> [!abstract] Introducción
> El corazón de LangChain moderno es **LCEL** (LangChain Expression Language): un modelo de composición donde cada pieza —prompt, modelo, parser— es un *Runnable* y se encadenan con el operador `|`, igual que un pipe de Unix. Esto hace que los flujos sean declarativos, soporten streaming y ejecución asíncrona por defecto, y sean fáciles de leer. Entender los componentes básicos y cómo se conectan es el 80% de LangChain.

## ¿De qué vamos a hablar?

Los componentes fundamentales (prompt templates, chat models, output parsers), la interfaz Runnable y el operador `|`, y cómo construir una cadena completa y tipada con salida estructurada.

### Conceptos que vamos a cubrir
- Chat models y mensajes
- Prompt templates
- Output parsers (incluido el structured output con Pydantic)
- LCEL: Runnables y el operador `|`

---

## El Concepto

### Chat models y prompts

Un *chat model* abstrae al proveedor: el mismo código sirve para Anthropic, OpenAI, etc. Un *prompt template* construye el prompt a partir de variables.

```python
from langchain_anthropic import ChatAnthropic
from langchain_core.prompts import ChatPromptTemplate

modelo = ChatAnthropic(model="claude-sonnet-4-6", temperature=0)

prompt = ChatPromptTemplate.from_messages([
    ("system", "Eres un traductor profesional al {idioma}."),
    ("human", "{texto}"),
])
```

### Output parsers

El modelo devuelve un mensaje. Un *parser* lo transforma en lo que tu código necesita: texto plano, JSON o un modelo Pydantic.

```python
from langchain_core.output_parsers import StrOutputParser

parser = StrOutputParser()  # extrae el texto del mensaje
```

### LCEL: encadenar con `|`

Aquí brilla LangChain. Cada componente es un **Runnable**, y `|` los conecta: la salida de uno es la entrada del siguiente. Una cadena completa se lee de un vistazo.

```python
cadena = prompt | modelo | parser

resultado = cadena.invoke({"idioma": "francés", "texto": "Buenos días"})
print(resultado)  # "Bonjour"
```

El flujo es: el dict de entrada rellena el `prompt` → el `prompt` produce mensajes que van al `modelo` → el `modelo` produce un mensaje que el `parser` convierte en string.

```
{idioma, texto} ──▶ prompt ──▶ mensajes ──▶ modelo ──▶ AIMessage ──▶ parser ──▶ str
                      └──────────────── cadena = prompt | modelo | parser ──────────┘
```

### La interfaz Runnable: invoke, stream, batch, async

Todo Runnable comparte la misma API, gratis para cualquier cadena que construyas:

```python
cadena.invoke({...})              # una ejecución
cadena.batch([{...}, {...}])      # varias en paralelo
for trozo in cadena.stream({...}): # streaming token a token
    print(trozo, end="")
await cadena.ainvoke({...})       # versión asíncrona
```

### Salida estructurada con Pydantic

LangChain se apoya en [[Desarrollo Profesional/Pydantic/Pydantic|Pydantic]] para salida tipada. `with_structured_output` hace que el modelo devuelva directamente una instancia validada.

```python
from pydantic import BaseModel, Field

class Receta(BaseModel):
    titulo: str = Field(description="Nombre del plato")
    ingredientes: list[str]
    minutos: int = Field(description="Tiempo total de preparación")

modelo_estructurado = ChatAnthropic(model="claude-sonnet-4-6").with_structured_output(Receta)
receta = modelo_estructurado.invoke("Dame una receta rápida de tortilla")
print(receta.titulo, receta.minutos)  # objeto Receta validado
```

> [!tip] LCEL te da streaming y async sin esfuerzo
> Como cada cadena es un Runnable, obtienes `stream`, `batch` y `ainvoke` sin escribir nada extra. Esto es lo que hace LCEL superior a encadenar funciones a mano: la composición trae las capacidades de ejecución de regalo.

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - Los **chat models** abstraen el proveedor; los **prompt templates** construyen el prompt desde variables.
> - Los **output parsers** transforman el mensaje del modelo en texto, JSON o un objeto Pydantic.
> - **LCEL** encadena componentes con `|`: cada pieza es un **Runnable** y la salida de una alimenta la siguiente.
> - Toda cadena hereda la misma API: **`invoke`, `batch`, `stream`, `ainvoke`** sin código adicional.
> - **`with_structured_output(Modelo)`** devuelve instancias Pydantic validadas, conectando con la fiabilidad de tipos.

### Para llevar a la práctica
- [ ] Construye una cadena `prompt | modelo | parser` y pruébala con `invoke`
- [ ] Cambia el `parser` por `with_structured_output` y un modelo Pydantic
- [ ] Usa `stream` sobre la misma cadena y observa la salida token a token

### Recursos
- 🌐 python.langchain.com/docs/concepts/lcel — LCEL en detalle
- 🌐 python.langchain.com/docs/concepts/runnables — la interfaz Runnable
- 🌐 python.langchain.com/docs/concepts/structured_outputs — salida estructurada

---
`#langchain` `#lcel` `#ia` `#runnables`
