# 🏛️ Caso de Estudio — News Feed / Timeline

[[Desarrollo Profesional/Diseño de Sistemas/Diseño de Sistemas|⬅️ Volver a Diseño de Sistemas]] | [[Desarrollo Profesional/Diseño de Sistemas/Páginas/06 - Caso URL Shortener|← 06]] | [[Desarrollo Profesional/Diseño de Sistemas/Páginas/08 - Caso Chat en Tiempo Real|08 →]]

> [!abstract] Introducción
> El news feed —el muro de Facebook, el timeline de Twitter/X, el feed de Instagram— es uno de los casos más instructivos porque obliga a enfrentar de cara un compromiso fundamental: ¿precalculas el feed de cada usuario cuando alguien publica (fan-out on write), o lo construyes al vuelo cuando el usuario lo abre (fan-out on read)? La respuesta correcta —un híbrido— y el famoso "problema de las celebridades" lo convierten en un caso que enseña a pensar en patrones de acceso, no solo en componentes.

## ¿De qué vamos a hablar?

El diseño de un feed social: el flujo de publicación, las dos estrategias de fan-out y por qué la solución real es un híbrido.

### Conceptos que vamos a cubrir
- Requisitos: las dos operaciones (publicar y leer el feed)
- Fan-out on write (push): precalcular feeds
- Fan-out on read (pull): construir al vuelo
- El problema de las celebridades y la solución híbrida
- Almacenamiento, caché y ranking

---

## El Diseño

### 1. Requisitos

**Funcionales**: un usuario publica contenido; el feed de un usuario muestra las publicaciones recientes de la gente que sigue, ordenadas (por tiempo o por relevancia).

**No funcionales**: el feed debe cargar **muy rápido** (es la pantalla principal, lectura-intensiva), y la publicación puede tolerar algo de retraso en propagarse (consistencia eventual aceptable: que un post tarde unos segundos en aparecer en los feeds está bien).

**Estimación**: 100M DAU, cada uno abre el feed varias veces → cientos de miles de lecturas de feed por segundo. **Lectura-intensivo de forma extrema.**

### 2. Las Dos Operaciones

El sistema tiene dos flujos que se diseñan casi por separado:
- **Feed publishing** (escritura): el usuario publica → el post se guarda → se propaga a los feeds de sus seguidores.
- **Feed retrieval** (lectura): el usuario abre la app → se le muestra su feed.

La pregunta central es **cuándo se hace el trabajo de juntar los posts de los seguidos**: al publicar (write) o al leer (read).

### 3. Fan-out on Write (Push)

Cuando alguien publica, **inmediatamente insertas ese post en el feed precalculado de cada uno de sus seguidores** (cada usuario tiene una lista/caché con su feed listo).

```
Ana publica → se escribe el post → se inserta en el feed-cache de
              los 500 seguidores de Ana (500 escrituras).
Cuando un seguidor abre la app → su feed YA está listo → lectura O(1), rapidísima.
```

- ✅ **Lectura instantánea**: el feed ya está montado. Perfecto para la operación más frecuente.
- ❌ **Escritura cara**: publicar implica N escrituras (una por seguidor). 
- ❌ **El problema de las celebridades**: si un usuario tiene 50M de seguidores, **un solo post dispara 50M de escrituras** (fan-out explosivo). Esto colapsa el sistema y desperdicia trabajo (muchos seguidores inactivos que nunca verán el post).
- ❌ Desperdicia recursos para usuarios inactivos (precalculas feeds que nadie mira).

### 4. Fan-out on Read (Pull)

Cuando el usuario abre el feed, **construyes el feed al vuelo**: buscas a quién sigue, traes sus posts recientes, los mezclas y ordenas.

```
Seguidor abre la app → buscar a quién sigue (500 personas) →
   traer los posts recientes de cada uno → mezclar y ordenar → mostrar.
```

- ✅ **Escritura barata**: publicar es solo un insert; sin fan-out. Sin problema de celebridades al publicar.
- ✅ No desperdicia: solo construyes feeds de usuarios activos que abren la app.
- ❌ **Lectura cara y lenta**: cada apertura hace muchas consultas y un merge en caliente. Para la operación más frecuente, es justo lo que no quieres.

### 5. La Solución: Híbrido

La realidad de producción combina ambos según el tipo de usuario, capturando lo mejor de cada uno:

- **Para usuarios normales** (la mayoría, con seguidores manejables): **fan-out on write**. Su feed se precalcula → lecturas instantáneas.
- **Para celebridades** (pocos usuarios, millones de seguidores): **fan-out on read**. Sus posts **no** se hacen push a millones de feeds (evitas la explosión). En su lugar, cuando un usuario abre su feed, el sistema **mezcla** su feed precalculado (de la gente normal que sigue) **con** los posts recientes de las celebridades que sigue, traídos al vuelo.

```
Feed de un usuario = (su feed precalculado por push, de cuentas normales)
                   + (posts recientes de las celebridades que sigue, traídos al leer)
                   → mezclado y ordenado
```

Así, la inmensa mayoría de las publicaciones (gente normal) se benefician de la lectura rápida del push, y las pocas cuentas problemáticas (celebridades) evitan el fan-out catastrófico usando pull solo para ellas. **Este patrón —elegir la estrategia según el perfil de acceso— es la lección transferible del caso.**

### 6. Almacenamiento, Caché y Ranking

- **Posts**: en una BD (los posts en sí). El **feed precalculado** de cada usuario es una lista de IDs de post, cacheada en **Redis** ([[Desarrollo Profesional/Redis/Redis|Redis]]) (una lista/sorted set por usuario). El feed guarda IDs, no posts completos: al renderizar, hidratas los posts desde su caché.
- **Fan-out asíncrono**: la propagación a los seguidores se hace con **workers** consumiendo una **cola** ([[Desarrollo Profesional/Sistemas Distribuidos/Páginas/02 - Mensajería y Brokers|Mensajería]]), no en el camino de la petición de publicar (que responde al instante).
- **Grafo social**: a quién sigue cada uno (relaciones) en su propio almacén, optimizado para "dame los seguidores de X" y "a quién sigue Y".
- **Ranking**: el orden puede ser cronológico (simple) o por **relevancia** (un modelo que puntúa cada post por engagement, afinidad, recencia…). El ranking por ML es donde compiten los feeds modernos, pero la infraestructura de fan-out es la misma.

### 7. Cuellos de Botella y Mejoras

- **Hotspots de celebridades**: resuelto con el híbrido; sus posts se cachean muy agresivamente (los pide muchísima gente).
- **Tamaño del feed cacheado**: no guardes infinito; mantén los N posts más recientes por usuario y pagina hacia atrás contra la BD.
- **Consistencia**: la propagación es eventual; un post puede tardar segundos en aparecer en todos los feeds — aceptable por requisitos.

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - El feed es **lectura-intensivo extremo**; la decisión central es **cuándo** se ensambla el feed: al publicar (write) o al leer (read).
> - **Fan-out on write (push)**: insertas el post en el feed precalculado de cada seguidor. Lectura instantánea, pero escritura cara y **explota con las celebridades** (millones de escrituras por post).
> - **Fan-out on read (pull)**: construyes el feed al abrir la app. Escritura barata, sin problema de celebridades, pero lectura lenta (lo contrario de lo que quieres para la operación frecuente).
> - **Solución híbrida**: push para usuarios normales (lectura rápida) + pull solo para celebridades (evita el fan-out catastrófico); al leer, **mezclas** ambos. La lección: elige la estrategia según el perfil de acceso.
> - El feed cacheado guarda **IDs de post** (Redis); el fan-out va por **cola asíncrona**; el grafo social en su almacén; el ranking puede ser cronológico o por ML sobre la misma infraestructura.

### Para llevar a la práctica
- [ ] Dibuja el flujo de publicar y de leer por separado; identifica dónde está el trabajo pesado en cada estrategia.
- [ ] Calcula el fan-out de una celebridad con 10M de seguidores: ¿cuántas escrituras genera un post? Entenderás por qué push puro no sirve.
- [ ] Diseña la estructura del feed cacheado en Redis (¿lista? ¿sorted set por timestamp?) y cómo lo paginas.
- [ ] Razona el umbral de "celebridad" a partir del cual cambias de push a pull.

### Recursos
- 📖 *System Design Interview, Vol. 1* — Alex Xu (capítulo 11: Design a News Feed System)
- 🌐 highscalability.com — arquitecturas de feed de Twitter/Instagram
- 📄 "Facebook's TAO" — el almacén del grafo social de Facebook
- 📄 Conexión con [[Desarrollo Profesional/Redis/Páginas/01 - Estructuras y Modelo|Redis: sorted sets]] y [[Desarrollo Profesional/Sistemas Distribuidos/Páginas/02 - Mensajería y Brokers|colas]]

---
`#diseño-de-sistemas` `#news-feed` `#fan-out` `#caso-estudio`
