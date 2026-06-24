# 🦜 LangChain 02 — RAG con LangChain

[[Desarrollo Profesional/LangChain/LangChain|⬅️ Volver a LangChain]] | [[Desarrollo Profesional/LangChain/Páginas/01 - Fundamentos y LCEL|← 01]] | [[Desarrollo Profesional/LangChain/Páginas/03 - Agentes y LangGraph|03 →]]

> [!abstract] Introducción
> **RAG** (Retrieval-Augmented Generation) es el patrón que permite a un LLM responder sobre **tus datos** —documentos, una base de conocimiento, un PDF— sin reentrenarlo. La idea: recuperar los fragmentos relevantes y pasarlos al modelo como contexto. LangChain ofrece todas las piezas estandarizadas: cargar documentos, trocearlos, vectorizarlos, guardarlos y recuperarlos. Esta página monta un pipeline RAG completo de principio a fin.

## ¿De qué vamos a hablar?

El pipeline RAG paso a paso con componentes de LangChain: carga, troceado (chunking), embeddings, vector store, retriever, y la cadena final que une recuperación y generación.

### Conceptos que vamos a cubrir
- Document loaders y text splitters
- Embeddings y vector stores
- Retrievers
- La cadena RAG completa con LCEL

### Relación con la teoría

Los fundamentos conceptuales de RAG (chunking, reranking, vector stores) están en [[Desarrollo Profesional/IA/Páginas/03 - RAG|IA · 03 — RAG]]. Aquí los implementamos con LangChain.

---

## El Concepto

### 1. Cargar y trocear

Primero se cargan los documentos y se parten en fragmentos (*chunks*) manejables. El troceado importa: trozos demasiado grandes diluyen la relevancia; demasiado pequeños pierden contexto.

```python
from langchain_community.document_loaders import TextLoader
from langchain_text_splitters import RecursiveCharacterTextSplitter

docs = TextLoader("base_conocimiento.md", encoding="utf-8").load()

splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000,      # caracteres por fragmento
    chunk_overlap=150,    # solape para no cortar ideas a la mitad
)
fragmentos = splitter.split_documents(docs)
```

### 2. Embeddings y vector store

Cada fragmento se convierte en un **vector** (embedding) y se guarda en un *vector store* que permite búsqueda por similitud semántica.

```python
from langchain_openai import OpenAIEmbeddings
from langchain_community.vectorstores import Chroma

embeddings = OpenAIEmbeddings(model="text-embedding-3-small")
vector_store = Chroma.from_documents(fragmentos, embeddings)
```

### 3. Retriever

El *retriever* es la interfaz de búsqueda: dada una pregunta, devuelve los `k` fragmentos más relevantes.

```python
retriever = vector_store.as_retriever(search_kwargs={"k": 4})
relevantes = retriever.invoke("¿Cómo se configura el despliegue?")
```

### 4. La cadena RAG completa

Ahora se une todo con LCEL: la pregunta va al retriever (para el contexto) y, en paralelo, se pasa tal cual; ambos rellenan el prompt, que va al modelo.

```python
from langchain_anthropic import ChatAnthropic
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnablePassthrough

prompt = ChatPromptTemplate.from_template(
    "Responde SOLO con el contexto. Si no está, di que no lo sabes.\n\n"
    "Contexto:\n{context}\n\nPregunta: {question}"
)
modelo = ChatAnthropic(model="claude-sonnet-4-6", temperature=0)

def formatear(docs) -> str:
    return "\n\n".join(d.page_content for d in docs)

cadena_rag = (
    {"context": retriever | formatear, "question": RunnablePassthrough()}
    | prompt
    | modelo
    | StrOutputParser()
)

print(cadena_rag.invoke("¿Cómo se configura el despliegue?"))
```

```
Pregunta ──┬──▶ retriever ──▶ fragmentos ──▶ formatear ──▶ {context}
           │                                                   ├──▶ prompt ──▶ modelo ──▶ respuesta
           └──────────────────────────────▶ {question} ───────┘
```

> [!tip] La calidad del RAG vive en la recuperación
> Si el sistema responde mal, casi siempre el problema está *antes* del LLM: chunking inadecuado, `k` mal elegido o falta de **reranking**. Ajusta `chunk_size`/`overlap`, sube `k` y añade un reranker (Cohere, bge-reranker) antes de tocar el prompt o el modelo. Mide con [[Desarrollo Profesional/Langfuse/Langfuse|Langfuse]].

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - **RAG** da al LLM acceso a tus datos recuperando fragmentos relevantes y pasándolos como contexto, sin reentrenar.
> - El pipeline: **cargar → trocear (chunking) → embeddings → vector store → retriever → generación**.
> - El **chunking** (tamaño y solape) y el **`k`** del retriever son las palancas que más afectan a la calidad.
> - La **cadena RAG** con LCEL une recuperación (`retriever | formatear`) y pregunta (`RunnablePassthrough`) en el prompt.
> - Si responde mal, **el fallo suele estar en la recuperación**, no en el modelo: ajusta chunking, `k` y añade reranking.

### Para llevar a la práctica
- [ ] Monta un RAG sobre un documento propio y haz preguntas sobre su contenido
- [ ] Experimenta con `chunk_size`/`chunk_overlap` y observa cómo cambian las respuestas
- [ ] Sube y baja `k` en el retriever y compara precisión vs ruido en el contexto

### Recursos
- 🌐 python.langchain.com/docs/tutorials/rag — tutorial RAG oficial
- 🌐 python.langchain.com/docs/concepts/text_splitters — estrategias de troceado
- 🌐 python.langchain.com/docs/concepts/retrievers — retrievers

---
`#langchain` `#rag` `#ia` `#embeddings`
