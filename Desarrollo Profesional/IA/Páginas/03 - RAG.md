# 🤖 IA 03 — RAG (Retrieval-Augmented Generation)

[[Desarrollo Profesional/IA/IA|⬅️ Volver a IA]] | [[Desarrollo Profesional/IA/Páginas/02 - Prompt Engineering|← 02]] | [[Desarrollo Profesional/IA/Páginas/04 - Agentes y MCP|04 →]]

> [!abstract] Introducción
> RAG resuelve el problema más inmediato de los LLMs en producción: saben muchas cosas, pero no saben las cosas específicas de tu negocio, tus documentos internos o los datos que cambian después de su fecha de corte. RAG combina búsqueda semántica (retrieval) con generación de texto (generation) para que el modelo responda sobre documentos que nunca vio en su entrenamiento.

## ¿De qué vamos a hablar?

El patrón RAG completo: desde la ingesta y chunking de documentos hasta la búsqueda vectorial y la generación de respuestas, con técnicas avanzadas de reranking y filtrado.

### Conceptos que vamos a cubrir
- El problema que resuelve RAG
- Pipeline de ingesta: chunking y embedding
- Vector stores: bases de datos vectoriales
- El pipeline de query: retrieve, rerank, generate
- Técnicas avanzadas: híbrido, reranking, filtrado por metadata

---

## El Concepto

### El Problema que Resuelve RAG

```
❌ Sin RAG:
Usuario: "¿Cuál es la política de devoluciones de mi empresa?"
LLM: "No tengo acceso a los documentos específicos de tu empresa..."

✅ Con RAG:
1. Usuario hace la pregunta
2. Sistema busca los fragmentos relevantes en la BD de documentos
3. LLM recibe la pregunta + los fragmentos relevantes
4. LLM genera una respuesta basada en los documentos reales
```

### Pipeline de Ingesta (Offline)

```python
from pathlib import Path
import anthropic
from openai import OpenAI

# Paso 1: Cargar documentos
def cargar_documentos(directorio: str) -> list[dict]:
    docs = []
    for path in Path(directorio).rglob("*.txt"):
        docs.append({
            "contenido": path.read_text(encoding="utf-8"),
            "fuente": str(path),
            "titulo": path.stem
        })
    return docs

# Paso 2: Chunking — dividir documentos en fragmentos
def chunking_fijo(texto: str, tamaño: int = 500, solapamiento: int = 100) -> list[str]:
    """Chunking simple por caracteres con solapamiento"""
    chunks = []
    inicio = 0
    while inicio < len(texto):
        fin = min(inicio + tamaño, len(texto))
        # Ajustar al final de la última frase completa
        if fin < len(texto):
            ultimo_punto = texto.rfind('.', inicio, fin)
            if ultimo_punto > inicio:
                fin = ultimo_punto + 1
        chunks.append(texto[inicio:fin].strip())
        inicio = fin - solapamiento
    return [c for c in chunks if c]

def chunking_por_parrafos(texto: str, max_tokens: int = 300) -> list[str]:
    """Mejor para documentos con párrafos bien definidos"""
    parrafos = [p.strip() for p in texto.split('\n\n') if p.strip()]
    chunks = []
    chunk_actual = ""

    for parrafo in parrafos:
        # Estimación: ~4 chars por token
        if len(chunk_actual) + len(parrafo) < max_tokens * 4:
            chunk_actual += parrafo + "\n\n"
        else:
            if chunk_actual:
                chunks.append(chunk_actual.strip())
            chunk_actual = parrafo + "\n\n"

    if chunk_actual:
        chunks.append(chunk_actual.strip())
    return chunks

# Paso 3: Generar embeddings
client_oai = OpenAI()

def generar_embeddings(textos: list[str]) -> list[list[float]]:
    response = client_oai.embeddings.create(
        input=textos,
        model="text-embedding-3-small"  # 1536 dimensiones, balance coste/calidad
        # model="text-embedding-3-large"  # 3072 dimensiones, mejor calidad
    )
    return [item.embedding for item in response.data]

# Pipeline completo de ingesta
def indexar_documentos(directorio: str, vector_store) -> None:
    documentos = cargar_documentos(directorio)

    for doc in documentos:
        chunks = chunking_por_parrafos(doc["contenido"])
        embeddings = generar_embeddings(chunks)

        for chunk, embedding in zip(chunks, embeddings):
            vector_store.insertar(
                texto=chunk,
                embedding=embedding,
                metadata={"fuente": doc["fuente"], "titulo": doc["titulo"]}
            )
```

### Vector Stores

```python
# pgvector — PostgreSQL con extensión de vectores (para proyectos que ya usan Postgres)
import asyncpg
import numpy as np

async def setup_pgvector(conn):
    await conn.execute("CREATE EXTENSION IF NOT EXISTS vector")
    await conn.execute("""
        CREATE TABLE IF NOT EXISTS embeddings (
            id SERIAL PRIMARY KEY,
            contenido TEXT NOT NULL,
            embedding vector(1536),
            fuente TEXT,
            titulo TEXT,
            creado_en TIMESTAMP DEFAULT NOW()
        )
    """)
    # Índice IVFFlat para búsqueda aproximada (más rápido, menos preciso)
    await conn.execute("""
        CREATE INDEX IF NOT EXISTS idx_embeddings_cosine
        ON embeddings USING ivfflat (embedding vector_cosine_ops)
        WITH (lists = 100)
    """)

async def insertar_embedding(conn, texto: str, embedding: list[float], metadata: dict):
    await conn.execute(
        "INSERT INTO embeddings (contenido, embedding, fuente, titulo) VALUES ($1, $2, $3, $4)",
        texto, embedding, metadata["fuente"], metadata["titulo"]
    )

async def buscar_similares(conn, query_embedding: list[float], k: int = 5) -> list[dict]:
    filas = await conn.fetch(
        """
        SELECT contenido, fuente, titulo,
               1 - (embedding <=> $1::vector) AS similitud
        FROM embeddings
        ORDER BY embedding <=> $1::vector
        LIMIT $2
        """,
        query_embedding, k
    )
    return [dict(f) for f in filas]

# ChromaDB — si no tienes Postgres, la opción más simple para empezar
import chromadb

client_chroma = chromadb.PersistentClient(path="./chroma_db")
coleccion = client_chroma.get_or_create_collection("mis_documentos")

coleccion.add(
    documents=["texto del chunk"],
    embeddings=[[0.1, 0.2, ...]],
    metadatas=[{"fuente": "doc.pdf"}],
    ids=["chunk-001"]
)

resultados = coleccion.query(
    query_embeddings=[query_embedding],
    n_results=5,
    where={"fuente": "politica-devoluciones.pdf"}  # filtro por metadata
)
```

### Pipeline de Query (Online)

```python
import anthropic

client = anthropic.Anthropic()

async def rag_query(pregunta: str, conn) -> str:
    # Paso 1: Generar embedding de la pregunta
    query_embedding = generar_embeddings([pregunta])[0]

    # Paso 2: Recuperar chunks relevantes
    chunks_relevantes = await buscar_similares(conn, query_embedding, k=5)

    # Paso 3: Filtrar por similitud mínima
    chunks_relevantes = [c for c in chunks_relevantes if c["similitud"] > 0.7]

    if not chunks_relevantes:
        return "No encontré información relevante en los documentos disponibles."

    # Paso 4: Construir contexto
    contexto = "\n\n---\n\n".join([
        f"Fuente: {c['titulo']}\n{c['contenido']}"
        for c in chunks_relevantes
    ])

    # Paso 5: Generar respuesta
    respuesta = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=1024,
        system="""Eres un asistente que responde preguntas basándose únicamente en los documentos proporcionados.
Si la respuesta no está en los documentos, dilo explícitamente.
Cita siempre la fuente cuando uses información de los documentos.""",
        messages=[{
            "role": "user",
            "content": f"""Contexto de los documentos:

{contexto}

---

Pregunta: {pregunta}"""
        }]
    )

    return respuesta.content[0].text
```

### Técnicas Avanzadas

**Reranking** — los embeddings de búsqueda no siempre ordenan perfectamente por relevancia. Un reranker (modelo cross-encoder) reordena los chunks recuperados:

```python
from sentence_transformers import CrossEncoder

reranker = CrossEncoder("cross-encoder/ms-marco-MiniLM-L-6-v2")

def rerankar(pregunta: str, chunks: list[dict], top_k: int = 3) -> list[dict]:
    pares = [(pregunta, c["contenido"]) for c in chunks]
    scores = reranker.predict(pares)
    chunks_con_score = sorted(zip(chunks, scores), key=lambda x: x[1], reverse=True)
    return [c for c, _ in chunks_con_score[:top_k]]
```

**Búsqueda híbrida** — combina búsqueda semántica (vectores) con búsqueda léxica (BM25/full-text):

```sql
-- En PostgreSQL, combinar vector search con full-text search
SELECT
    id,
    contenido,
    titulo,
    -- Score semántico
    1 - (embedding <=> $1::vector) AS score_semantico,
    -- Score léxico (full-text search)
    ts_rank(to_tsvector('spanish', contenido), plainto_tsquery('spanish', $2)) AS score_lexico
FROM embeddings
WHERE
    1 - (embedding <=> $1::vector) > 0.5
    OR to_tsvector('spanish', contenido) @@ plainto_tsquery('spanish', $2)
ORDER BY
    -- RRF (Reciprocal Rank Fusion) para combinar ambos scores
    (1.0 / (60 + score_semantico)) + (1.0 / (60 + score_lexico)) DESC
LIMIT 10;
```

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - RAG = **Retrieval** (búsqueda de documentos relevantes) + **Augmented Generation** (LLM genera usando esos documentos). Resuelve las limitaciones de la context window y el knowledge cutoff.
> - El **chunking** importa mucho: chunks demasiado pequeños pierden contexto; demasiado grandes incluyen información irrelevante. ~300-500 tokens con solapamiento es un buen punto de partida.
> - Usa **pgvector** si ya tienes PostgreSQL — evita añadir otra base de datos al stack. ChromaDB para prototipos rápidos.
> - El **reranker** (cross-encoder) mejora la relevancia de los chunks recuperados por el vector search. Vale la pena añadirlo cuando la calidad de respuesta no es suficiente.
> - La **búsqueda híbrida** (vector + BM25) supera a la búsqueda solo vectorial en la mayoría de benchmarks.

### Recursos
- 📄 *Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks* — Lewis et al., 2020
- 🌐 github.com/pgvector/pgvector — extensión de vectores para PostgreSQL
- 🌐 python.langchain.com/docs/modules/data_connection — LangChain para RAG
- 🌐 llamaindex.ai — framework alternativo para RAG

---
`#ia` `#rag` `#embeddings` `#vector-store` `#retrieval`
