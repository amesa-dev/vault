# 🐘 PostgreSQL

[[Desarrollo Profesional/Inicio Profesional|⬅️ Volver a Desarrollo Profesional]]

> [!abstract] PostgreSQL
> PostgreSQL es un sistema de gestión de bases de datos relacionales de código abierto, potente y altamente extensible, que soporta tanto consultas SQL (relacionales) como JSON (no relacionales).

---

## 🔑 Consultas Comunes y Sintaxis

### 1. Creación de Tablas
```sql
CREATE TABLE usuarios (
    id SERIAL PRIMARY KEY,
    nombre VARCHAR(100) NOT NULL,
    email VARCHAR(150) UNIQUE NOT NULL,
    creado_en TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### 2. Tipos de JOINs
- **INNER JOIN:** Devuelve filas cuando hay una coincidencia en ambas tablas.
- **LEFT JOIN (o LEFT OUTER JOIN):** Devuelve todas las filas de la tabla de la izquierda, y las filas coincidentes de la derecha.

```sql
SELECT u.nombre, p.nombre_proyecto
FROM usuarios u
LEFT JOIN proyectos p ON u.id = p.usuario_id;
```

---

## ⚡ Índices y Rendimiento
Los índices se utilizan para acelerar la búsqueda de datos.

```sql
-- Crear un índice básico
CREATE INDEX idx_usuarios_email ON usuarios(email);

-- Crear un índice compuesto
CREATE INDEX idx_usuarios_nombre_creado ON usuarios(nombre, creado_en);
```

---

## 🛠️ Comandos Útiles de la Consola (`psql`)

| Comando | Descripción |
| --- | --- |
| `\l` | Listar todas las bases de datos |
| `\c nombre_bd` | Conectarse a una base de datos |
| `\dt` | Listar todas las tablas de la base de datos actual |
| `\d nombre_tabla` | Describir la estructura de una tabla |
| `\q` | Salir de `psql` |

---
`#postgresql` `#sql` `#database` `#apuntes`
