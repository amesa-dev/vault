# 🏛️ Diseño de Sistemas 01 — El Método y Estimaciones

[[Desarrollo Profesional/Diseño de Sistemas/Diseño de Sistemas|⬅️ Volver a Diseño de Sistemas]] | [[Desarrollo Profesional/Diseño de Sistemas/Páginas/02 - Escalar de Cero a Millones|02 →]]

> [!abstract] Introducción
> Diseñar un sistema —en una entrevista o en la vida real— no es escupir una arquitectura de memoria, es un **proceso**: aclarar qué se pide, estimar la escala con números, proponer un diseño de alto nivel, profundizar donde importa y discutir los cuellos de botella. Esta página establece ese método (el framework de 4 pasos de Alex Xu) y la herramienta que sostiene todas las decisiones: las **estimaciones back-of-the-envelope**, esos cálculos rápidos que te dicen si necesitas una base de datos o cien, una caché o un clúster de cachés.

## ¿De qué vamos a hablar?

El proceso estructurado para abordar cualquier diseño de sistema y cómo estimar su escala con cálculos rápidos.

### Conceptos que vamos a cubrir
- El framework de 4 pasos
- Requisitos funcionales vs no funcionales
- Back-of-the-envelope: QPS, almacenamiento, ancho de banda
- Los números de latencia que todo ingeniero debe conocer
- Errores comunes al estimar

---

## El Concepto

### El Framework de 4 Pasos

Lo peor que puedes hacer es lanzarte a dibujar cajas. Sigue un proceso (Alex Xu lo estructura así para entrevistas de ~45 min, pero sirve igual para diseño real):

**1. Entender el problema y acotar el alcance (≈5 min).** No asumas; pregunta. ¿Qué features exactamente? ¿Cuántos usuarios? ¿Lectura o escritura intensiva? ¿Qué escala? ¿Qué es lo crítico (latencia, consistencia, disponibilidad)? Un buen diseño empieza por **acotar**: es imposible diseñar "Twitter entero" en 45 minutos; diseñas las 2-3 features núcleo.

**2. Diseño de alto nivel y acuerdo (≈10-15 min).** Dibuja los componentes principales (clientes, API gateway, servicios, bases de datos, cachés) y el flujo de datos. Haz una estimación back-of-the-envelope para dimensionar. Acuerda este esqueleto antes de profundizar.

**3. Diseño en profundidad (≈10-15 min).** Profundiza en los componentes que más importan o que el entrevistador (o los requisitos) señalen: el esquema de datos, el algoritmo clave, cómo se particiona, cómo se cachea. Aquí es donde demuestras criterio.

**4. Cierre: cuellos de botella y mejoras (≈5 min).** ¿Dónde se rompe esto a 10× la escala? ¿Single points of failure? ¿Qué monitorizas? ¿Cómo manejas fallos? Discute los compromisos que tomaste.

### Requisitos: Funcionales vs No Funcionales

La distinción que ordena el paso 1:

- **Funcionales** (qué hace): "el usuario puede acortar una URL y ser redirigido". Definen las features.
- **No funcionales** (cómo de bien): escala, latencia, disponibilidad, consistencia, durabilidad. **Aquí se juega el diseño.** "100M de URLs nuevas al día", "redirección en <100ms", "99,99% de disponibilidad", "puede tolerar consistencia eventual". Los no funcionales determinan la arquitectura mucho más que los funcionales — un servicio para 100 usuarios y otro para 100 millones hacen lo mismo funcionalmente y son sistemas radicalmente distintos.

### Back-of-the-Envelope: Estimar la Escala

Cálculos rápidos con órdenes de magnitud para dimensionar. No buscas precisión, buscas saber si hablas de 1 servidor o de 1.000. Las magnitudes que estimas:

**QPS (Queries Per Second).** Del número de usuarios y su actividad:

```
Ejemplo: red social con 100M usuarios activos diarios (DAU),
cada uno publica 2 veces al día.

Escrituras/día = 100M × 2 = 200M
QPS escritura  = 200M / 86.400 s ≈ 2.300 QPS
QPS pico       ≈ 2× media ≈ 4.600 QPS

Si la ratio lectura:escritura es 100:1 (típico en redes sociales):
QPS lectura    ≈ 230.000 QPS  → esto define que necesitas caché y réplicas
```

Atajo útil: **un día ≈ 86.400 s ≈ 10⁵ s**. Así, "N eventos al día" → "N / 100.000 QPS" mentalmente.

**Almacenamiento.** Del volumen × tamaño × tiempo de retención:

```
200M posts/día × 1 KB/post     = 200 GB/día
× 365 días × 5 años             ≈ 365 TB  (solo texto)
Si hay imágenes (200 KB, 10% de posts): suma órdenes de magnitud más.
→ esto define si cabe en una BD o necesitas sharding + object storage
```

**Ancho de banda.** QPS × tamaño de payload. **Memoria de caché.** Aplica la regla 80/20: el 20% de los datos genera el 80% de las peticiones, así que estima cachear ese 20% caliente.

```
Caché del feed: 230.000 QPS lectura × 1 KB ≈ 230 MB/s de lectura servida.
Datos calientes a cachear ≈ 20% del working set diario.
```

### Los Números de Latencia que Hay que Saber

Las "Latency Numbers Every Programmer Should Know" (Jeff Dean) — interiorízalas en órdenes de magnitud, porque determinan qué es caro:

| Operación | Latencia aprox. | Intuición |
|-----------|-----------------|-----------|
| Referencia a caché L1 | ~1 ns | — |
| Acceso a memoria RAM | ~100 ns | rapidísimo |
| Lectura secuencial 1 MB de RAM | ~250 µs | — |
| Round-trip dentro del datacenter | ~500 µs | red local |
| Lectura aleatoria en SSD | ~100 µs | — |
| Lectura secuencial 1 MB de SSD | ~1 ms | — |
| Seek en disco duro (HDD) | ~10 ms | **lento** |
| Round-trip de red intercontinental | ~150 ms | **muy lento** |

Las lecciones que se derivan:
- **La memoria es ~100.000× más rápida que el disco duro** → por eso existen las cachés en RAM (Redis).
- **Una llamada de red intercontinental (~150 ms) es enorme** → por eso existen las CDN y los multi-datacenter: acercar los datos al usuario.
- **Evita el disco en el camino crítico**; evita idas y vueltas de red innecesarias (un N+1 de queries que cruza la red 100 veces es 100 round-trips).

### Errores Comunes al Estimar

- **No redondear**: trabaja con potencias de 10 y unidades redondas (1 KB, no 1.024 B). Buscas la magnitud, no el decimal.
- **Olvidar el pico**: el tráfico no es uniforme; dimensiona para el pico (2-10× la media), no para la media.
- **Ignorar la ratio lectura/escritura**: casi todos los sistemas son mucho más de lectura; eso decide si necesitas réplicas y caché.
- **Calcular sin propósito**: cada estimación debe llevar a una decisión de diseño ("230k QPS de lectura → necesito caché"). Si un número no cambia ninguna decisión, no lo calcules.

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - Sigue el **framework de 4 pasos**: (1) acotar el problema preguntando, (2) diseño de alto nivel + estimación acordados, (3) profundizar donde importa, (4) cuellos de botella y mejoras. No empieces por dibujar cajas.
> - Separa **requisitos funcionales** (qué hace) de **no funcionales** (escala, latencia, disponibilidad, consistencia). Los no funcionales determinan la arquitectura.
> - **Back-of-the-envelope**: estima QPS (usuarios × actividad / 10⁵ s), almacenamiento (volumen × tamaño × retención), ancho de banda y memoria de caché (regla 80/20). Busca órdenes de magnitud, no precisión.
> - Interioriza los **números de latencia**: RAM ~100.000× más rápida que HDD (→ cachés), red intercontinental ~150 ms (→ CDN/multi-DC). Evita disco y round-trips en el camino crítico.
> - Cada estimación debe **llevar a una decisión**. Dimensiona para el **pico** (2-10× la media) y considera la **ratio lectura/escritura**.

### Para llevar a la práctica
- [ ] Coge un producto que uses y estima su QPS de lectura y escritura con datos públicos de usuarios.
- [ ] Memoriza los órdenes de magnitud de la tabla de latencias (RAM vs SSD vs HDD vs red).
- [ ] Practica el atajo "eventos al día / 10⁵ = QPS" hasta hacerlo mental.
- [ ] En tu próximo diseño, escribe explícitamente los requisitos no funcionales antes de dibujar nada.

### Recursos
- 📖 *System Design Interview – An Insider's Guide, Vol. 1* — Alex Xu (capítulos 2 y 3)
- 🌐 colin-scott.github.io/personal_website/research/interactive_latency.html — latencias interactivas por año
- 🌐 github.com/donnemartin/system-design-primer — recurso libre exhaustivo
- 📄 "Numbers Everyone Should Know" — Jeff Dean (Google)

---
`#diseño-de-sistemas` `#estimaciones` `#metodo` `#latencia`
