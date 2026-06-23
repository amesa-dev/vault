# 🏛️ Caso de Estudio — Sistema de Notificaciones

[[Desarrollo Profesional/Diseño de Sistemas/Diseño de Sistemas|⬅️ Volver a Diseño de Sistemas]] | [[Desarrollo Profesional/Diseño de Sistemas/Páginas/08 - Caso Chat en Tiempo Real|← 08]] | [[Desarrollo Profesional/Diseño de Sistemas/Páginas/10 - Caso Autocompletado|10 →]]

> [!abstract] Introducción
> Un sistema de notificaciones envía mensajes a los usuarios por varios canales —push móvil, SMS, email, in-app— disparados por eventos del sistema. Es un caso que enseña mucho sobre **desacoplo con colas, integración con terceros poco fiables y fiabilidad de entrega**: tienes que hablar con APNs, FCM, proveedores de SMS y email que fallan, limitan y tienen sus propias reglas, sin que un proveedor lento tumbe tu sistema.

## ¿De qué vamos a hablar?

El diseño de un sistema de notificaciones multicanal: la arquitectura desacoplada, la integración con proveedores externos y la fiabilidad de entrega.

### Conceptos que vamos a cubrir
- Requisitos y los canales (push, SMS, email, in-app)
- Arquitectura desacoplada con colas
- Integración con proveedores de terceros
- Fiabilidad: reintentos, dedup, idempotencia
- Preferencias, rate limiting y plantillas

---

## El Diseño

### 1. Requisitos

**Funcionales**: enviar notificaciones por **push** (APNs para iOS, FCM para Android), **SMS** (Twilio…), **email** (SendGrid/SES) e **in-app**. Disparadas por eventos de servicios. Respetar las preferencias del usuario (qué canales, qué tipos).

**No funcionales**: **alta fiabilidad** (no perder notificaciones importantes), escala (miles de millones al día), evitar **spam/duplicados**, y baja latencia para las urgentes.

### 2. Arquitectura Desacoplada

La clave del diseño es **desacoplar** la generación de la entrega con **colas** ([[Desarrollo Profesional/Sistemas Distribuidos/Páginas/02 - Mensajería y Brokers|Mensajería]]), porque la entrega depende de terceros lentos e impredecibles.

```
[Servicios] → [API de Notificaciones] → valida, dedup, aplica preferencias
                                              │
                                        [Cola por canal]
                              ┌───────────────┼───────────────┐
                          [Workers push]  [Workers SMS]   [Workers email]
                              │                │                │
                           [APNs/FCM]      [Twilio]        [SendGrid]
```

- Una **API/servicio de notificaciones** recibe la petición de enviar, valida, comprueba preferencias y deduplica.
- Encola por canal (colas separadas: un proveedor de SMS caído no bloquea los emails — es un **bulkhead**, ver [[Desarrollo Profesional/Patrones de Resiliencia/Páginas/02 - Circuit Breaker y Aislamiento|Resiliencia]]).
- **Workers** por canal consumen su cola y hablan con el proveedor externo correspondiente.

Beneficios: absorbe picos (buffer), aísla fallos por canal, y escala cada canal independientemente.

### 3. Integración con Terceros

Los proveedores externos son la parte frágil. Hay que tratarlos con todo el arsenal de [[Desarrollo Profesional/Patrones de Resiliencia/Patrones de Resiliencia|resiliencia]]:
- **Timeouts** y **reintentos con backoff** ante fallos transitorios.
- **Circuit breaker** por proveedor: si APNs está caído, deja de intentarlo un rato.
- **Rate limiting**: los proveedores imponen límites; respétalos o te bloquean.
- **Gestión de tokens de dispositivo**: los tokens de push caducan o se invalidan; hay que actualizarlos y limpiar los muertos.

### 4. Fiabilidad de Entrega

- **At-least-once + idempotencia**: para no perder notificaciones, reintentas; pero reintentar puede **duplicar** (enviar el mismo push dos veces). Se deduplica con una **clave de idempotencia** por notificación y una tabla de "ya enviadas" ([[Desarrollo Profesional/Sistemas Distribuidos/Páginas/03 - Transacciones Distribuidas|idempotencia]]).
- **Dead Letter Queue**: notificaciones que fallan repetidamente van a una DLQ para inspección, sin bloquear la cola.
- **Seguimiento de estado**: registrar enviada/entregada/fallida (los proveedores devuelven callbacks/webhooks de entrega).

### 5. Preferencias, Rate Limiting y Plantillas

- **Preferencias del usuario**: qué canales acepta y para qué tipos de evento (un servicio de settings consultado antes de encolar). Respetar el opt-out es legal y de producto.
- **Rate limiting / agrupación**: no bombardear (10 notificaciones de golpe → agrúpalas en una; "tienes 10 mensajes nuevos"). Evita la fatiga de notificaciones.
- **Plantillas**: el contenido se genera de plantillas con variables (no se hardcodea), versionadas y traducibles.
- **Prioridades**: una cola/flujo prioritario para lo urgente (OTP de login) frente a lo informativo (newsletter).

### 6. Cuellos de Botella y Mejoras

- **Proveedor lento/caído**: aislado por cola+circuit breaker; las notificaciones se acumulan en su cola sin afectar a otros canales.
- **Picos masivos** (un evento que notifica a millones): la cola absorbe; los workers drenan a ritmo sostenible y respetando límites de proveedor.
- **Analíticas de entrega**: tasas de apertura, entrega y fallo por canal para optimizar.

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - Multicanal (push APNs/FCM, SMS, email, in-app) disparado por eventos. Reto: **fiabilidad** e integración con terceros poco fiables a gran escala.
> - **Desacopla** generación y entrega con **colas por canal**: un proveedor caído no bloquea los demás (bulkhead), absorbe picos y escala por canal.
> - Trata a los proveedores externos con **resiliencia**: timeouts, reintentos con backoff, **circuit breaker por proveedor**, respeto de sus rate limits, y gestión de tokens caducados.
> - **At-least-once + idempotencia** (clave por notificación + tabla de enviadas) para no perder ni duplicar; **DLQ** para los fallos persistentes.
> - Respeta **preferencias/opt-out**, agrupa para evitar fatiga, usa **plantillas** versionadas y un **flujo prioritario** para lo urgente (OTP) frente a lo informativo.

### Para llevar a la práctica
- [ ] Dibuja el flujo desde "un servicio quiere notificar" hasta "el push llega al móvil", con las colas por canal.
- [ ] Razona cómo evitas enviar el mismo push dos veces cuando reintentas por un timeout.
- [ ] Diseña el aislamiento: si Twilio se cae, ¿por qué no debe afectar a los emails?
- [ ] Define un esquema de preferencias de usuario y dónde se consulta en el flujo.

### Recursos
- 📖 *System Design Interview, Vol. 1* — Alex Xu (capítulo 10: Design a Notification System)
- 🌐 firebase.google.com/docs/cloud-messaging — FCM
- 📄 Conexión con [[Desarrollo Profesional/Patrones de Resiliencia/Patrones de Resiliencia|Resiliencia]] y [[Desarrollo Profesional/Sistemas Distribuidos/Páginas/02 - Mensajería y Brokers|Mensajería]]

---
`#diseño-de-sistemas` `#notificaciones` `#colas` `#caso-estudio`
