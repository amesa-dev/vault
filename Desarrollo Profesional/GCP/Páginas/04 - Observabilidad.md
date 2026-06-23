# ☁️ GCP 04 — Observabilidad

[[Desarrollo Profesional/GCP/GCP|⬅️ Volver a GCP]] | [[Desarrollo Profesional/GCP/Páginas/03 - Networking e IAM|← 03]]

> [!abstract] Introducción
> La observabilidad es la capacidad de entender el estado interno de un sistema a partir de sus salidas externas: logs, métricas y trazas. En GCP, Cloud Logging, Cloud Monitoring y Cloud Trace forman el stack de observabilidad nativo. Entender estas herramientas es lo que separa "el sistema está caído y no sé por qué" de "el servicio de pagos tardó 12 segundos porque la consulta de PostgreSQL no usó el índice".

## ¿De qué vamos a hablar?

Cloud Logging para consulta y análisis de logs, Cloud Monitoring para métricas y alertas, Cloud Trace para trazas distribuidas, y cómo instrumentar una aplicación Python.

---

## El Concepto

### Cloud Logging — Logs Estructurados

Cloud Logging recoge automáticamente los logs de todos los servicios de GCP. Para sacarle el máximo partido, escribe logs estructurados (JSON):

```python
import logging
import json
import sys

# Logger estructurado para Cloud Logging
class CloudLoggingFormatter(logging.Formatter):
    def format(self, record: logging.LogRecord) -> str:
        log_entry = {
            "severity": record.levelname,  # Cloud Logging espera "severity"
            "message": record.getMessage(),
            "logger": record.name,
        }
        # Campos adicionales del contexto
        if hasattr(record, "request_id"):
            log_entry["request_id"] = record.request_id
        if hasattr(record, "user_id"):
            log_entry["user_id"] = record.user_id
        if record.exc_info:
            log_entry["exception"] = self.formatException(record.exc_info)
        return json.dumps(log_entry)

# Configurar logger
handler = logging.StreamHandler(sys.stdout)
handler.setFormatter(CloudLoggingFormatter())
logger = logging.getLogger("mi-app")
logger.addHandler(handler)
logger.setLevel(logging.INFO)

# Uso en código
logger.info("Pedido procesado", extra={"request_id": "req-123", "pedido_id": "ped-456"})
logger.error("Error al cobrar", extra={"request_id": "req-124", "motivo": "tarjeta_caducada"})
```

```bash
# Consultas en Cloud Logging (Log Explorer) — usa el lenguaje de consulta de GCP

# Errores del servicio de pedidos en la última hora
resource.type="cloud_run_revision"
resource.labels.service_name="servicio-pedidos"
severity>=ERROR
timestamp>="2024-01-15T10:00:00Z"

# Buscar por campo JSON específico
jsonPayload.pedido_id="ped-456"

# Latencias altas — si logeas el tiempo de respuesta
jsonPayload.duracion_ms>5000

# Exportar logs a BigQuery para análisis histórico
gcloud logging sinks create mi-sink \
  bigquery.googleapis.com/projects/mi-proyecto/datasets/logs \
  --log-filter='resource.type="cloud_run_revision"'
```

### Cloud Monitoring — Métricas y Alertas

```python
# Instrumentación con OpenTelemetry (estándar abierto, funciona con Cloud Monitoring)
from opentelemetry import metrics
from opentelemetry.sdk.metrics import MeterProvider
from opentelemetry.exporter.cloud_monitoring import CloudMonitoringMetricsExporter

provider = MeterProvider(
    metric_readers=[CloudMonitoringMetricsExporter(project_id="mi-proyecto")]
)
metrics.set_meter_provider(provider)

meter = metrics.get_meter("mi-app")

# Crear métricas de negocio
pedidos_procesados = meter.create_counter(
    name="pedidos_procesados_total",
    description="Número de pedidos procesados",
    unit="1"
)

latencia_pago = meter.create_histogram(
    name="latencia_pago_ms",
    description="Tiempo de procesamiento de pagos en milisegundos",
    unit="ms"
)

# Uso en código
import time

def procesar_pago(pedido_id: str) -> bool:
    inicio = time.perf_counter()
    try:
        resultado = _ejecutar_cobro(pedido_id)
        pedidos_procesados.add(1, {"estado": "exito", "metodo": "tarjeta"})
        return resultado
    except Exception as e:
        pedidos_procesados.add(1, {"estado": "error", "metodo": "tarjeta"})
        raise
    finally:
        duracion_ms = (time.perf_counter() - inicio) * 1000
        latencia_pago.record(duracion_ms, {"metodo": "tarjeta"})
```

```bash
# Crear una alerta en Cloud Monitoring via gcloud
gcloud monitoring policies create \
  --policy-from-file=alertas/latencia-alta.yaml

# alertas/latencia-alta.yaml
# displayName: "Latencia de API alta"
# conditions:
#   - displayName: "P95 > 2s"
#     conditionThreshold:
#       filter: metric.type="custom.googleapis.com/latencia_pago_ms"
#       aggregations:
#         - alignmentPeriod: 60s
#           perSeriesAligner: ALIGN_PERCENTILE_95
#       comparison: COMPARISON_GT
#       thresholdValue: 2000
# notificationChannels:
#   - projects/mi-proyecto/notificationChannels/[CANAL_ID]
```

### Cloud Trace — Trazas Distribuidas

Las trazas muestran el flujo de una petición a través de múltiples servicios. Fundamentales para debug de latencias en sistemas con microservicios:

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.exporter.cloud_trace import CloudTraceSpanExporter

provider = TracerProvider()
provider.add_span_processor(
    BatchSpanProcessor(CloudTraceSpanExporter(project_id="mi-proyecto"))
)
trace.set_tracer_provider(provider)

tracer = trace.get_tracer("mi-app")

async def procesar_pedido(pedido_id: str) -> dict:
    with tracer.start_as_current_span("procesar_pedido") as span:
        span.set_attribute("pedido.id", pedido_id)

        with tracer.start_as_current_span("validar_stock"):
            stock_ok = await verificar_stock(pedido_id)
            span.set_attribute("stock.disponible", stock_ok)

        if stock_ok:
            with tracer.start_as_current_span("cobrar_pago"):
                resultado_pago = await cobrar(pedido_id)
                span.set_attribute("pago.exitoso", resultado_pago["ok"])

        return {"ok": True}
```

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - Escribe logs **estructurados** (JSON) con campo `severity` en lugar de texto plano — Cloud Logging los indexa y puedes filtrar por campos específicos.
> - Usa **OpenTelemetry** para métricas y trazas — es el estándar abierto. Cloud Monitoring y Cloud Trace tienen exporters nativos.
> - Define alertas basadas en métricas de **negocio** (pedidos fallidos, latencia P95) no solo de infraestructura (CPU, memoria).
> - Las **trazas distribuidas** son esenciales para diagnosticar latencias en sistemas con múltiples microservicios — muestran dónde se gasta el tiempo en cada petición.

### Recursos
- 🌐 cloud.google.com/logging/docs
- 🌐 cloud.google.com/monitoring/docs
- 🌐 opentelemetry.io/docs/languages/python — OpenTelemetry para Python

---
`#gcp` `#observabilidad` `#logging` `#monitoring` `#tracing`
