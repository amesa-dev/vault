# 🧭 Kibana 01 — Discover y KQL

[[Desarrollo Profesional/Kibana/Kibana|⬅️ Volver a Kibana]] | [[Desarrollo Profesional/Kibana/Páginas/02 - Dashboards y APM|02 →]]

> [!abstract] Introducción
> Discover es el módulo de Kibana para exploración libre de logs. KQL (Kibana Query Language) es el lenguaje para filtrar esos logs. Dominar Discover y KQL es la habilidad práctica más importante para debuggear problemas en producción con el stack ELK.

---

## El Concepto

### Discover — Exploración de Logs

```
Interfaz de Discover:
├── Time picker (arriba derecha) → seleccionar rango temporal
├── Search bar → KQL o Lucene
├── Fields list (izquierda) → campos disponibles en el índice
├── Documents list (centro) → logs que cumplen el filtro
└── Histogram (arriba) → distribución temporal de los logs
```

**Flujo de debugging típico:**
1. Seleccionar el rango temporal donde ocurrió el problema
2. Filtrar por servicio/pod
3. Filtrar por nivel de error
4. Buscar el mensaje de error específico
5. Explorar los campos del documento para entender el contexto

### KQL — Kibana Query Language

```kql
# Búsqueda por campo exacto
kubernetes.labels.app: "mi-api"

# Búsqueda por valor con comodín
message: *error*
kubernetes.pod.name: mi-api-*

# Campos numéricos
http.response.status_code: 500
http.response.bytes > 10000

# Rangos
http.response.bytes >= 1000 AND http.response.bytes <= 5000

# Operadores lógicos
kubernetes.labels.app: "mi-api" AND log.level: "error"
log.level: "error" OR log.level: "critical"
NOT log.level: "debug"

# Búsqueda en objetos anidados
event.module: "nginx" AND http.response.status_code: 404

# Wildcards en nombres de campo
http.*: 500  # busca 500 en cualquier campo que empiece por http.

# Búsqueda de frase exacta
message: "connection refused"
message: "null pointer exception"

# Ejemplos reales de debugging:
# Todos los errores 5xx del servicio en la última hora
kubernetes.labels.app: "mi-api" AND http.response.status_code >= 500

# Logs de un pod específico
kubernetes.pod.name: "mi-api-7d8f9b-xyz" AND log.level: "error"

# Requests lentos (más de 2 segundos)
kubernetes.labels.app: "mi-api" AND http.response_time > 2000

# Errores de conexión a BD
message: "could not connect" AND kubernetes.labels.app: "mi-api"
```

### Lens — Visualizaciones Rápidas

Lens es el editor interactivo de visualizaciones de Kibana. Para crear una visualización rápida desde Discover:

```
Desde Discover → "Visualize" (botón) → se abre Lens con los datos actuales

Casos de uso frecuentes en Lens:
├── Distribución de códigos HTTP por tiempo → Bar chart agrupado por status_code
├── Top 10 endpoints con más errores       → Treemap por url + status_code=5xx
├── Latencia media por servicio             → Line chart con Average(response_time)
└── Error rate (%) en el tiempo            → Line chart con Ratio
```

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - KQL usa `campo: valor` (no `campo = valor` como SQL). Los strings no necesitan comillas a menos que contengan espacios o caracteres especiales.
> - El **time picker** es lo primero que hay que ajustar — reducir el rango temporal reduce el número de documentos y acelera la búsqueda.
> - Los **wildcards** (`*`) son costosos — úsalos en los valores, no en los nombres de campos, y en rangos de tiempo pequeños.
> - Guarda las búsquedas frecuentes con "Save" — puedes reutilizarlas en dashboards.

### Recursos
- 🌐 elastic.co/guide/en/kibana/current/discover.html
- 🌐 elastic.co/guide/en/kibana/current/kuery-query.html

---
`#kibana` `#kql` `#discover` `#logs` `#elk`
