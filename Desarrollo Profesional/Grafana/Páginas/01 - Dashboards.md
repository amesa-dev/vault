# 📊 Grafana 01 — Dashboards y Paneles

[[Desarrollo Profesional/Grafana/Grafana|⬅️ Volver a Grafana]] | [[Desarrollo Profesional/Grafana/Páginas/02 - Alertas y Data Sources|02 →]]

> [!abstract] Introducción
> Un dashboard de Grafana es un conjunto de paneles que muestran el estado del sistema en tiempo real. El arte de construir buenos dashboards es el arte de responder preguntas de negocio y operativas con datos — no mostrar todos los datos disponibles, sino los que importan para tomar decisiones.

## ¿De qué vamos a hablar?

Paneles, variables, time range, anotaciones y cómo automatizar la creación de dashboards con provisioning.

---

## El Concepto

### Tipos de Paneles

```
Panel → visualización individual dentro de un dashboard

Tipos principales:
├── Time series    → series temporales (el más usado para métricas)
├── Stat           → valor único grande (KPI: errores/s, latencia actual)
├── Table          → datos tabulares
├── Bar chart      → comparación entre categorías
├── Pie/Donut      → distribución porcentual
├── Gauge          → valor en un rango (como un velocímetro)
├── Logs           → panel de logs (con fuente Loki o Elasticsearch)
└── Heatmap        → distribución de valores en el tiempo (histogramas)
```

### Variables — Dashboards Dinámicos

Las variables permiten filtrar todos los paneles del dashboard simultáneamente:

```
Dashboard Variables (Settings → Variables):

Tipo: Query → consulta los valores posibles desde la fuente de datos
Nombre: namespace
Label: "Namespace"
Query (Prometheus): label_values(kube_pod_info, namespace)

Tipo: Query
Nombre: pod
Label: "Pod"
Query: label_values(kube_pod_info{namespace="$namespace"}, pod)
# ↑ usa la variable $namespace para filtrar pods

Tipo: Interval
Nombre: intervalo
Values: 1m, 5m, 10m, 30m, 1h
Default: 5m
# Usado en las queries como: rate(metric[${intervalo}])
```

```promql
# Query usando variables en Grafana
rate(http_requests_total{namespace="$namespace", pod="$pod"}[$intervalo])

# Repetir panel por variable (Row/Panel Repeat)
# En Panel Options → Repeat → seleccionar variable
# Crea un panel por cada valor de la variable
```

### Estructura de un Dashboard Bien Diseñado

```
Dashboard: "Servicio de Pedidos"

Row 1: 📊 Overview (RED metrics)
├── Requests/s (rate)        → Stat panel
├── Error rate (%)           → Stat panel con threshold rojo >1%
├── Latencia P50/P95/P99     → Stat panel o Table
└── Disponibilidad (uptime)  → Stat panel

Row 2: 📈 Métricas de Negocio
├── Pedidos creados/hora     → Time series
├── Tasa de conversión       → Time series
└── Ingresos por minuto      → Time series

Row 3: 🔧 Infraestructura
├── CPU usage por pod        → Time series con variable $pod
├── Memory usage             → Time series
└── Réplicas activas         → Stat

Row 4: 🐛 Errores y Latencia
├── Distribución de códigos HTTP  → Bar chart
├── Top 10 endpoints por latencia → Table
└── Errores por tipo              → Time series
```

### Provisioning — Dashboards como Código

```yaml
# grafana/provisioning/dashboards/mi-servicio.json → se carga automáticamente
# grafana/provisioning/datasources/prometheus.yaml

apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    jsonData:
      httpMethod: POST
      exemplarTraceIdDestinations:
        - name: trace_id
          datasourceUid: tempo
```

```json
// Estructura básica de un dashboard JSON (para versionado en Git)
{
  "title": "Servicio de Pedidos",
  "uid": "pedidos-overview",
  "tags": ["pedidos", "produccion"],
  "refresh": "30s",
  "time": {"from": "now-1h", "to": "now"},
  "templating": {
    "list": [
      {
        "name": "namespace",
        "type": "query",
        "query": "label_values(kube_pod_info, namespace)"
      }
    ]
  },
  "panels": [...]
}
```

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - Un buen dashboard responde preguntas específicas — empieza por las **RED metrics** (Rate, Errors, Duration) para cualquier servicio.
> - Las **variables** hacen los dashboards reutilizables — usa `label_values()` para poblarlas desde Prometheus.
> - Guarda los dashboards en Git con **provisioning** — un dashboard que solo existe en la UI de Grafana es un dashboard que se puede perder.
> - Usa **thresholds** en los Stat panels (verde/amarillo/rojo) para comunicar el estado de un vistazo sin leer los números.

### Recursos
- 🌐 grafana.com/docs/grafana/latest/dashboards
- 🌐 grafana.com/docs/grafana/latest/administration/provisioning

---
`#grafana` `#dashboards` `#paneles` `#variables` `#promql`
