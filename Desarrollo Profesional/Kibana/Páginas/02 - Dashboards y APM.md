# 🧭 Kibana 02 — Dashboards y APM

[[Desarrollo Profesional/Kibana/Kibana|⬅️ Volver a Kibana]] | [[Desarrollo Profesional/Kibana/Páginas/01 - Discover y KQL|← 01]]

> [!abstract] Introducción
> Los dashboards de Kibana combinan visualizaciones de logs y métricas en una vista unificada. APM (Application Performance Monitoring) es el módulo que integra trazas distribuidas directamente en Kibana — sin necesidad de Jaeger o Zipkin separados. Para el stack ELK completo, APM es el equivalente de Tempo en el stack de Grafana.

---

## El Concepto

### Dashboards de Logs

```
Estructura recomendada de un dashboard de logs:

Row 1: Overview
├── Total de logs (Metric)
├── Error rate (%)          → Count(status=error) / Count(all) * 100
├── Logs por nivel          → Donut chart (debug/info/warn/error)
└── Tendencia en el tiempo  → Bar chart temporal

Row 2: Errores
├── Top 10 mensajes de error     → Data table
├── Errores por servicio         → Bar chart agrupado por kubernetes.labels.app
└── Error spike detector         → Anomaly detection (ML Job)

Row 3: Performance
├── Latencia media por endpoint  → Data table
├── P95 por servicio             → Line chart
└── Throughput (requests/min)    → Bar chart
```

```
Crear dashboard desde cero:
1. Dashboard → Create dashboard
2. Add panel → Lens (arrastrar y soltar)
3. Configurar cada panel:
   - Seleccionar índice (logstash-*, filebeat-*, etc.)
   - Definir el eje X (time) y el eje Y (count, sum, avg...)
   - Filtrar con KQL
4. Añadir controles de filtrado (Controls):
   - Service selector: permite filtrar todo el dashboard por servicio
   - Time range: ajustar el rango temporal
5. Guardar y compartir URL
```

### Kibana APM — Application Performance Monitoring

APM integra trazas distribuidas directamente en Kibana usando los agentes de Elastic APM:

```python
# Instrumentar una aplicación Python/FastAPI con Elastic APM
from elasticapm.contrib.starlette import make_apm_app
import elasticapm

# Configuración del agente APM
apm_config = {
    "SERVICE_NAME": "mi-api",
    "SECRET_TOKEN": "tu-secret-token",
    "SERVER_URL": "http://apm-server:8200",
    "ENVIRONMENT": "production",
    "CAPTURE_BODY": "errors",  # captura body en errores
    "TRANSACTION_SAMPLE_RATE": 0.1,  # samplea el 10% de transacciones
}

# Añadir middleware APM a FastAPI
from fastapi import FastAPI
app = FastAPI()
apm = make_apm_app(apm_config)
app.add_middleware(apm)

# Spans personalizados para medir operaciones específicas
from elasticapm import capture_span

async def procesar_pedido(pedido_id: str):
    with capture_span("validar_stock", span_type="custom"):
        stock = await verificar_stock(pedido_id)

    with capture_span("consulta_bd", span_type="db.postgresql.query"):
        resultado = await db.fetch("SELECT * FROM pedidos WHERE id = $1", pedido_id)

    return resultado
```

```
Kibana APM te da:
├── Service Map     → mapa visual de dependencias entre servicios
├── Transactions    → distribución de latencias por endpoint
├── Errors          → errores agrupados por tipo con stack traces
├── Metrics         → CPU, memoria, GC de la JVM (si aplica)
└── Traces          → trazas individuales con spans detallados

Correlación automática:
← Un log con trace.id → puede abrirse directamente como traza en APM
← Una traza lenta → puede ver los logs del pod en ese momento
```

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - Los dashboards de Kibana son más potentes cuando combinan múltiples índices (logs + APM + métricas de sistema) en una vista unificada.
> - **Elastic APM** instrumenta con un agente que captura automáticamente las trazas de HTTP, BD y colas — sin modificar el código de negocio.
> - La correlación entre logs y trazas (`trace.id`) es la herramienta más potente del stack ELK — permite ir del log de error directamente a la traza completa.
> - El **Transaction Sample Rate** debe estar entre 0.1 (10%) y 0.01 (1%) en producción alta para no saturar el APM server.

### Recursos
- 🌐 elastic.co/guide/en/apm/agent/python/current/getting-started.html
- 🌐 elastic.co/guide/en/kibana/current/dashboard.html

---
`#kibana` `#apm` `#dashboards` `#trazas` `#elk`
