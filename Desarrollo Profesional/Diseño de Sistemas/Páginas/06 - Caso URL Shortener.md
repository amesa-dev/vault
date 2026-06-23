# 🏛️ Caso de Estudio — Acortador de URLs

[[Desarrollo Profesional/Diseño de Sistemas/Diseño de Sistemas|⬅️ Volver a Diseño de Sistemas]] | [[Desarrollo Profesional/Diseño de Sistemas/Páginas/05 - Caso Rate Limiter|← 05]] | [[Desarrollo Profesional/Diseño de Sistemas/Páginas/07 - Caso News Feed|07 →]]

> [!abstract] Introducción
> Un acortador de URLs (tipo bit.ly) convierte una URL larga en una corta (`bit.ly/3xK9p`) que redirige a la original. Parece trivial, y por eso es un caso clásico: la dificultad no está en acortar, sino en hacerlo a escala —miles de millones de URLs, redirecciones con latencia mínima— y en la decisión central de **cómo generar el código corto**. Es un caso excelente para razonar sobre generación de IDs, ratios lectura/escritura extremas y el papel de la caché.

## ¿De qué vamos a hablar?

El diseño completo de un acortador: el cálculo de la longitud del código, las estrategias de generación de IDs, el flujo de redirección y el escalado de lecturas.

### Conceptos que vamos a cubrir
- Requisitos y estimación de escala
- ¿Cuántos caracteres necesita el código corto?
- Generación del código: hash, contador+Base62, KGS
- El flujo de redirección (301 vs 302)
- Escalado de lecturas y caché

---

## El Diseño

### 1. Requisitos y Estimación

**Funcionales**: dado una URL larga, devolver una corta; al acceder a la corta, redirigir a la larga. Opcional: URLs personalizadas, expiración, analíticas.

**No funcionales**: redirección con **latencia muy baja**, alta disponibilidad, códigos no predecibles.

**Estimación** ([[Desarrollo Profesional/Diseño de Sistemas/Páginas/01 - El Método y Estimaciones|método]]):
```
100M URLs nuevas/día → 100M / 86.400 ≈ 1.160 escrituras/seg
Ratio lectura:escritura ≈ 100:1 → ≈ 116.000 lecturas/seg
Almacenamiento: 100M/día × 365 × 10 años ≈ 365.000M URLs
× ~500 bytes/registro ≈ 180 TB
```
Conclusión inmediata: **lectura-intensivo** (caché obligatoria) y **muchísimas URLs** (la longitud del código importa).

### 2. ¿Cuántos Caracteres?

El código usa **Base62** (a-z, A-Z, 0-9 = 62 símbolos; legible y URL-safe). ¿Cuántos caracteres para cubrir las URLs previstas?

```
62^6 ≈ 56.800 millones      → insuficiente para 365.000M
62^7 ≈ 3,5 billones (10^12) → de sobra
```

Con **7 caracteres** Base62 tienes capacidad para billones de URLs. Esta es la clase de cálculo que el método pide: un número que decide directamente el diseño.

### 3. Generación del Código Corto

La decisión central, con tres enfoques:

**A) Hash de la URL.** Aplicas un hash (MD5/SHA) a la URL larga y tomas los primeros 7 caracteres en Base62.
- *Problema*: **colisiones** (dos URLs → mismo prefijo). Hay que comprobar en BD y, si colisiona, reintentar con otro trozo o un salt. La comprobación añade una lectura por escritura.
- *Ventaja*: la misma URL da el mismo código (deduplicación natural, si la quieres).

**B) Contador + Base62 (ID único auto-incremental).** Mantienes un contador global; cada URL nueva toma el siguiente entero, que conviertes a Base62.
- *Ventaja*: **sin colisiones por construcción**, corto y simple.
- *Problema*: los códigos son **secuenciales y predecibles** (1, 2, 3 → b, c, d), lo que filtra cuántas URLs hay y permite enumerarlas. Mitigación: ofuscar el ID, o usar un generador de IDs distribuido.
- *Problema de escala*: un contador único es un cuello de botella. Solución: un **generador de IDs distribuido** (tipo Snowflake, ver caso del [[Desarrollo Profesional/Diseño de Sistemas/Páginas/11 - Caso Generador de IDs|Generador de IDs]]) o repartir rangos del contador entre servidores (cada uno reserva un bloque de 1.000 IDs).

**C) Key Generation Service (KGS).** Un servicio dedicado **pregenera** claves aleatorias únicas de 7 caracteres y las guarda en una base de "claves disponibles". Al crear una URL corta, tomas una clave libre y la marcas como usada.
- *Ventaja*: sin colisiones (se garantiza al generar), sin predictibilidad (aleatorias), y la creación es rapidísima (solo coger una clave).
- *Coste*: hay que gestionar el KGS (concurrencia al repartir claves, evitar dar la misma dos veces — se reparten en bloques cacheados por cada servidor).

**Recomendación**: KGS o contador distribuido con ofuscación, según si te importa la no-predictibilidad. El hash es simple pero las colisiones lo afean.

### 4. El Flujo de Redirección

```
Crear:  POST /urls {long_url} → guarda (codigo, long_url) → devuelve "bit.ly/{codigo}"

Acceder: GET bit.ly/{codigo}
   1. ¿está {codigo} en la caché? → sí: redirige. No: ↓
   2. busca en BD: codigo → long_url; cachea con TTL.
   3. responde con redirección a long_url.
```

**¿301 o 302?** Decisión sutil con consecuencias:
- **301 Moved Permanently**: el navegador **cachea** la redirección y las siguientes veces va directo a la URL larga sin pasar por tu servidor. Reduce tu carga, pero **pierdes las analíticas** (no ves esos clics) y no puedes cambiar el destino.
- **302 Found (temporal)**: el navegador **siempre** pregunta a tu servidor. Más carga, pero **ves cada clic** (analíticas) y controlas el destino.

Si las analíticas importan (suelen importar en estos productos), **302**. Si solo quieres redirigir y descargar tu infra, 301.

### 5. Almacenamiento y Escalado de Lecturas

- **Modelo de datos** simple: `(codigo_corto PK, url_larga, creado_en, expira_en, user_id)`. Es esencialmente un **key-value** gigante → encaja con una BD clave-valor/NoSQL ([[Desarrollo Profesional/Diseño de Sistemas/Páginas/03 - Almacenamiento Replicación y Sharding|almacenamiento]]) que escala horizontalmente, o PostgreSQL sharded por `codigo_corto`.
- **Caché**: como es 100:1 lectura, una caché ([[Desarrollo Profesional/Redis/Redis|Redis]]) con los códigos más accedidos (regla 80/20: las URLs virales se piden millones de veces) absorbe la mayoría del tráfico. La redirección desde caché es casi instantánea.
- **CDN/edge**: los códigos muy populares pueden resolverse incluso en el edge.

### 6. Cuellos de Botella y Mejoras

- **Generación de IDs a escala**: el contador único es el límite; KGS o Snowflake lo resuelven.
- **Hotspots**: una URL viral concentra lecturas en un registro → la caché lo absorbe perfectamente (es una sola clave muy caliente).
- **Expiración y limpieza**: un job borra URLs caducadas; o TTL en la BD.
- **Analíticas**: cada redirección publica un evento en una **cola** ([[Desarrollo Profesional/Sistemas Distribuidos/Páginas/02 - Mensajería y Brokers|Mensajería]]) para procesar clics asíncronamente sin frenar la redirección.

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - Es **lectura-intensivo** (≈100:1) → la caché es obligatoria; y maneja **billones de URLs** → la longitud del código importa.
> - **Base62 con 7 caracteres** (62⁷ ≈ 3,5 billones) cubre la escala. Un cálculo que decide el diseño.
> - Generación del código: **hash** (simple, pero colisiones a comprobar), **contador+Base62** (sin colisiones pero predecible y con cuello de botella → distribuir/ofuscar), **KGS** (pregenera claves únicas y aleatorias, rápido, hay que gestionarlo). KGS o contador distribuido es lo recomendable.
> - **301 vs 302**: 301 cachea en el navegador (menos carga, pierdes analíticas); **302** siempre pasa por ti (analíticas y control del destino). Con analíticas → 302.
> - El modelo es un **key-value** gigante (NoSQL o SQL sharded). Caché para los códigos calientes (URLs virales) y cola para procesar analíticas sin frenar la redirección.

### Para llevar a la práctica
- [ ] Calcula cuántos caracteres Base62 necesitarías para 10× tu escala estimada.
- [ ] Implementa la conversión entero↔Base62 y la generación por contador; observa la predictibilidad.
- [ ] Razona para tu caso si usarías 301 o 302 según si necesitas analíticas.
- [ ] Diseña el flujo de caché: ¿qué TTL pondrías y cómo manejarías una URL viral (clave caliente)?

### Recursos
- 📖 *System Design Interview, Vol. 1* — Alex Xu (capítulo 8: Design a URL Shortener)
- 🌐 github.com/donnemartin/system-design-primer (ejercicio del pastebin/URL shortener)
- 📄 Conexión con [[Desarrollo Profesional/Diseño de Sistemas/Páginas/11 - Caso Generador de IDs|el caso del Generador de IDs]]

---
`#diseño-de-sistemas` `#url-shortener` `#caso-estudio` `#base62`
