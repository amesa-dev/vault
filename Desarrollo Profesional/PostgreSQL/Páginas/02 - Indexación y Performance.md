# 🐘 PostgreSQL 02 — Indexación y Performance

[[Desarrollo Profesional/PostgreSQL/PostgreSQL|⬅️ Volver a PostgreSQL]] | [[Desarrollo Profesional/PostgreSQL/Páginas/01 - Fundamentos|← 01]] | [[Desarrollo Profesional/PostgreSQL/Páginas/03 - Transacciones y MVCC|03 →]]

> [!abstract] Introducción
> La indexación es la diferencia entre una consulta que tarda 50ms y una que tarda 30 segundos en una tabla de 10 millones de filas. Pero los índices tienen coste: ralentizan los escrituras, ocupan disco y el planner puede elegir el equivocado. Esta página cubre los tipos de índice de PostgreSQL, cómo leer un plan de ejecución y las técnicas de optimización más efectivas.

## ¿De qué vamos a hablar?

Los tipos de índice de PostgreSQL, el plan de ejecución con EXPLAIN ANALYZE y las técnicas de optimización de consultas.

### Conceptos que vamos a cubrir
- Tipos de índice: B-Tree, GIN, GiST, Hash, BRIN
- EXPLAIN ANALYZE: leer el plan de ejecución
- Técnicas de optimización: índices parciales, índices de expresión, índices compuestos
- Patrones de consulta problemáticos

---

## El Concepto

### Tipos de Índice

**B-Tree** — el índice por defecto. Para comparaciones de igualdad y rango (`=`, `<`, `>`, `BETWEEN`, `LIKE 'prefijo%'`):

```sql
-- Índice simple — para búsquedas por email
CREATE INDEX idx_usuarios_email ON usuarios (email);

-- Índice único — equivalente a constraint UNIQUE con índice
CREATE UNIQUE INDEX idx_usuarios_email_unique ON usuarios (email);

-- Índice compuesto — para consultas con múltiples condiciones
-- El orden importa: PostgreSQL puede usar el índice si la consulta usa
-- las columnas desde la izquierda (email solo, o email+estado, pero no estado solo)
CREATE INDEX idx_pedidos_usuario_estado ON pedidos (usuario_id, estado);

-- Índice descending — para ORDER BY DESC frecuente
CREATE INDEX idx_pedidos_fecha_desc ON pedidos (creado_en DESC);
```

**Índice Parcial** — indexa solo las filas que cumplen una condición. Mucho más pequeño y rápido para consultas sobre un subconjunto de datos:

```sql
-- Solo indexa pedidos activos (los cancelados raramente se consultan)
CREATE INDEX idx_pedidos_activos ON pedidos (creado_en)
    WHERE estado IN ('pendiente', 'pagado', 'enviado');

-- Index for soft deletes — consultas casi siempre excluyen eliminados
CREATE INDEX idx_usuarios_activos ON usuarios (email)
    WHERE eliminado_en IS NULL;
```

**Índice de Expresión** — indexa el resultado de una función:

```sql
-- Para búsquedas case-insensitive
CREATE INDEX idx_usuarios_email_lower ON usuarios (LOWER(email));

-- Ahora esta consulta usa el índice
SELECT * FROM usuarios WHERE LOWER(email) = LOWER('ANA@EJEMPLO.COM');

-- Para extraer año de un timestamp
CREATE INDEX idx_pedidos_año ON pedidos (EXTRACT(YEAR FROM creado_en));
```

**GIN (Generalized Inverted Index)** — para arrays, JSONB y full-text search. Excelente para "¿contiene X?":

```sql
-- Para búsquedas en arrays
CREATE INDEX idx_articulos_tags ON articulos USING GIN (tags);
SELECT * FROM articulos WHERE tags @> ARRAY['python', 'backend'];

-- Para búsquedas en JSONB
CREATE INDEX idx_pedidos_metadata ON pedidos USING GIN (metadata);
SELECT * FROM pedidos WHERE metadata @> '{"pais": "ES"}';

-- Para full-text search
CREATE INDEX idx_articulos_fts ON articulos USING GIN (to_tsvector('spanish', titulo || ' ' || contenido));
SELECT * FROM articulos WHERE to_tsvector('spanish', titulo || ' ' || contenido) @@ to_tsquery('spanish', 'python & kubernetes');
```

### EXPLAIN ANALYZE — Leer el Plan de Ejecución

```sql
-- EXPLAIN — muestra el plan estimado
EXPLAIN SELECT * FROM pedidos WHERE usuario_id = '550e8400-e29b-41d4-a716-446655440000';

-- EXPLAIN ANALYZE — ejecuta la consulta y muestra plan real vs estimado
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT u.nombre, COUNT(p.id) AS total_pedidos
FROM usuarios u
LEFT JOIN pedidos p ON u.id = p.usuario_id
WHERE u.activo = true
GROUP BY u.id, u.nombre
ORDER BY total_pedidos DESC
LIMIT 10;
```

**Cómo leer el plan:**

```
Sort  (cost=150.25..150.30 rows=10 width=48) (actual time=5.234..5.237 rows=10 loops=1)
  Sort Key: (count(p.id)) DESC
  Sort Method: top-N heapsort  Memory: 25kB
  ->  HashAggregate  (cost=148.50..149.50 rows=100 width=48) (actual time=5.100..5.180 rows=100 loops=1)
        ->  Hash Left Join  (cost=30.00..135.00 rows=2700 width=24) (actual time=0.450..3.900 rows=2700 loops=1)
              Hash Cond: (p.usuario_id = u.id)
              ->  Seq Scan on pedidos p  (cost=0.00..85.00 rows=2700)
                  ← ¡SEQ SCAN! Aquí falta un índice
              ->  Hash  (cost=20.00..20.00 rows=800 width=24)
                    ->  Seq Scan on usuarios u  (cost=0.00..20.00 rows=800)
                          Filter: (activo = true)

Planning Time: 0.5 ms
Execution Time: 5.4 ms
```

**Lo que debes buscar:**
- `Seq Scan` en tablas grandes con filtros → probable falta de índice
- `cost=` alto → operación costosa
- `rows=estimado` muy diferente de `rows=real` → estadísticas desactualizadas (`ANALYZE`)
- `loops=N` → el nodo se ejecutó N veces (problemas de nested loops)

```sql
-- Actualizar estadísticas para que el planner tome mejores decisiones
ANALYZE pedidos;
ANALYZE usuarios;
ANALYZE;  -- todas las tablas
```

### Patrones de Consulta Problemáticos

```sql
-- ❌ LIKE con prefijo comodín — no puede usar índice B-Tree
SELECT * FROM usuarios WHERE email LIKE '%@gmail.com';
-- ✅ Alternativa: índice GIN con pg_trgm para búsqueda de texto
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX idx_usuarios_email_trgm ON usuarios USING GIN (email gin_trgm_ops);
SELECT * FROM usuarios WHERE email LIKE '%@gmail.com';  -- ahora usa el índice

-- ❌ Función sobre la columna indexada — el índice no se puede usar
SELECT * FROM pedidos WHERE DATE(creado_en) = '2024-01-15';
-- ✅ Reformular como rango
SELECT * FROM pedidos
WHERE creado_en >= '2024-01-15' AND creado_en < '2024-01-16';

-- ❌ OR sobre columnas distintas — puede requerir seq scan
SELECT * FROM usuarios WHERE email = 'a@b.com' OR nombre = 'Ana';
-- ✅ UNION puede usar índices de cada condición por separado
SELECT * FROM usuarios WHERE email = 'a@b.com'
UNION
SELECT * FROM usuarios WHERE nombre = 'Ana';

-- ❌ SELECT * en tablas con muchas columnas — transfiere datos innecesarios
SELECT * FROM pedidos WHERE estado = 'pendiente';
-- ✅ Solo las columnas que necesitas
SELECT id, usuario_id, total, creado_en FROM pedidos WHERE estado = 'pendiente';

-- Index-Only Scan — cuando todas las columnas de la consulta están en el índice
-- Mucho más rápido: no necesita acceder al heap (la tabla)
CREATE INDEX idx_pedidos_covering ON pedidos (estado, creado_en) INCLUDE (total, usuario_id);
SELECT total, usuario_id FROM pedidos WHERE estado = 'pendiente' AND creado_en > NOW() - INTERVAL '7 days';
-- Puede hacer Index Only Scan
```

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - **B-Tree** es el índice por defecto — para igualdad y rangos. **GIN** para arrays, JSONB y full-text search.
> - **Índices parciales** son más pequeños y rápidos que los índices completos — úsalos cuando consultas un subconjunto (soft deletes, estados activos).
> - `EXPLAIN (ANALYZE, BUFFERS)` es la herramienta de diagnóstico. Busca `Seq Scan` en tablas grandes con filtros.
> - `LIKE '%prefijo%'` no puede usar índice B-Tree. Usa `pg_trgm` con índice GIN para búsqueda de texto arbitraria.
> - Los índices compuestos se pueden usar si la consulta incluye las columnas desde la izquierda. El orden de las columnas importa.

### Recursos
- 🌐 postgresql.org/docs/current/indexes.html
- 🌐 use-the-index-luke.com — la referencia definitiva de indexación en SQL
- 🌐 explain.dalibo.com — visualizador gráfico de planes EXPLAIN de PostgreSQL

---
`#postgresql` `#indices` `#performance` `#explain` `#b-tree` `#gin`
