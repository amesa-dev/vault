# 🏛️ Caso de Estudio — Google Drive / Almacenamiento de Ficheros

[[Desarrollo Profesional/Diseño de Sistemas/Diseño de Sistemas|⬅️ Volver a Diseño de Sistemas]] | [[Desarrollo Profesional/Diseño de Sistemas/Páginas/13 - Caso YouTube y Streaming|← 13]] | [[Desarrollo Profesional/Diseño de Sistemas/Páginas/15 - Caso Proximidad Geoespacial|15 →]]

> [!abstract] Introducción
> Un servicio de almacenamiento y sincronización de ficheros (Google Drive, Dropbox) plantea un problema que parece simple —guardar archivos— pero esconde lo difícil: **sincronizar** el mismo fichero entre varios dispositivos de un usuario de forma eficiente y consistente, detectando cambios, resolviendo conflictos y sin retransmitir gigabytes cada vez que cambias una coma. La clave es el **bloque** como unidad y el versionado.

## ¿De qué vamos a hablar?

El diseño de un almacenamiento de ficheros con sincronización: el modelo de bloques, los metadatos, la sincronización entre dispositivos y los conflictos.

### Conceptos que vamos a cubrir
- Requisitos: subir, descargar, sincronizar, compartir
- El almacenamiento por bloques y la deduplicación
- Metadatos vs contenido: dos sistemas separados
- Sincronización: detección de cambios y delta sync
- Conflictos, versionado y consistencia

---

## El Diseño

### 1. Requisitos

**Funcionales**: subir/descargar ficheros, **sincronizarlos** entre los dispositivos del usuario, compartirlos, ver el historial de versiones.

**No funcionales**: **fiabilidad** (no perder datos jamás — la durabilidad es sagrada), eficiencia de ancho de banda (no resubir todo ante un cambio pequeño), escala (miles de millones de ficheros), y resolución razonable de conflictos.

### 2. Almacenamiento por Bloques

La decisión fundacional: **un fichero no se guarda como un blob único, sino partido en bloques** (chunks de tamaño fijo, p. ej. 4 MB). Cada bloque se identifica por el **hash de su contenido**. Esto habilita tres cosas potentes:

- **Delta sync (sincronización incremental)**: si cambias una parte de un fichero grande, solo cambian los bloques afectados. Sincronizas **solo esos bloques**, no el fichero entero. Cambiar una línea de un PDF de 1 GB transfiere unos KB, no 1 GB.
- **Deduplicación**: si dos usuarios (o dos ficheros) tienen el mismo bloque (mismo hash), se almacena **una sola vez**. Ahorra enormemente (mucha gente sube los mismos PDFs, imágenes…).
- **Subida/descarga paralela y reanudable**: los bloques se transfieren en paralelo y, si se corta, se reanuda por bloque.

```
fichero.pdf → [bloque1: hash a3f...] [bloque2: hash b7c...] [bloque3: hash 9d2...]
Editas el final → solo cambia bloque3 → solo se sincroniza bloque3.
```

### 3. Metadatos vs Contenido: Dos Sistemas

Se separan claramente dos planos:
- **El contenido** (los bloques): va a **almacenamiento de objetos** (S3/GCS), barato, durable, replicado. Los bloques son inmutables (identificados por hash).
- **Los metadatos** (estructura de carpetas, nombres, qué bloques componen cada fichero y en qué orden, versiones, permisos de compartición): va a una **base de datos** (relacional o NoSQL), porque tiene estructura, relaciones y se consulta mucho.

```
[Metadata DB]                          [Block storage (objetos)]
fichero.pdf v3 = [a3f, b7c, 9d2]  →    bloque a3f: <bytes>
carpeta /trabajo contiene...           bloque b7c: <bytes>
permisos, versiones...                 bloque 9d2: <bytes>
```

Esta separación es la columna vertebral: el contenido pesado en almacenamiento barato e inmutable; la estructura ligera y mutable en una BD.

### 4. Sincronización entre Dispositivos

El corazón del producto. Cada dispositivo tiene un cliente que mantiene su copia local sincronizada:

```
1. El cliente detecta un cambio local (un fichero editado).
2. Calcula qué bloques cambiaron (hashes) → sube solo esos al block storage.
3. Actualiza los metadatos (nueva versión del fichero) en el servidor.
4. El servidor notifica a los OTROS dispositivos del usuario que hay un cambio.
5. Los otros dispositivos descargan solo los bloques nuevos y actualizan su copia.
```

La **notificación de cambios** a los otros dispositivos se hace con una conexión persistente (long polling o WebSocket, como en el [[Desarrollo Profesional/Diseño de Sistemas/Páginas/08 - Caso Chat en Tiempo Real|chat]]): el servidor empuja "hay novedades" y el cliente sincroniza. Un **servicio de notificación de cambios** desacopla esto, a menudo vía cola.

### 5. Conflictos, Versionado y Consistencia

- **Conflictos**: dos dispositivos editan el mismo fichero offline y luego sincronizan. ¿Qué versión gana? Estrategia común: **el primero que sincroniza gana**, y al segundo se le crea una **copia de conflicto** ("documento (conflicto de Ana)") en vez de perder datos silenciosamente. Nunca pierdas datos del usuario; ante la duda, conserva ambas versiones.
- **Versionado**: como los bloques son inmutables y los metadatos guardan qué bloques forman cada versión, mantener el **historial** es casi gratis (una versión nueva referencia los bloques que cambiaron + los que no). Permite "restaurar versión anterior".
- **Consistencia**: la sincronización es **eventual** (los dispositivos convergen tras unos segundos). Aceptable y esperado en este dominio.

### 6. Cuellos de Botella y Mejoras

- **Durabilidad**: replicación del block storage en varias zonas; checksums para detectar corrupción. Perder un fichero es inaceptable.
- **Compartición**: un nivel de permisos sobre los metadatos (quién puede ver/editar) — su propia complejidad de autorización ([[Desarrollo Profesional/Seguridad Aplicada/Páginas/02 - Autenticación y Autorización|RBAC/ABAC]]).
- **Ficheros muy grandes**: el chunking lo maneja; subida resumible.
- **Coste de almacenamiento**: deduplicación + compresión + tiers fríos para lo viejo.

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - La durabilidad es sagrada (no perder datos) y el reto real es la **sincronización eficiente** entre dispositivos sin retransmitir ficheros enteros.
> - Decisión fundacional: **partir los ficheros en bloques** identificados por hash de contenido. Habilita **delta sync** (solo sincronizas los bloques que cambian), **deduplicación** (bloques iguales se guardan una vez) y transferencia paralela/reanudable.
> - **Separa contenido y metadatos**: los bloques (inmutables, pesados) en almacenamiento de objetos barato y replicado; la estructura (carpetas, versiones, qué bloques forman cada fichero, permisos) en una **BD**.
> - **Sincronización**: el cliente sube solo los bloques cambiados y actualiza metadatos; el servidor **notifica** a los demás dispositivos (conexión persistente/cola) que descargan solo lo nuevo.
> - **Conflictos**: nunca pierdas datos — el primero gana y al otro se le crea una **copia de conflicto**. El **versionado** es casi gratis gracias a los bloques inmutables. Consistencia **eventual**.

### Para llevar a la práctica
- [ ] Explica por qué el chunking por hash habilita delta sync y deduplicación a la vez.
- [ ] Diseña el modelo de metadatos: ¿cómo representas que la versión 3 de un fichero son los bloques [a3f, b7c, 9d2]?
- [ ] Razona el flujo de sincronización completo cuando editas un fichero en el móvil y lo tienes abierto en el portátil.
- [ ] Decide tu estrategia de conflictos y por qué nunca debe perder datos del usuario.

### Recursos
- 📖 *System Design Interview, Vol. 2* — Alex Xu (Design Google Drive)
- 📄 "Dropbox's storage architecture" — blogs de ingeniería de Dropbox (Magic Pocket)
- 📄 Conexión con [[Desarrollo Profesional/Diseño de Sistemas/Páginas/08 - Caso Chat en Tiempo Real|notificación push]] y almacenamiento de objetos

---
`#diseño-de-sistemas` `#almacenamiento` `#sincronizacion` `#caso-estudio`
