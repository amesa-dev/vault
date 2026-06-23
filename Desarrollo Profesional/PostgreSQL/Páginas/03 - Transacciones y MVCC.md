# 🐘 PostgreSQL 03 — Transacciones y MVCC

[[Desarrollo Profesional/PostgreSQL/PostgreSQL|⬅️ Volver a PostgreSQL]] | [[Desarrollo Profesional/PostgreSQL/Páginas/02 - Indexación y Performance|← 02]] | [[Desarrollo Profesional/PostgreSQL/Páginas/04 - Funciones Avanzadas|04 →]]

> [!abstract] Introducción
> Las transacciones y el MVCC (Multi-Version Concurrency Control) son los fundamentos que hacen de PostgreSQL una base de datos de producción fiable. MVCC es la razón por la que los lectores no bloquean a los escritores y viceversa — PostgreSQL mantiene múltiples versiones de cada fila para garantizar consistencia sin locks agresivos. Entender esto explica el comportamiento de deadlocks, vacuum, y los niveles de aislamiento.

## ¿De qué vamos a hablar?

Las propiedades ACID, el mecanismo MVCC de PostgreSQL, los niveles de aislamiento de transacciones y cómo gestionar deadlocks.

---

## El Concepto

### ACID — Las Propiedades de las Transacciones

```sql
-- Todo lo que ocurre dentro de BEGIN...COMMIT es una transacción
BEGIN;
    UPDATE cuentas SET saldo = saldo - 100 WHERE id = 1;
    UPDATE cuentas SET saldo = saldo + 100 WHERE id = 2;
    -- Si cualquiera de las dos falla, ambas se revierten
COMMIT;

-- Rollback explícito
BEGIN;
    DELETE FROM usuarios WHERE id = 42;
    -- cambié de opinión
ROLLBACK;
```

**Atomicidad**: o todas las operaciones se aplican, o ninguna.  
**Consistencia**: la transacción lleva la BD de un estado válido a otro estado válido (las constraints se respetan).  
**Aislamiento**: las transacciones concurrentes no se interfieren entre sí (hasta el nivel configurado).  
**Durabilidad**: los cambios confirmados sobreviven a fallos del sistema (WAL).

### MVCC — Multi-Version Concurrency Control

En lugar de bloquear filas para evitar lecturas de datos en modificación, PostgreSQL mantiene **múltiples versiones de cada fila**. Cada fila tiene metadatos ocultos:

```sql
-- Columnas de sistema ocultas (no aparecen en SELECT *)
SELECT xmin, xmax, ctid, * FROM usuarios LIMIT 5;
-- xmin: ID de la transacción que creó esta versión de la fila
-- xmax: ID de la transacción que eliminó/actualizó esta versión (0 si activa)
-- ctid:  posición física de la fila (bloque, offset)
```

**Cómo funciona una UPDATE:**
1. La UPDATE no modifica la fila en el lugar — crea una nueva versión de la fila
2. Marca la versión antigua con `xmax = ID_transacción_actual`
3. Crea la nueva versión con `xmin = ID_transacción_actual`
4. Las lecturas que empezaron antes del UPDATE siguen viendo la versión antigua

Esto significa: **los lectores nunca bloquean a los escritores, y los escritores nunca bloquean a los lectores**.

**El precio: bloat y VACUUM**  
Las versiones antiguas de las filas (llamadas *dead tuples*) se acumulan. `VACUUM` es el proceso que las limpia:

```sql
-- Ver cuántas dead tuples hay
SELECT relname, n_dead_tup, n_live_tup, last_vacuum, last_autovacuum
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;

-- Manual VACUUM — raramente necesario, autovacuum lo hace
VACUUM mi_tabla;

-- VACUUM FULL — recupera espacio en disco (bloquea la tabla, úsalo con cuidado)
VACUUM FULL mi_tabla;

-- ANALYZE — actualiza estadísticas del planner
ANALYZE mi_tabla;
```

### Niveles de Aislamiento

PostgreSQL acepta los cuatro niveles del estándar SQL, pero internamente solo implementa **tres comportamientos distintos**: **Read Uncommitted se comporta exactamente igual que Read Committed** (PostgreSQL nunca permite lecturas sucias / *dirty reads*). Los tres niveles reales son Read Committed, Repeatable Read y Serializable:

```sql
-- Read Committed (por defecto) — ve commits de otras transacciones durante la transacción
-- Puede ver: non-repeatable reads, phantom reads
BEGIN;
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
SELECT saldo FROM cuentas WHERE id = 1;  -- devuelve el valor actual
-- (otra transacción hace UPDATE y COMMIT)
SELECT saldo FROM cuentas WHERE id = 1;  -- puede devolver un valor diferente
COMMIT;

-- Repeatable Read — ve un snapshot del inicio de la transacción
-- Evita: non-repeatable reads. Puede ver: phantom reads (en teoría, no en Postgres)
BEGIN;
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SELECT saldo FROM cuentas WHERE id = 1;  -- snapshot tomado aquí
-- (otra transacción hace UPDATE y COMMIT)
SELECT saldo FROM cuentas WHERE id = 1;  -- mismo valor que antes
COMMIT;

-- Serializable — garantía más fuerte: el resultado es equivalente a ejecución serial
-- Puede abortar transacciones (serialization failure) — requiere retry logic
BEGIN;
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
-- ...operaciones...
COMMIT;  -- puede fallar con "ERROR: could not serialize access due to concurrent update"
```

### Locks y Deadlocks

```sql
-- Tipos de locks en PostgreSQL:
-- SELECT — no toma lock de fila
-- UPDATE/DELETE — toma RowExclusiveLock en la fila
-- SELECT FOR UPDATE — lock explícito de lectura (bloquea UPDATEs concurrentes)
-- SELECT FOR SHARE — permite otras SELECT FOR SHARE pero no SELECT FOR UPDATE

-- SELECT FOR UPDATE — para leer y luego modificar sin que otro lo cambie entre medio
BEGIN;
SELECT * FROM inventario WHERE producto_id = 42 FOR UPDATE;
-- El producto_id 42 está bloqueado hasta el COMMIT
UPDATE inventario SET cantidad = cantidad - 1 WHERE producto_id = 42;
COMMIT;

-- Evitar deadlocks — siempre bloquea en el mismo orden
-- MAL (puede deadlocks si dos transacciones lo hacen en orden inverso):
-- T1: UPDATE cuenta 1, luego UPDATE cuenta 2
-- T2: UPDATE cuenta 2, luego UPDATE cuenta 1

-- BIEN — usando SELECT FOR UPDATE NOWAIT o ordenando por ID
BEGIN;
SELECT * FROM cuentas WHERE id IN (1, 2) ORDER BY id FOR UPDATE;
UPDATE cuentas SET saldo = saldo - 100 WHERE id = 1;
UPDATE cuentas SET saldo = saldo + 100 WHERE id = 2;
COMMIT;

-- Ver locks activos
SELECT pid, relation::regclass, mode, granted, query
FROM pg_locks l
JOIN pg_stat_activity a ON l.pid = a.pid
WHERE relation IS NOT NULL AND NOT granted;

-- Matar una transacción bloqueada
SELECT pg_terminate_backend(pid);
```

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - **MVCC** permite que lectores y escritores no se bloqueen entre sí — PostgreSQL mantiene múltiples versiones de cada fila. El precio son las *dead tuples* que AUTOVACUUM limpia.
> - El nivel de aislamiento por defecto es **Read Committed** — es suficiente para la mayoría de casos. Usa **Repeatable Read** cuando necesites que una transacción vea un snapshot consistente.
> - **SELECT FOR UPDATE** bloquea filas para modificarlas — úsalo cuando lees y luego escribes en la misma transacción (patrón optimistic/pessimistic locking).
> - Para evitar **deadlocks**, siempre bloquea los recursos en el mismo orden en todas las transacciones.

### Recursos
- 🌐 postgresql.org/docs/current/mvcc.html
- 🌐 2ndquadrant.com/en/blog/mvcc-unmasked — explicación visual del MVCC
- 📖 *PostgreSQL 14 Internals* — Egor Rogov

---
`#postgresql` `#transacciones` `#mvcc` `#acid` `#locks`
