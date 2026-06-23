# 🔥 Prometheus 02 — Configuración y Alertas

[[Desarrollo Profesional/Prometheus/Prometheus|⬅️ Volver a Prometheus]] | [[Desarrollo Profesional/Prometheus/Páginas/01 - Métricas y PromQL|← 01]]

> [!abstract] Introducción
> La configuración de Prometheus define qué scrape, con qué frecuencia y cómo etiqueta las series. Alertmanager es el componente que gestiona las alertas: las agrupa, las silencia y las envía a los canales correctos. Juntos forman el sistema de observabilidad más usado en Kubernetes.

---

## El Concepto

### prometheus.yml — Configuración de Scraping

```yaml
# prometheus.yml
global:
  scrape_interval: 15s       # cada cuánto scrape los endpoints
  evaluation_interval: 15s   # cada cuánto evalúa las reglas de alerta
  scrape_timeout: 10s

# Reglas de alerta
rule_files:
  - "rules/*.yaml"

# Configuración de Alertmanager
alerting:
  alertmanagers:
    - static_configs:
        - targets: ["alertmanager:9093"]

# Jobs de scraping
scrape_configs:
  # Scraping de Prometheus mismo
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  # Scraping de una aplicación Python/FastAPI
  - job_name: "mi-api"
    metrics_path: /metrics
    static_configs:
      - targets: ["mi-api:8080"]
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance
        regex: "([^:]+).*"
        replacement: "$1"

  # Service Discovery en Kubernetes — descubre pods automáticamente
  - job_name: "kubernetes-pods"
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      # Solo scrape pods con la anotación prometheus.io/scrape: "true"
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: "true"
      # Usa la anotación para el path (/metrics por defecto)
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      # Añade labels de Kubernetes a las series
      - source_labels: [__meta_kubernetes_namespace]
        target_label: namespace
      - source_labels: [__meta_kubernetes_pod_name]
        target_label: pod
      - source_labels: [__meta_kubernetes_pod_label_app]
        target_label: app
```

```yaml
# Anotaciones en un Deployment de Kubernetes para que Prometheus lo descubra
spec:
  template:
    metadata:
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/metrics"
```

### Reglas de Alerta

```yaml
# rules/mi-servicio.yaml
groups:
  - name: mi-servicio-alerts
    interval: 30s
    rules:
      # ── DISPONIBILIDAD ────────────────────────────────────────────────────
      - alert: ServicioNoDisponible
        expr: up{job="mi-api"} == 0
        for: 1m
        labels:
          severity: critical
          team: backend
        annotations:
          summary: "mi-api no está disponible"
          description: "El servicio {{ $labels.instance }} lleva {{ $value }}s caído"
          runbook: "https://wiki/runbooks/servicio-caido"

      # ── ERROR RATE ─────────────────────────────────────────────────────────
      - alert: TasaErrorAlta
        expr: |
          (
            sum(rate(http_requests_total{job="mi-api", status=~"5.."}[5m]))
            /
            sum(rate(http_requests_total{job="mi-api"}[5m]))
          ) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Error rate > 5% en mi-api"
          description: "Error rate actual: {{ humanizePercentage $value }}"

      # ── LATENCIA ───────────────────────────────────────────────────────────
      - alert: LatenciaP95Alta
        expr: |
          histogram_quantile(0.95,
            sum by (le) (rate(http_request_duration_seconds_bucket{job="mi-api"}[5m]))
          ) > 1.0
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "P95 latencia > 1s en mi-api"

      # ── RECURSOS ───────────────────────────────────────────────────────────
      - alert: MemoriaAltaPod
        expr: |
          container_memory_working_set_bytes{container!=""}
          / container_spec_memory_limit_bytes{container!=""} > 0.9
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Pod {{ $labels.pod }} usando >90% de memoria"
```

### Alertmanager — Routing de Alertas

```yaml
# alertmanager.yml
global:
  slack_api_url: "https://hooks.slack.com/services/..."

route:
  receiver: "equipo-backend"  # receptor por defecto
  group_by: ["alertname", "team"]
  group_wait: 30s       # espera para agrupar alertas relacionadas
  group_interval: 5m    # intervalo para reenviar alertas activas
  repeat_interval: 4h   # repetir si la alerta sigue activa

  routes:
    # Alertas críticas → PagerDuty
    - match:
        severity: critical
      receiver: "pagerduty-backend"
      continue: true  # también continúa a las siguientes rutas

    # Alertas de DB → equipo de datos
    - match:
        team: datos
      receiver: "slack-datos"

receivers:
  - name: "equipo-backend"
    slack_configs:
      - channel: "#alertas-backend"
        text: |
          *{{ .GroupLabels.alertname }}*
          {{ range .Alerts }}
          • {{ .Annotations.summary }}
          {{ end }}

  - name: "pagerduty-backend"
    pagerduty_configs:
      - routing_key: "${PAGERDUTY_ROUTING_KEY}"
        description: "{{ .GroupLabels.alertname }}"

inhibit_rules:
  # Si el servicio está caído, no disparar alertas de latencia/errores
  - source_match:
      alertname: ServicioNoDisponible
    target_match_re:
      alertname: (TasaErrorAlta|LatenciaP95Alta)
    equal: [job]
```

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - El **Kubernetes Service Discovery** (`kubernetes_sd_configs`) descubre pods automáticamente — solo necesitas anotar los pods con `prometheus.io/scrape: "true"`.
> - Las reglas de alerta van en ficheros `rules/*.yaml` separados del `prometheus.yml` — facilita su gestión y versionado en Git.
> - El campo `for:` es fundamental — evita falsas alarmas por spikes momentáneos. Mínimo `for: 5m` para alertas críticas.
> - Las **inhibit_rules** de Alertmanager evitan el spam de alertas cuando un servicio está caído — no quieres 50 alertas de latencia cuando el servicio ya está caído.

### Recursos
- 🌐 prometheus.io/docs/alerting/latest/alertmanager
- 🌐 prometheus.io/docs/prometheus/latest/configuration/configuration
- 🌐 monitoring.mixins.dev — mixins de alertas predefinidos para servicios comunes

---
`#prometheus` `#alertmanager` `#configuracion` `#alertas`
