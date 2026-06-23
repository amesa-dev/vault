# 🏛️ Caso de Estudio — Web Crawler

[[Desarrollo Profesional/Diseño de Sistemas/Diseño de Sistemas|⬅️ Volver a Diseño de Sistemas]] | [[Desarrollo Profesional/Diseño de Sistemas/Páginas/11 - Caso Generador de IDs|← 11]] | [[Desarrollo Profesional/Diseño de Sistemas/Páginas/13 - Caso YouTube y Streaming|13 →]]

> [!abstract] Introducción
> Un web crawler (la araña de un buscador) recorre la web descargando páginas para indexarlas. Es un caso fascinante porque es un sistema distribuido de gran escala que debe ser **educado** (no tumbar los sitios que visita), **robusto** (la web está llena de trampas y contenido malformado) y **eficiente** (miles de millones de páginas). Enseña BFS a escala, deduplicación masiva y la cortesía hacia servidores ajenos.

## ¿De qué vamos a hablar?

El diseño de un crawler a escala web: el bucle de rastreo, la frontera de URLs, la deduplicación y las trampas a evitar.

### Conceptos que vamos a cubrir
- Requisitos: escala, frescura, cortesía
- El bucle BFS y la "frontera de URLs"
- Cortesía: robots.txt y rate limiting por dominio
- Deduplicación de URLs y de contenido
- Trampas: spider traps, contenido dinámico, robustez

---

## El Diseño

### 1. Requisitos

**Funcionales**: dado un conjunto de URLs semilla, descargar sus páginas, extraer los enlaces, y repetir; almacenar el contenido para indexar.

**No funcionales**: **escalable** (miles de millones de páginas), **educado** (no sobrecargar ningún servidor), **robusto** (sobrevivir a HTML roto, trampas, enlaces infinitos), **fresco** (revisitar para detectar cambios) y **extensible**.

**Estimación**: 1.000M páginas/mes → ~400 páginas/seg de media, picos mucho mayores; petabytes de almacenamiento.

### 2. El Bucle BFS y la Frontera de URLs

Rastrear la web es esencialmente un **BFS** (recorrido en anchura) sobre el grafo de páginas, donde las aristas son los hiperenlaces:

```
[URLs semilla] → cola "frontera" → 
  tomar URL → descargar HTML → extraer enlaces → 
  filtrar (¿ya vista? ¿permitida?) → añadir nuevas a la frontera → repetir
```

La pieza central es la **URL Frontier**: la cola de URLs pendientes de rastrear. No es una cola simple; debe gestionar **dos prioridades en tensión**:
- **Prioridad** (qué rastrear antes): una página de noticias importante se rastrea antes y más a menudo que una oscura. Se implementan colas por prioridad.
- **Cortesía** (no martillear un dominio): aunque tengas 10.000 URLs de `wikipedia.org`, no las descargues todas a la vez. La frontera agrupa por dominio y limita el ritmo por dominio.

El diseño clásico de la frontera tiene dos niveles de colas: unas que priorizan y otras que garantizan que cada dominio se visita con un intervalo educado.

### 3. Cortesía

Un crawler agresivo es indistinguible de un ataque DoS. Reglas de buena vecindad:
- **`robots.txt`**: cada sitio publica qué se puede rastrear y qué no. El crawler **debe** leerlo y respetarlo (cachear el robots.txt por dominio).
- **Rate limiting por dominio**: un retardo entre peticiones al mismo host (p. ej. 1 req/seg por dominio), idealmente respetando `Crawl-delay`. Por eso la frontera agrupa por dominio.
- **User-Agent identificable**: decir quién eres, para que los administradores puedan contactarte o bloquearte.

### 4. Deduplicación

Dos niveles, ambos críticos a escala:
- **URLs ya vistas**: antes de añadir una URL a la frontera, comprueba si ya la rastreaste. Con miles de millones de URLs, guardar el set entero es caro → se usan **Bloom filters** (estructura probabilística que dice "seguro que no la he visto" o "probablemente sí", con poca memoria; ver [[Desarrollo Profesional/Redis/Páginas/02 - Patrones de Uso|el truco de cachear el "no existe"]]).
- **Contenido duplicado**: muchas URLs distintas sirven el mismo contenido (mirrors, parámetros irrelevantes). Se detecta con un **hash del contenido** (o *simhash* para casi-duplicados) para no indexar lo mismo mil veces.

### 5. Trampas y Robustez

La web es hostil; el crawler debe sobrevivir a:
- **Spider traps**: páginas que generan enlaces infinitos (un calendario con "mes siguiente" eterno, URLs con parámetros infinitos). Defensa: **límite de profundidad** y de número de URLs por dominio, detección de patrones.
- **Contenido malformado**: HTML roto, encodings raros, archivos enormes. El parser debe ser tolerante y con límites de tamaño/tiempo.
- **Contenido dinámico (JS)**: muchas páginas se renderizan con JavaScript; rastrearlas requiere un navegador headless (caro), así que se decide qué merece ese coste.
- **Fallos**: descargas que fallan, timeouts → reintentos con backoff ([[Desarrollo Profesional/Patrones de Resiliencia/Páginas/01 - Timeouts Reintentos y Backoff|Resiliencia]]) y luego abandonar.

### 6. Arquitectura y Escalado

```
[Frontera de URLs] → [Descargadores (muchos workers)] → [Parser/extractor de enlaces]
        ▲                       │                                │
        └── nuevas URLs ────────┴── contenido → [Almacén] ──→ [Indexador]
                                          │
                              [Dedup: Bloom filter + hash de contenido]
```

- **Descargadores distribuidos**: cientos de workers descargando en paralelo, coordinados por la frontera.
- **DNS**: resolver dominios es un cuello de botella sorprendente (millones de lookups) → caché de DNS.
- **Almacenamiento**: el contenido crudo va a almacenamiento de objetos (petabytes); los metadatos y el grafo de enlaces a sus propios almacenes.
- **Frescura**: las páginas se revisitan según cuánto cambian (una home de noticias a diario; una página estática rara vez), priorizado en la frontera.

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - Un crawler es un **BFS distribuido** sobre el grafo de la web. Su pieza central es la **URL Frontier**: la cola de pendientes que equilibra **prioridad** (qué rastrear antes) y **cortesía** (no martillear un dominio).
> - **Cortesía** es obligatoria: respetar `robots.txt`, limitar el ritmo por dominio (la frontera agrupa por host), e identificarse. Un crawler agresivo = un DoS.
> - **Deduplicación** en dos niveles: URLs ya vistas (con **Bloom filters** por la escala) y contenido duplicado (hash/simhash del contenido).
> - **Robustez** ante la web hostil: límites de profundidad para **spider traps** (enlaces infinitos), parser tolerante para HTML roto, decisión sobre contenido dinámico (JS, caro), y reintentos para fallos.
> - Escala con **descargadores distribuidos**, caché de **DNS** (cuello de botella inesperado), almacenamiento de objetos para el contenido, y **revisita según frecuencia de cambio** para la frescura.

### Para llevar a la práctica
- [ ] Diseña la URL Frontier: ¿cómo combinas colas de prioridad con el límite de ritmo por dominio?
- [ ] Implementa una deduplicación de URLs con un Bloom filter y razona el trade-off de falsos positivos.
- [ ] Identifica tres tipos de spider trap y cómo los detectarías/limitarías.
- [ ] Razona por qué el DNS puede ser un cuello de botella y cómo lo mitigarías.

### Recursos
- 📖 *System Design Interview, Vol. 1* — Alex Xu (capítulo 9: Design a Web Crawler)
- 🌐 robotstxt.org — el estándar robots.txt
- 📄 "Mercator: A Scalable, Extensible Web Crawler" — el diseño clásico de referencia
- 📄 Conexión con Bloom filters y [[Desarrollo Profesional/Patrones de Resiliencia/Patrones de Resiliencia|resiliencia]]

---
`#diseño-de-sistemas` `#web-crawler` `#bfs` `#caso-estudio`
