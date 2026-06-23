# 🏛️ Caso de Estudio — Servicio de Proximidad (Geoespacial)

[[Desarrollo Profesional/Diseño de Sistemas/Diseño de Sistemas|⬅️ Volver a Diseño de Sistemas]] | [[Desarrollo Profesional/Diseño de Sistemas/Páginas/14 - Caso Google Drive|← 14]] | [[Desarrollo Profesional/Diseño de Sistemas/Páginas/16 - Caso Sistema de Pagos|16 →]]

> [!abstract] Introducción
> "Restaurantes cerca de mí", "conductores de Uber disponibles en mi zona", "amigos cercanos": todos son el mismo problema de **búsqueda por proximidad geoespacial**. ¿Cómo encuentras eficientemente los puntos dentro de un radio, cuando tienes millones de ellos y consultas constantes? Una búsqueda ingenua (calcular la distancia a todos) no escala. La solución está en cómo **indexar el espacio** para no mirar todo el mapa. Es el caso que introduce los índices geoespaciales: geohash y quadtrees.

## ¿De qué vamos a hablar?

El diseño de un servicio de búsqueda por proximidad: por qué la indexación ingenua falla y cómo geohash y quadtrees resuelven la búsqueda espacial a escala.

### Conceptos que vamos a cubrir
- Requisitos: buscar dentro de un radio
- Por qué indexar lat/long por separado no funciona
- Geohash: convertir 2D en 1D
- Quadtree: subdividir el espacio
- Hotspots, densidad variable y objetos en movimiento

---

## El Diseño

### 1. Requisitos

**Funcionales**: dado mi ubicación (lat, long) y un radio (o "los k más cercanos"), devolver los lugares/usuarios dentro de esa zona.

**No funcionales**: **baja latencia** (es interactivo), escala (millones de puntos, muchas consultas), y a veces los puntos **se mueven** (conductores) y hay que actualizar su posición constantemente.

### 2. Por Qué la Indexación Ingenua Falla

**Opción ingenua A — calcular la distancia a todos**: para cada consulta, calcula la distancia de mi ubicación a cada uno de los millones de puntos y filtra. Es O(n) por consulta → inviable a escala.

**Opción ingenua B — índice en lat y long por separado**: `WHERE lat BETWEEN ... AND long BETWEEN ...`. El problema: una BD puede usar **un índice eficientemente, no dos rangos a la vez** de forma óptima. Devuelve un "rectángulo" enorme de candidatos (toda la franja de latitud + toda la franja de longitud) que luego hay que filtrar. Ineficiente.

El problema de fondo: el espacio es **2D** y los índices clásicos (B-Tree) son **1D**. Necesitamos una forma de mapear la cercanía 2D a algo 1D indexable, o de subdividir el espacio. Dos enfoques dominantes:

### 3. Geohash — De 2D a 1D

El **geohash** convierte una coordenada (lat, long) en una **cadena corta** (p. ej. `ezs42`) subdividiendo el mundo recursivamente en una cuadrícula y entrelazando los bits de lat y long. Su propiedad mágica:

> **Puntos cercanos en el mapa comparten prefijos de geohash.** Cuanto más largo el prefijo común, más cerca están.

```
geohash de 6 chars ≈ celda de ~1,2 km
"ezs42" y "ezs43" → vecinos (comparten prefijo "ezs4")
"ezs42" y "u4pruy" → lejanos (no comparten prefijo)
```

Esto convierte la búsqueda por proximidad en una **búsqueda por prefijo de string**, que un índice B-Tree (o Redis con `GEOADD`/`GEOSEARCH`, que usa geohash por debajo) resuelve eficientemente: "dame los puntos cuyo geohash empieza por `ezs4`" = los de esa celda. Para cubrir un radio que cae entre celdas, consultas la celda del usuario **y sus 8 vecinas** (un punto cerca del borde de una celda tiene vecinos en la de al lado).

- **Precisión ajustable**: eliges la longitud del geohash según el radio buscado (prefijo más corto = celda más grande).
- Es la opción más usada por su simplicidad y porque encaja en cualquier BD con índice de strings y en Redis nativamente.

### 4. Quadtree — Subdividir el Espacio

Un **quadtree** es un árbol donde cada nodo representa una región cuadrada que se **subdivide en 4 cuadrantes** cuando contiene demasiados puntos. Las zonas densas (centro de una ciudad) se subdividen mucho (celdas pequeñas); las vacías (océano) quedan como una celda grande.

```
        [Mundo]
       / | | \
   [NO][NE][SO][SE]   ← cada cuadrante se subdivide solo si tiene muchos puntos
            / | | \
         (zona densa subdividida en más detalle)
```

- **Ventaja sobre geohash**: se adapta a la **densidad** (geohash usa celdas de tamaño fijo, lo que crea celdas vacías y celdas saturadas). El quadtree mantiene un número equilibrado de puntos por celda.
- **Coste**: es una estructura en memoria que hay que construir y mantener; rebalancear cuando los puntos se mueven es más complejo que con geohash. Lo usan sistemas que necesitan adaptarse a densidad muy variable.

**Geohash vs quadtree**: geohash es más simple y encaja en BDs/Redis estándar (empieza por aquí); quadtree maneja mejor la densidad desigual pero es más complejo de operar. Otros: índices R-tree (PostGIS de [[Desarrollo Profesional/PostgreSQL/Páginas/04 - Funciones Avanzadas|PostgreSQL]]), curvas de Hilbert, S2 de Google.

### 5. Objetos en Movimiento y Hotspots

- **Puntos que se mueven** (conductores de Uber): su ubicación cambia cada pocos segundos → muchísimas actualizaciones de escritura. Se guardan en un almacén rápido en memoria (Redis con geo) y se actualizan con cada *ping* de ubicación. No necesitas durabilidad fuerte de cada posición intermedia (consistencia eventual; si pierdes un ping, el siguiente lo corrige).
- **Hotspots de consulta**: el centro de una ciudad concentra consultas y puntos → caché de las celdas calientes, y el quadtree ayuda a no devolver demasiados candidatos de una celda densa.
- **Filtrado posterior**: tras obtener los candidatos de la celda + vecinas, se calcula la distancia real (Haversine) solo sobre ese conjunto pequeño y se ordenan — barato porque ya son pocos.

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - Buscar puntos dentro de un radio a escala: la indexación ingenua falla porque calcular la distancia a todos es O(n), e indexar lat/long por separado no aprovecha bien un índice (devuelve demasiados candidatos). El espacio es 2D y los índices clásicos 1D.
> - **Geohash**: codifica (lat,long) en una cadena donde **puntos cercanos comparten prefijo**. Convierte la proximidad en búsqueda por prefijo (B-Tree o Redis `GEOSEARCH` nativo). Consulta la celda del usuario **y sus 8 vecinas**. Simple y encaja en cualquier BD; el más usado.
> - **Quadtree**: subdivide el espacio en 4 recursivamente, adaptándose a la **densidad** (celdas pequeñas donde hay muchos puntos). Mejor para densidad desigual, pero más complejo de mantener.
> - Empieza con **geohash** (o PostGIS); pasa a quadtree/estructuras avanzadas si la densidad muy variable lo exige.
> - **Objetos en movimiento** (conductores): ubicación en memoria (Redis geo), actualizada con cada ping, consistencia eventual. **Filtra la distancia real (Haversine)** solo sobre el pequeño conjunto de candidatos de la celda + vecinas.

### Para llevar a la práctica
- [ ] Explica por qué dos puntos cercanos comparten prefijo de geohash y cómo lo aprovechas para buscar.
- [ ] Razona por qué necesitas consultar las 8 celdas vecinas además de la del usuario.
- [ ] Compara geohash vs quadtree para una ciudad con densidad muy desigual.
- [ ] Diseña cómo actualizarías la posición de millones de conductores en movimiento sin saturar la BD.

### Recursos
- 📖 *System Design Interview, Vol. 2* — Alex Xu (Proximity Service / Nearby Friends)
- 🌐 redis.io/docs/latest/develop/data-types/geospatial — geoespacial en Redis
- 🌐 postgis.net — extensión geoespacial de PostgreSQL (R-tree/GiST)
- 📄 s2geometry.io — la librería S2 de Google

---
`#diseño-de-sistemas` `#geoespacial` `#geohash` `#quadtree` `#caso-estudio`
