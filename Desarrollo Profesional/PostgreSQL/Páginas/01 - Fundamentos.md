# 🐘 PostgreSQL 01 — Fundamentos

[[Desarrollo Profesional/PostgreSQL/PostgreSQL|⬅️ Volver a PostgreSQL]] | [[Desarrollo Profesional/PostgreSQL/Páginas/02 - Indexación y Performance|02 →]]

> [!abstract] Introducción
> PostgreSQL es mucho más que un sistema de gestión de bases de datos relacionales. Su sistema de tipos es extremadamente rico, soporta JSON nativo, arrays, tipos geométricos, rangos temporales y extensiones. Esta página cubre los fundamentos: tipos de datos que más importan, DDL correcto, DML eficiente y las construcciones SQL que separan al que sabe PostgreSQL del que sabe SQL genérico.

## ¿De qué vamos a hablar?

Los tipos de datos de PostgreSQL, el DDL (definición de tablas y constraints), el DML (consultas: JOINs, agregaciones, CTEs, window functions) y los comandos de psql más útiles.

### Conceptos que vamos a cubrir
- Tipos de datos: cuándo usar cada uno
- DDL: constraints correctas desde el diseño
- JOINs y sus diferencias
- CTEs y window functions
- psql: navegación eficiente

---

## El Concepto

### Tipos de Datos — Los que Más Importan

```sql
-- Enteros
SMALLINT       -- 2 bytes, -32768 a 32767
INTEGER        -- 4 bytes, -2^31 a 2^31 — el más usado
BIGINT         -- 8 bytes, -2^63 a 2^63 — para IDs en tablas grandes

-- IDs: SERIAL vs IDENTITY (usar IDENTITY desde PostgreSQL 10)
id SERIAL PRIMARY KEY        -- forma antigua
id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY  -- forma moderna

-- UUID — para IDs distribuidos o que no quieres exponer secuencialmente
id UUID DEFAULT gen_random_uuid() PRIMARY KEY  -- desde PostgreSQL 13

-- Texto
TEXT           -- longitud variable sin límite — preferir sobre VARCHAR
VARCHAR(n)     -- con límite, la diferencia de rendimiento con TEXT es mínima
CHAR(n)        -- longitud fija, rellena con espacios — casi nunca tiene sentido

-- Numéricos
NUMERIC(p, s)  -- precisión exacta — SIEMPRE para dinero (no FLOAT)
FLOAT          -- IEEE 754, impreciso — NUNCA para dinero
REAL           -- 4 bytes float

-- Temporales
TIMESTAMP WITH TIME ZONE  -- almacena en UTC, muestra en tz local — preferir
TIMESTAMP                 -- sin zona horaria — puede causar confusión
DATE                      -- solo fecha
INTERVAL                  -- duración: '1 day', '2 hours', '30 minutes'

-- Booleano
BOOLEAN        -- true/false/null

-- Arrays — PostgreSQL soporte arrays nativo
tags TEXT[]               -- array de texto
numeros INTEGER[]

-- JSON
JSON           -- texto validado como JSON, sin índices
JSONB          -- binario, permite índices GIN — preferir siempre sobre JSON
```

### DDL — Definición de Tablas y Constraints

```sql
-- Tabla bien definida con constraints adecuados
CREATE TABLE usuarios (
    id          UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    email       TEXT NOT NULL UNIQUE,
    nombre      TEXT NOT NULL CHECK (length(nombre) >= 2),
    edad        SMALLINT CHECK (edad >= 0 AND edad <= 150),
    activo      BOOLEAN NOT NULL DEFAULT TRUE,
    rol         TEXT NOT NULL DEFAULT 'usuario'
                    CHECK (rol IN ('admin', 'editor', 'usuario')),
    creado_en   TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    actualizado_en TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

-- Tabla con clave foránea
CREATE TABLE pedidos (
    id          UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    usuario_id  UUID NOT NULL REFERENCES usuarios(id) ON DELETE CASCADE,
    total       NUMERIC(10, 2) NOT NULL CHECK (total >= 0),
    estado      TEXT NOT NULL DEFAULT 'pendiente'
                    CHECK (estado IN ('pendiente', 'pagado', 'enviado', 'cancelado')),
    creado_en   TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

-- Índice para la clave foránea — PostgreSQL NO crea índices automáticamente para FKs
CREATE INDEX idx_pedidos_usuario_id ON pedidos (usuario_id);

-- Índice parcial — solo indexa filas que cumplen la condición
CREATE INDEX idx_pedidos_pendientes ON pedidos (creado_en)
    WHERE estado = 'pendiente';

-- Modificar tabla de forma segura
ALTER TABLE usuarios ADD COLUMN telefono TEXT;       -- añade columna nullable (sin default, muy rápido)
ALTER TABLE usuarios ADD COLUMN puntos INTEGER NOT NULL DEFAULT 0;  -- lento en tablas grandes
ALTER TABLE usuarios ALTER COLUMN telefono SET NOT NULL;  -- requiere que no haya NULLs
ALTER TABLE usuarios DROP COLUMN IF EXISTS campo_obsoleto;
```

### DML — Consultas Eficientes

```sql
-- INSERT con retorno de valores
INSERT INTO usuarios (email, nombre) VALUES ('ana@test.com', 'Ana García')
RETURNING id, creado_en;

-- INSERT ON CONFLICT — upsert
INSERT INTO configuracion (clave, valor) VALUES ('timeout', '30')
ON CONFLICT (clave) DO UPDATE SET valor = EXCLUDED.valor, actualizado_en = NOW();

-- UPDATE con JOIN
UPDATE pedidos p
SET estado = 'enviado'
FROM usuarios u
WHERE p.usuario_id = u.id
  AND u.pais = 'ES'
  AND p.estado = 'pagado';

-- DELETE con retorno
DELETE FROM sesiones WHERE expira_en < NOW()
RETURNING id, usuario_id;

-- Tipos de JOIN
-- INNER JOIN — solo filas con coincidencia en ambas tablas
SELECT u.nombre, COUNT(p.id) AS num_pedidos
FROM usuarios u
INNER JOIN pedidos p ON u.id = p.usuario_id
GROUP BY u.id, u.nombre;

-- LEFT JOIN — todos los usuarios, incluso los sin pedidos
SELECT u.nombre, COUNT(p.id) AS num_pedidos
FROM usuarios u
LEFT JOIN pedidos p ON u.id = p.usuario_id
GROUP BY u.id, u.nombre;

-- Diferencia entre WHERE y HAVING
SELECT usuario_id, COUNT(*) AS total
FROM pedidos
WHERE estado != 'cancelado'     -- filtra ANTES de agrupar
GROUP BY usuario_id
HAVING COUNT(*) > 5;            -- filtra DESPUÉS de agrupar
```

### CTEs — Common Table Expressions

```sql
-- CTE básica — mejora la legibilidad
WITH pedidos_recientes AS (
    SELECT *
    FROM pedidos
    WHERE creado_en > NOW() - INTERVAL '30 days'
),
usuarios_activos AS (
    SELECT DISTINCT usuario_id
    FROM pedidos_recientes
    WHERE estado = 'pagado'
)
SELECT u.nombre, u.email, COUNT(pr.id) AS pedidos_ultimo_mes
FROM usuarios u
JOIN usuarios_activos ua ON u.id = ua.usuario_id
JOIN pedidos_recientes pr ON u.id = pr.usuario_id
GROUP BY u.id, u.nombre, u.email;

-- CTE recursiva — para jerarquías (categorías, organigramas)
WITH RECURSIVE categoria_arbol AS (
    -- Caso base: categorías raíz
    SELECT id, nombre, padre_id, 0 AS nivel
    FROM categorias
    WHERE padre_id IS NULL

    UNION ALL

    -- Caso recursivo: hijos de las categorías ya encontradas
    SELECT c.id, c.nombre, c.padre_id, ca.nivel + 1
    FROM categorias c
    JOIN categoria_arbol ca ON c.padre_id = ca.id
)
SELECT * FROM categoria_arbol ORDER BY nivel, nombre;
```

### Window Functions

```sql
-- Ranking por columna
SELECT
    nombre,
    total,
    ROW_NUMBER() OVER (ORDER BY total DESC) AS posicion,
    RANK() OVER (ORDER BY total DESC) AS rank_con_empates,
    DENSE_RANK() OVER (ORDER BY total DESC) AS dense_rank
FROM pedidos;

-- Ranking por partición — ranking dentro de cada usuario
SELECT
    usuario_id,
    id AS pedido_id,
    total,
    ROW_NUMBER() OVER (PARTITION BY usuario_id ORDER BY total DESC) AS pedido_rank
FROM pedidos;

-- Acumulado y media móvil
SELECT
    fecha,
    ventas_dia,
    SUM(ventas_dia) OVER (ORDER BY fecha) AS ventas_acumuladas,
    AVG(ventas_dia) OVER (ORDER BY fecha ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS media_7_dias
FROM ventas_diarias;

-- LAG/LEAD — valor de la fila anterior/siguiente
SELECT
    fecha,
    ventas_dia,
    LAG(ventas_dia) OVER (ORDER BY fecha) AS ventas_dia_anterior,
    ventas_dia - LAG(ventas_dia) OVER (ORDER BY fecha) AS diferencia
FROM ventas_diarias;
```

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - Usa `UUID` con `gen_random_uuid()` para IDs distribuidos; `GENERATED ALWAYS AS IDENTITY` para IDs secuenciales. Evita `SERIAL` en código nuevo.
> - Usa `NUMERIC` para dinero y cálculos financieros. Nunca `FLOAT`.
> - Usa `TIMESTAMP WITH TIME ZONE` para almacenar instantes. Almacena en UTC, muestra en la zona horaria del cliente.
> - PostgreSQL **no crea índices automáticamente para claves foráneas** — tienes que crearlos manualmente.
> - Las CTEs hacen legible el SQL complejo. Las window functions (`ROW_NUMBER`, `LAG`, `SUM OVER`) evitan subconsultas correlated costosas.

### Recursos
- 🌐 postgresql.org/docs/current — documentación oficial completa
- 📖 *The Art of PostgreSQL* — Dimitri Fontaine
- 🌐 postgresqlco.nf — explicaciones de parámetros de configuración

---
`#postgresql` `#sql` `#ddl` `#cte` `#window-functions`
