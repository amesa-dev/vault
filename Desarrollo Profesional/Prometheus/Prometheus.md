# 🔥 Prometheus

[[Desarrollo Profesional/Inicio Profesional|⬅️ Volver a Desarrollo Profesional]]

> [!abstract] Prometheus
> Sistema de monitoreo y alerta de código abierto basado en métricas de series temporales. Utiliza un modelo de recopilación mediante consultas HTTP (pull model) a endpoints expuestos por los servicios.

---

## 🏗️ Tipos de Métricas en Prometheus

1. **Counter (Contador):**
   - Valor monótonamente creciente (solo sube o vuelve a cero al reiniciarse).
   - *Ejemplo:* Número total de peticiones HTTP completadas.
2. **Gauge (Indicador):**
   - Valor que puede subir y bajar de forma arbitraria.
   - *Ejemplo:* Uso de memoria actual o temperatura de CPU.
3. **Histogram (Histograma):**
   - Mide la duración del evento o tamaño del mensaje dividiéndolo en rangos configurables (buckets).
   - *Ejemplo:* Latencia de solicitudes.
4. **Summary (Resumen):**
   - Similar al histograma, pero calcula cuantiles configurables sobre una ventana de tiempo deslizante.

---

## ⚡ Sintaxis Básica de PromQL (Prometheus Query Language)

### 1. Consultas Simples
```promql
# Obtener el valor actual de la métrica
http_requests_total

# Filtrar por etiquetas (labels)
http_requests_total{status="200", method="GET"}
```

### 2. Rangos y Funciones de Ratio
```promql
# Tasa de peticiones por segundo en los últimos 5 minutos
rate(http_requests_total[5m])

# Latencia media de peticiones
sum(rate(http_request_duration_seconds_sum[5m])) / sum(rate(http_request_duration_seconds_count[5m]))
```

---
`#prometheus` `#monitoreo` `#devops` `#apuntes`
