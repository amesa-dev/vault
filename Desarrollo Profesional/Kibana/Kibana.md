# 🧭 Kibana

[[Desarrollo Profesional/Inicio Profesional|⬅️ Volver a Desarrollo Profesional]]

> [!abstract] Kibana
> Interfaz de usuario gráfica para explorar, visualizar y administrar datos indexados en Elasticsearch, que forma parte del conocido stack Elastic (ELK).

---

## 🔑 Módulos Principales

### 1. Discover (Descubrir)
- Permite buscar y filtrar documentos de registro indexados en tiempo real.
- Utiliza **KQL (Kibana Query Language)** o consultas Lucene para filtrar.

### 2. Visualize Library (Visualizar)
- Creación de gráficos individuales (barras, líneas, circulares, coordenadas geográficas, TSVB).
- **Lens:** Herramienta interactiva de arrastrar y soltar para crear visualizaciones rápidas.

### 3. Dashboards (Paneles)
- Lienzos interactivos que reúnen múltiples visualizaciones para dar una perspectiva unificada del estado de la infraestructura o la aplicación.

---

## ⚡ Consultas con KQL (Kibana Query Language)

### Sintaxis Básica:
```kql
# Búsqueda exacta
response: 200

# Búsqueda con comodín (wildcard)
machine.os: win*

# Operadores lógicos
response: 200 AND (status: "error" OR status: "warning")

# Rangos numéricos
bytes > 1000 AND bytes <= 5000
```

---
`#kibana` `#elk` `#logs` `#visualizacion` `#apuntes`
