# 🐘 PostgreSQL 04 — Funciones Avanzadas

[[Desarrollo Profesional/PostgreSQL/PostgreSQL|⬅️ Volver a PostgreSQL]] | [[Desarrollo Profesional/PostgreSQL/Páginas/03 - Transacciones y MVCC|← 03]]

> [!abstract] Introducción
> PostgreSQL va mucho más allá del SQL estándar. Su soporte nativo de JSONB permite usarlo como base de datos de documentos. El particionado gestiona tablas de cientos de millones de filas. El full-text search elimina la necesidad de Elasticsearch para casos de uso moderados. Y extensiones como pgvector añaden búsqueda vectorial para aplicaciones de IA. Esta página cubre las características que hacen de PostgreSQL una plataforma completa.

## ¿De qué vamos a hablar?

JSONB y operadores JSON, full-text search, particionado de tablas, funciones almacenadas y las extensiones más útiles.

---

## El Concepto

### JSONB — PostgreSQL como Base de Datos de Documentos

JSONB almacena JSON en formato binario — permite índices GIN y operaciones eficientes:

```sql
-- Tabla con columna JSONB
CREATE TABLE eventos (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    tipo TEXT NOT NULL,
    payload JSONB NOT NULL,
    creado_en TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Insertar datos JSONB
INSERT INTO eventos (tipo, payload) VALUES
('pedido_creado', '{"pedido_id": "123", "cliente": {"nombre": "Ana", "email": "ana@test.com"}, "total": 89.99, "items": ["prod-1", "prod-2"]}'),
('pago_procesado', '{"pedido_id": "123", "metodo": "tarjeta", "exitoso": true}');

-- Operadores JSONB
-- -> devuelve JSON, ->> devuelve texto
SELECT payload -> 'cliente' AS cliente_json          -- retorna JSON
SELECT payload ->> 'tipo'                            -- retorna texto
SELECT payload -> 'cliente' ->> 'nombre' AS nombre   -- acceso anidado

-- Filtrar por valor JSONB
SELECT * FROM eventos
WHERE payload ->> 'pedido_id' = '123';

-- Operador @> (contiene) — muy eficiente con índice GIN
SELECT * FROM eventos
WHERE payload @> '{"cliente": {"email": "ana@test.com"}}';

-- Índice GIN para búsquedas en JSONB
CREATE INDEX idx_eventos_payload ON eventos USING GIN (payload);

-- Descomponer JSONB en columnas con jsonb_to_recordset
SELECT * FROM jsonb_to_recordset(
    '[{"nombre": "Ana", "edad": 30}, {"nombre": "Luis", "edad": 25}]'::jsonb
) AS t(nombre TEXT, edad INT);

-- Agregar/modificar campos JSONB
UPDATE eventos
SET payload = payload || '{"procesado": true}'::jsonb
WHERE tipo = 'pago_procesado';

-- Eliminar campo
UPDATE eventos
SET payload = payload - 'campo_obsoleto'
WHERE tipo = 'pedido_creado';

-- jsonb_build_object — construir JSONB dinámicamente
SELECT jsonb_build_object(
    'id', id,
    'tipo', tipo,
    'fecha', creado_en
) FROM eventos;
```

### Full-Text Search

```sql
-- Convertir texto a tsvector (documento de búsqueda)
SELECT to_tsvector('spanish', 'El usuario creó un nuevo pedido en la tienda online');
-- 'cre':3 'nuev':5 'onlin':9 'pedid':6 'tiend':8 'usuari':2

-- Convertir una consulta a tsquery
SELECT to_tsquery('spanish', 'pedido & online');     -- AND
SELECT to_tsquery('spanish', 'pedido | factura');    -- OR
SELECT plainto_tsquery('spanish', 'pedido online');  -- más natural, mismo resultado

-- Buscar
SELECT titulo, contenido
FROM articulos
WHERE to_tsvector('spanish', titulo || ' ' || contenido) @@ plainto_tsquery('spanish', 'kubernetes pods');

-- Índice para FTS (GIN es más rápido para queries, GiST para updates)
ALTER TABLE articulos ADD COLUMN fts_vector tsvector
    GENERATED ALWAYS AS (to_tsvector('spanish', titulo || ' ' || coalesce(contenido, ''))) STORED;

CREATE INDEX idx_articulos_fts ON articulos USING GIN (fts_vector);

-- Búsqueda con el índice generado
SELECT titulo, ts_rank(fts_vector, query) AS rank
FROM articulos, plainto_tsquery('spanish', 'kubernetes pods') query
WHERE fts_vector @@ query
ORDER BY rank DESC
LIMIT 10;

-- Resaltado de resultados (snippets)
SELECT titulo,
       ts_headline('spanish', contenido, plainto_tsquery('spanish', 'kubernetes'),
                   'MaxFragments=2, MaxWords=30, MinWords=15')
FROM articulos
WHERE fts_vector @@ plainto_tsquery('spanish', 'kubernetes');
```

### Particionado de Tablas

El particionado divide una tabla lógica en tablas físicas más pequeñas. Permite eliminar datos por partición (sin DELETE costoso), paralelizar queries y mejorar el rendimiento en tablas grandes:

```sql
-- Particionado por rango de fechas
CREATE TABLE metricas (
    id BIGSERIAL,
    servicio TEXT NOT NULL,
    metrica TEXT NOT NULL,
    valor DOUBLE PRECISION NOT NULL,
    timestamp TIMESTAMP WITH TIME ZONE NOT NULL
) PARTITION BY RANGE (timestamp);

-- Crear particiones por mes
CREATE TABLE metricas_2024_01 PARTITION OF metricas
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE metricas_2024_02 PARTITION OF metricas
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

-- Crear partición automáticamente (mejor con pg_partman extension)
-- El INSERT automáticamente va a la partición correcta
INSERT INTO metricas VALUES (DEFAULT, 'api-gateway', 'latencia_ms', 45.2, NOW());

-- Eliminar datos antiguos — MUY EFICIENTE (sin DELETE, solo DROP TABLE)
DROP TABLE metricas_2024_01;

-- Particionado por lista (para datos categóricos)
CREATE TABLE pedidos_por_pais (
    id UUID,
    pais TEXT NOT NULL,
    total NUMERIC(10,2)
) PARTITION BY LIST (pais);

CREATE TABLE pedidos_es PARTITION OF pedidos_por_pais FOR VALUES IN ('ES');
CREATE TABLE pedidos_mx PARTITION OF pedidos_por_pais FOR VALUES IN ('MX');
```

### Extensiones Esenciales

```sql
-- Ver extensiones instaladas
SELECT * FROM pg_extension;

-- pg_trgm — búsqueda de texto con similitud (LIKE sin prefijo)
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX idx_nombre_trgm ON usuarios USING GIN (nombre gin_trgm_ops);
SELECT nombre FROM usuarios WHERE nombre % 'garsia';  -- similitud fuzzy
SELECT nombre, similarity(nombre, 'garsia') FROM usuarios
WHERE nombre % 'garsia'
ORDER BY similarity(nombre, 'garsia') DESC;

-- pgcrypto — funciones criptográficas
CREATE EXTENSION IF NOT EXISTS pgcrypto;
-- Hash de contraseñas
SELECT crypt('mi_password', gen_salt('bf', 10)) AS hash_bcrypt;
-- Verificar
SELECT (crypt('mi_password', hash_almacenado) = hash_almacenado) AS valido;

-- uuid-ossp / pgcrypto para UUIDs
SELECT gen_random_uuid();  -- disponible desde PostgreSQL 13 sin extensión

-- pgvector — búsqueda vectorial para aplicaciones de IA
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE documentos (
    id BIGSERIAL PRIMARY KEY,
    contenido TEXT,
    embedding vector(1536)  -- dimensiones del modelo de embeddings
);

-- Índice para búsqueda aproximada de vecinos más cercanos
CREATE INDEX ON documentos USING ivfflat (embedding vector_cosine_ops)
    WITH (lists = 100);

-- Búsqueda por similitud coseno
SELECT contenido, 1 - (embedding <=> '[0.1, 0.2, ...]'::vector) AS similitud
FROM documentos
ORDER BY embedding <=> '[0.1, 0.2, ...]'::vector
LIMIT 5;
```

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - **JSONB** con índice GIN permite usar PostgreSQL como base de datos de documentos para datos semi-estructurados. El operador `@>` (contiene) es el más eficiente con el índice.
> - El **full-text search** nativo de PostgreSQL es suficiente para muchos casos de uso — elimina la necesidad de Elasticsearch en sistemas medianos.
> - El **particionado** es la herramienta correcta para tablas que crecen indefinidamente (logs, métricas, eventos). Permite eliminar datos antiguos con `DROP TABLE` en lugar de `DELETE`.
> - **pgvector** convierte PostgreSQL en un vector store — si ya usas Postgres, añade esta extensión antes de añadir una base de datos vectorial separada.

### Recursos
- 🌐 postgresql.org/docs/current/textsearch.html — documentación oficial de FTS
- 🌐 postgresql.org/docs/current/ddl-partitioning.html
- 🌐 github.com/pgvector/pgvector — extensión pgvector

---
`#postgresql` `#jsonb` `#full-text-search` `#particionado` `#pgvector`
