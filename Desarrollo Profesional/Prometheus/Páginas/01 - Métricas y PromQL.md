# 🔥 Prometheus 01 — Métricas y PromQL

[[Desarrollo Profesional/Prometheus/Prometheus|⬅️ Volver a Prometheus]] | [[Desarrollo Profesional/Prometheus/Páginas/02 - Configuración y Alertas|02 →]]

> [!abstract] Introducción
> Prometheus tiene cuatro tipos de métricas con semánticas bien definidas — usar el tipo correcto es la diferencia entre un dashboard con información útil y uno con datos sin sentido. PromQL es el lenguaje de consultas que permite extraer señal del ruido: calcular tasas, percentiles y comparaciones entre instancias.

---

## El Concepto

### Los 4 Tipos de Métricas

```python
from prometheus_client import Counter, Gauge, Histogram, Summary, start_http_server
import time

# COUNTER — solo crece. Para contar eventos.
# Nunca uses Gauge para contar eventos que no disminuyen
peticiones_total = Counter(
    "http_requests_total",
    "Número total de peticiones HTTP",
    labelnames=["method", "path", "status"]
)

# Uso: incrementar al procesar cada petición
peticiones_total.labels(method="GET", path="/api/pedidos", status="200").inc()

# GAUGE — puede subir y bajar. Para valores de estado.
conexiones_activas = Gauge(
    "db_connections_active",
    "Conexiones de BD activas",
    labelnames=["database"]
)
conexiones_activas.labels(database="produccion").set(42)
conexiones_activas.labels(database="produccion").inc()  # también válido
conexiones_activas.labels(database="produccion").dec()

# HISTOGRAM — distribución de valores (latencias, tamaños).
# Crucial: los buckets deben cubrir el rango esperado de valores
latencia_peticion = Histogram(
    "http_request_duration_seconds",
    "Duración de las peticiones HTTP",
    labelnames=["method", "path"],
    buckets=[0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0]
)

# Uso con context manager
with latencia_peticion.labels(method="GET", path="/api/pedidos").time():
    resultado = procesar_peticion()

# SUMMARY — similar a Histogram pero calcula percentiles en el cliente
# Problema: no se pueden agregar entre instancias — PREFERIR Histogram
latencia_summary = Summary(
    "rpc_duration_seconds",
    "Duración de llamadas RPC"
)
```

```python
# Instrumentar una aplicación FastAPI
from fastapi import FastAPI, Request, Response
from prometheus_client import Counter, Histogram, make_asgi_app
import time

app = FastAPI()

REQUEST_COUNT = Counter("http_requests_total", "Total HTTP requests", ["method", "endpoint", "status"])
REQUEST_LATENCY = Histogram("http_request_duration_seconds", "HTTP request latency", ["method", "endpoint"])

@app.middleware("http")
async def metrics_middleware(request: Request, call_next):
    start = time.perf_counter()
    response = await call_next(request)
    duration = time.perf_counter() - start

    REQUEST_COUNT.labels(
        method=request.method,
        endpoint=request.url.path,
        status=response.status_code
    ).inc()

    REQUEST_LATENCY.labels(
        method=request.method,
        endpoint=request.url.path
    ).observe(duration)

    return response

# Endpoint de métricas para Prometheus
metrics_app = make_asgi_app()
app.mount("/metrics", metrics_app)
```

### PromQL — El Lenguaje de Consultas

```promql
# ── SELECTORES BÁSICOS ────────────────────────────────────────────────────────
# Todas las series de una métrica
http_requests_total

# Filtrar por label (exacto)
http_requests_total{status="200"}

# Filtrar por label (regex)
http_requests_total{status=~"2.."}    # empieza por 2
http_requests_total{status!~"5.."}   # que NO sean errores 5xx

# ── FUNCIONES DE RATE ─────────────────────────────────────────────────────────
# rate() — tasa por segundo para Counters (usa los últimos N minutos)
rate(http_requests_total[5m])   # peticiones/segundo en los últimos 5m

# irate() — tasa instantánea (más reactiva pero más ruidosa)
irate(http_requests_total[5m])

# increase() — incremento total en el periodo
increase(http_requests_total[1h])  # peticiones totales en la última hora

# ── AGREGACIONES ────────────────────────────────────────────────────────────────
# sum() — suma todas las series
sum(rate(http_requests_total[5m]))

# sum by — suma agrupando por label
sum by (status) (rate(http_requests_total[5m]))

# sum without — suma sin agrupar por ese label
sum without (instance) (rate(http_requests_total[5m]))

# topk y bottomk
topk(5, sum by (endpoint) (rate(http_requests_total[5m])))

# ── PERCENTILES CON HISTOGRAM ─────────────────────────────────────────────────
# P50 (mediana)
histogram_quantile(0.5, sum by (le) (rate(http_request_duration_seconds_bucket[5m])))

# P95
histogram_quantile(0.95, sum by (le) (rate(http_request_duration_seconds_bucket[5m])))

# P99 por endpoint
histogram_quantile(0.99,
  sum by (le, endpoint) (rate(http_request_duration_seconds_bucket[5m]))
)

# ── MÉTRICAS DERIVADAS ÚTILES ────────────────────────────────────────────────
# Error rate (%)
sum(rate(http_requests_total{status=~"5.."}[5m]))
/
sum(rate(http_requests_total[5m]))
* 100

# Latencia media
sum(rate(http_request_duration_seconds_sum[5m]))
/
sum(rate(http_request_duration_seconds_count[5m]))

# Disponibilidad (uptime como %)
avg_over_time(up{job="mi-servicio"}[24h]) * 100
```

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - **Counter**: para eventos que solo crecen. Usa `rate()` para calcular la tasa, nunca graficar el valor crudo.
> - **Gauge**: para valores de estado (conexiones activas, uso de memoria). Se grafican directamente.
> - **Histogram**: para latencias y tamaños. Usa `histogram_quantile()` para calcular percentiles. Los buckets deben cubrir el rango esperado de valores.
> - `rate()` necesita al menos dos muestras en el rango — la ventana debe ser mayor que el intervalo de scraping.
> - Para agregar percentiles entre instancias, **debes usar Histogram** (no Summary).

### Recursos
- 🌐 prometheus.io/docs/concepts/metric_types
- 🌐 prometheus.io/docs/prometheus/latest/querying/basics
- 🌐 promlabs.com/promql-explained — visual explicado de PromQL

---
`#prometheus` `#promql` `#metricas` `#counter` `#histogram`
