# 📊 Grafana 02 — Alertas y Data Sources

[[Desarrollo Profesional/Grafana/Grafana|⬅️ Volver a Grafana]] | [[Desarrollo Profesional/Grafana/Páginas/01 - Dashboards|← 01]]

> [!abstract] Introducción
> Grafana Alerting unifica las alertas de Prometheus, Loki y otras fuentes en un único sistema de gestión. Los data sources son la capa de conexión entre Grafana y tus sistemas de datos — entender cuáles usar y cómo configurarlos es fundamental para una plataforma de observabilidad coherente.

---

## El Concepto

### Grafana Alerting

```
Arquitectura de alertas en Grafana:

Alert Rule (regla)
    ↓ evalúa periódicamente
Contact Point (destino)
    ↓ notifica a
Notification Policy (política)
    ↓ decide quién recibe qué

Alert Rule → define la condición (query + threshold)
Contact Point → define dónde notificar (Slack, PagerDuty, email, webhook)
Notification Policy → reglas de routing: qué alertas van a qué contact point
Silences → silenciar alertas durante mantenimiento
```

```yaml
# Alert Rule example (formato YAML para provisioning)
groups:
  - name: mi-servicio
    interval: 1m
    rules:
      # Alerta: error rate > 5% durante 5 minutos
      - alert: ErrorRateAlta
        expr: |
          sum(rate(http_requests_total{status=~"5..", service="mi-api"}[5m]))
          /
          sum(rate(http_requests_total{service="mi-api"}[5m]))
          > 0.05
        for: 5m  # debe cumplirse durante 5 minutos para disparar
        labels:
          severity: critical
          team: backend
        annotations:
          summary: "Error rate alta en mi-api"
          description: "El error rate es {{ humanizePercentage $value }} (umbral: 5%)"
          runbook_url: "https://wiki.empresa.com/runbooks/error-rate"

      # Alerta: latencia P95 > 2 segundos
      - alert: LatenciaAltaP95
        expr: |
          histogram_quantile(0.95, 
            sum by (le) (rate(http_request_duration_seconds_bucket{service="mi-api"}[5m]))
          ) > 2
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Latencia P95 supera 2s en mi-api"
```

### Data Sources Principales

```
Stack de observabilidad moderno (LGTM):
├── Loki      → logs (búsqueda y análisis de logs)
├── Grafana   → visualización (el centro de todo)
├── Tempo     → trazas distribuidas
└── Mimir     → métricas de Prometheus a escala

Stack ELK alternativo:
├── Elasticsearch → almacén de logs/documentos
├── Logstash      → ingestión y transformación
└── Kibana        → visualización
```

```yaml
# Configurar múltiples data sources (provisioning)
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    url: http://prometheus:9090
    isDefault: true

  - name: Loki
    type: loki
    url: http://loki:3100

  - name: Tempo
    type: tempo
    url: http://tempo:3200
    jsonData:
      tracesToLogsV2:
        datasourceUid: loki  # enlaza trazas con logs
      tracesToMetrics:
        datasourceUid: prometheus  # enlaza trazas con métricas

  - name: PostgreSQL
    type: postgres
    url: localhost:5432
    database: analytics
    user: grafana_reader
    secureJsonData:
      password: "${GRAFANA_PG_PASSWORD}"
    jsonData:
      sslmode: require
```

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - Grafana Alerting tiene tres componentes: **Alert Rules** (la condición), **Contact Points** (el destino) y **Notification Policies** (el routing).
> - Añade `for: 5m` a las alertas críticas para evitar falsos positivos por spikes momentáneos.
> - Incluye `runbook_url` en las anotaciones — cuando la alerta dispara a las 3am, el oncall necesita saber qué hacer.
> - El stack LGTM (Loki+Grafana+Tempo+Mimir) es la evolución natural de Prometheus+Grafana cuando necesitas correlacionar logs, métricas y trazas.

### Recursos
- 🌐 grafana.com/docs/grafana/latest/alerting
- 🌐 grafana.com/docs/grafana/latest/datasources

---
`#grafana` `#alertas` `#data-sources` `#lgtm`
