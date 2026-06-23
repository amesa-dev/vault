# 🏛️ Caso de Estudio — Autocompletado / Typeahead

[[Desarrollo Profesional/Diseño de Sistemas/Diseño de Sistemas|⬅️ Volver a Diseño de Sistemas]] | [[Desarrollo Profesional/Diseño de Sistemas/Páginas/09 - Caso Sistema de Notificaciones|← 09]] | [[Desarrollo Profesional/Diseño de Sistemas/Páginas/11 - Caso Generador de IDs|11 →]]

> [!abstract] Introducción
> El autocompletado de búsqueda (las sugerencias que aparecen mientras escribes en Google) parece un detalle de UI, pero a escala es un problema de diseño exigente: las sugerencias deben aparecer en **decenas de milisegundos** mientras el usuario teclea, con un volumen de peticiones brutal (una por cada tecla pulsada por millones de usuarios). Es el caso perfecto para hablar de la estructura de datos **trie**, de precálculo y de cómo separar el camino de lectura (rapidísimo) del de actualización (lento, offline).

## ¿De qué vamos a hablar?

El diseño de un sistema de sugerencias en tiempo real: la estructura trie, el precálculo de las top-k sugerencias y la separación lectura/escritura.

### Conceptos que vamos a cubrir
- Requisitos: latencia y volumen
- El trie como estructura base
- Precalcular las top-k por nodo
- Separar recopilación de datos (offline) de servicio (online)
- Sharding y caché del trie

---

## El Diseño

### 1. Requisitos y Escala

**Funcionales**: dado un prefijo que el usuario teclea, devolver las **k sugerencias más populares** que empiezan por él (típicamente top-5 o top-10), ordenadas por frecuencia.

**No funcionales**: **latencia ínfima** (<100ms, idealmente <50ms — debe sentirse instantáneo mientras tecleas), volumen enorme (cada pulsación es una petición → si hay 10M búsquedas/día y cada una son ~20 pulsaciones, son ~200M peticiones de sugerencia/día). **Lectura-intensivo extremo.**

### 2. El Trie

La estructura de datos natural para sugerencias por prefijo es el **trie** (árbol de prefijos): cada nodo es un carácter, y un camino de la raíz a un nodo deletrea un prefijo. Todas las palabras que empiezan por un prefijo cuelgan del nodo de ese prefijo.

```
            (raíz)
           /   |   \
          c    t    s
          |    |    |
          a    e    o
         /|    |    |
        t s    a    l
   "cat"  "cas..." "tea" "sol..."
```

Buscar sugerencias para "ca" = ir al nodo "ca" y recoger las palabras de su subárbol. El problema: **recorrer el subárbol en cada pulsación es demasiado lento** si el subárbol es grande (un prefijo corto como "a" tiene millones de descendientes).

### 3. Precalcular las Top-k por Nodo

La optimización clave: **en cada nodo del trie, guarda precalculadas las k sugerencias más frecuentes de su subárbol.** Así, responder a un prefijo es: ir al nodo (O(longitud del prefijo)) y devolver su lista top-k ya lista (O(1)). Sin recorrer nada en tiempo de consulta.

```
Nodo "ca" → top-5 cacheada: ["casa", "cama", "calle", "cara", "café"]
Consulta "ca" → navega al nodo → devuelve esa lista directamente. Instantáneo.
```

El coste se traslada a la **construcción/actualización** del trie (calcular las top-k de cada nodo), que es cara — pero eso se hace **offline**, no en el camino crítico.

### 4. Separar Recopilación (Offline) de Servicio (Online)

La arquitectura se parte en dos sistemas con ritmos muy distintos:

```
OFFLINE (lento, periódico):
  [Logs de búsquedas] → [Agregación: contar frecuencias] →
  [Construir/actualizar el trie con top-k por nodo] → [Publicar el trie]

ONLINE (rapidísimo, por cada pulsación):
  [Usuario teclea prefijo] → [Servidor de autocompletado: trie en memoria] →
  devuelve top-k del nodo → (con caché delante)
```

- El **camino de actualización** (recopilar qué busca la gente, contar frecuencias, reconstruir el trie) corre en **batch**, cada hora o cada día. Las sugerencias no necesitan ser de hace un segundo; que reflejen las tendencias de ayer está bien (consistencia eventual perfectamente aceptable).
- El **camino de servicio** solo lee un trie ya construido y cargado en memoria. Por eso es tan rápido.

Esta separación lectura-pesada/escritura-pesada con ritmos distintos es la lección transferible: **cuando la lectura debe ser ultrarrápida y la frescura no es crítica, precalcula offline y sirve online.**

### 5. Almacenamiento, Sharding y Caché

- **El trie en memoria**: para máxima velocidad, el trie de servicio vive en RAM. Se persiste (serializado) para reconstruir al arrancar.
- **Sharding del trie**: a escala global el trie es enorme. Se **shardea por prefijo** (las palabras que empiezan por 'a'-'m' en un grupo de servidores, 'n'-'z' en otro). Cuidado con el balanceo: no todos los prefijos son igual de populares (hotspots), así que el reparto se hace por carga, no alfabéticamente plano.
- **Caché**: una caché ([[Desarrollo Profesional/Redis/Redis|Redis]]/CDN) delante de los prefijos más populares absorbe la mayoría (poca gente teclea prefijos raros; "fa", "yo", "co" se piden constantemente — regla 80/20).
- **Optimización de cliente**: el navegador puede *debounce* (esperar a que dejes de teclear unos ms antes de pedir) y cachear prefijos ya consultados, reduciendo peticiones.

### 6. Cuellos de Botella y Mejoras

- **Coste de reconstruir el trie**: actualizaciones incrementales en vez de reconstruir entero; o reconstruir en background y cambiar atómicamente.
- **Prefijos cortos muy demandados**: caché agresiva (son pocos y muy pedidos).
- **Personalización y contexto** (sugerencias según ubicación, historial): complica el precálculo (ya no hay un top-k global único); se combina trie global con ajustes por usuario.
- **Filtrado** (contenido inapropiado, sugerencias eliminadas): lista de bloqueo aplicada al servir.

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - Sugerencias por prefijo con **latencia ínfima** (<50-100ms) y volumen brutal (una petición por pulsación). Lectura-intensivo extremo.
> - La estructura base es el **trie** (árbol de prefijos), pero recorrer el subárbol en cada consulta es demasiado lento.
> - Optimización clave: **precalcular las top-k sugerencias en cada nodo**, de modo que responder es navegar al nodo (O(prefijo)) y devolver su lista ya lista (O(1)).
> - **Separa offline/online**: la construcción del trie (agregando logs de búsqueda, contando frecuencias) corre en **batch** (frescura no crítica); el servicio solo lee el trie en memoria. Precalcula offline, sirve online.
> - El trie vive en **RAM**, se **shardea por prefijo** (balanceando por carga, no alfabéticamente), y una **caché** delante absorbe los prefijos populares (80/20). El cliente hace *debounce*.

### Para llevar a la práctica
- [ ] Implementa un trie con top-k por nodo y mide la diferencia de latencia frente a recorrer el subárbol.
- [ ] Diseña el pipeline offline: de logs de búsqueda a un trie con frecuencias actualizadas.
- [ ] Razona cómo shardearías el trie evitando que el prefijo 'a' (enorme) desequilibre.
- [ ] Añade *debounce* en el cliente y estima cuántas peticiones ahorra.

### Recursos
- 📖 *System Design Interview, Vol. 1* — Alex Xu (capítulo 13: Design a Search Autocomplete System)
- 🌐 es.wikipedia.org/wiki/Trie — la estructura trie
- 📄 Conexión con [[Desarrollo Profesional/Diseño de Sistemas/Páginas/04 - Caché CDN Balanceo y Colas|caché]] y precálculo

---
`#diseño-de-sistemas` `#autocompletado` `#trie` `#caso-estudio`
