# CLAUDE.md — Guía para desarrollar este vault

Este repositorio es un **vault de Obsidian**: una base de conocimiento personal en Markdown. El usuario (Andrés) va pidiendo nuevas **áreas de conocimiento** y tu trabajo es desarrollarlas como páginas de calidad **siguiendo exactamente el esquema que se describe aquí**. La consistencia con el esquema es lo más importante: una página nueva debe ser indistinguible de las existentes.

> Idioma de todo el contenido: **español de España**. Tono cercano, directo, didáctico, tuteando. Nunca traduzcas los términos técnicos consagrados (Bounded Context, Deep Work, Circuit Breaker…); explícalos en español.

---

## 1. Estructura del vault

```
Inicio.md                          ← portada/MOC raíz, enlaza las dos áreas
Progreso de Lectura.md             ← índice raíz con un checkbox por sección/página (ver §5)
Bandeja de Ideas.md                ← inbox de captura (ver §6)
Desarrollo Personal/
  Inicio Personal.md               ← MOC del área (lista todas sus secciones)
  Mentalidad/  Productividad/  ...  ← categorías con páginas-concepto sueltas
  Journaling/  Lecturas/            ← carpetas "sistema": plantillas y trackers personales
Desarrollo Profesional/
  Inicio Profesional.md            ← MOC del área (lista todas sus secciones)
  DDD/  Kubernetes/  Python/  ...   ← secciones temáticas
```

Hay **dos áreas principales**, cada una con su página `Inicio …md`. Dentro de cada área hay **secciones** (carpetas). Una sección sigue uno de estos tres patrones:

- **Patrón A — Tema técnico (el más común).** Carpeta `Tema/` con:
  - `Tema/Tema.md` → índice de la sección (MOC).
  - `Tema/Páginas/01 - Título.md`, `02 - Título.md`… → páginas de contenido numeradas.
  - Ej.: DDD, Kubernetes, GCP, Seguridad Aplicada, IA.
- **Patrón B — Curso plano.** Carpeta `Tema/` con `Tema/Tema.md` (índice) y las páginas numeradas **directamente** en la carpeta: `00 - Intro.md`, `01 - …md`, … `12 - …md`. Sin subcarpeta `Páginas`. Ej.: Python, TypeScript.
- **Patrón C — Categoría personal.** Una carpeta agrupadora (Mentalidad, Productividad, Finanzas, Salud y Bienestar, Relaciones) con **páginas-concepto sueltas**, una por idea (`Stoicismo.md`, `Deep Work.md`). No tienen índice propio: se listan en `Inicio Personal.md`.

**Regla de elección:** contenido técnico secuencial/extenso → Patrón A. Curso de un lenguaje de principio a fin → Patrón B. Concepto autocontenido de crecimiento personal → Patrón C (página suelta dentro de la categoría que corresponda).

---

## 2. Anatomía de un **índice de sección** (Patrón A y B)

```markdown
# <emoji> <Nombre de la sección>

[[Desarrollo Profesional/Inicio Profesional|⬅️ Volver a Desarrollo Profesional]]

> [!abstract] Sobre esta sección
> 2–4 frases que sitúan el tema, por qué importa y qué cubre.

---

## 📚 Páginas de esta sección

1. [[Desarrollo Profesional/<Sección>/Páginas/01 - Título|01 — Título]] — descripción breve con em-dash
2. [[Desarrollo Profesional/<Sección>/Páginas/02 - Título|02 — Título]] — descripción breve

---

<opcional: "¿Qué es X en una frase?", "Ruta recomendada", "Recursos generales">

---
`#tag` `#tag` `#indice`
```

En Patrón B la lista puede agruparse por nivel (Fundamentos / Intermedio / Avanzado) y suele incluir un bloque "🗺️ Ruta Recomendada".

---

## 3. Anatomía de una **página de contenido**

```markdown
# <emoji> <SECCIÓN NN — Título>          (en Patrón C: solo "# <emoji> Título")

[[.../Índice|⬅️ Volver a <Sección>]] | [[.../01 - Anterior|← 01]] | [[.../03 - Siguiente|03 →]]

> [!abstract] Introducción
> 2–4 frases que enganchan y explican por qué este tema importa.

## ¿De qué vamos a hablar?

Párrafo que resume el recorrido de la página.

### Conceptos que vamos a cubrir
- Concepto 1
- Concepto 2
- Concepto 3

---

## El Concepto

### <Subtema 1>
Explicación. Usa **código (Python por defecto)**, **diagramas ASCII en bloque**, **tablas** y **mermaid** cuando aporten.
Para enseñar buenas prácticas, contrasta con comentarios `# MAL` / `# BIEN`.

### <Subtema 2>
...

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - Punto 1
> - Punto 2
> - Punto 3

### Para llevar a la práctica
- [ ] Acción concreta y accionable
- [ ] Otra acción

### Recursos
- 📖 *Libro* — Autor
- 🌐 dominio.com — qué aporta

---
`#tag` `#tag` `#subtema`
```

**Navegación (breadcrumb, primera línea bajo el título):**
- Páginas numeradas: `[[índice|⬅️ Volver a X]] | [[anterior|← NN]] | [[siguiente|NN →]]`. La primera no tiene `←`, la última no tiene `→`.
- Páginas-concepto (Patrón C): solo `[[Desarrollo Personal/Inicio Personal|⬅️ Volver a Desarrollo Personal]]`.

---

## 4. Convenciones de estilo (no negociables)

- **Wikilinks siempre con ruta completa desde la raíz del vault** y alias legible: `[[Desarrollo Profesional/DDD/DDD|DDD]]`. Nunca uses rutas relativas.
- **Callouts de Obsidian** según función: `[!abstract]` (intro/sobre la sección), `[!summary]` (puntos clave), `[!tip]`, `[!info]`, `[!note]`, `[!quote]`. Respeta esa semántica.
- **Emoji** representativo y consistente para cada sección/página (en el `#` del título y al referenciarla).
- **Código: SIEMPRE en Python por defecto.** Python es el lenguaje franco del vault — **todos los ejemplos de código van en Python**, salvo que el tema sea intrínsecamente de otro lenguaje o herramienta (el curso de TypeScript en TS, React en TSX, PostgreSQL en SQL, Kubernetes/Docker en YAML/Dockerfile). Ante la duda, Python. Ejemplos cortos, idiomáticos y ejecutables mentalmente. El patrón `# MAL` / `# BIEN` es muy usado.
- **Diagramas**: cajas ASCII dentro de bloque ``` para arquitecturas/relaciones; tablas para comparativas; `mermaid` para flujos.
- **Tags** al pie, en minúscula y kebab-case, terminando con un tag de tipo: `#indice` / `#curso` / `#apuntes` para índices, y tags temáticos en las páginas.
- **Numeración**: ficheros `NN - Título.md` con dos dígitos. Mantén el orden sin huecos.
- **Profundidad**: páginas autocontenidas, con sustancia real (no esqueletos). Equilibra teoría, ejemplo de código y aplicación práctica, como en las páginas existentes.

---

## 5. Crear una nueva sección o área (flujo)

Cuando Andrés pida "hazme una sección/área de X":

1. **Decide el patrón** (§1) y el área a la que pertenece. Si dudas entre opciones que cambian el resultado, pregunta; si no, elige el patrón obvio y dilo.
2. **Crea las carpetas y ficheros**: el índice `Tema/Tema.md` y las páginas. Divide el tema en páginas coherentes (mira cuántas tienen secciones similares: 2–4 en temas medianos, más en cursos).
3. **Escribe cada página** siguiendo §2 y §3 con contenido de calidad y enlaces prev/next correctos.
4. **Registra la sección en el MOC del área**: añade su entrada en `Desarrollo Profesional/Inicio Profesional.md` o `Desarrollo Personal/Inicio Personal.md`, en el grupo temático adecuado y con el mismo formato de lista enlazada que el resto. **Una sección que no aparece en su Inicio no está terminada.**
5. **Añade la(s) entrada(s) en `Progreso de Lectura.md`**: una casilla `- [ ] [[ruta|alias]]` por sección (en Profesional) o por página-concepto (en Personal), en el grupo que corresponda. Es el segundo registro obligatorio: si no está aquí, la sección tampoco está terminada.
6. Si encaja, añade también un enlace en `Inicio.md` (portada raíz) cuando sea un área o tema de primer nivel.
7. **No toques** la carpeta `.obsidian/`.
8. **Verifica los wikilinks** antes de cerrar: usa rutas completas desde la raíz y comprueba que todos resuelven (un script rápido en Python que recorra los `.md` y valide cada `[[...]]` contra los ficheros existentes evita enlaces rotos).

---

## 6. Bandeja de Ideas (`Bandeja de Ideas.md`)

Es el **inbox de captura rápida**. Andrés apunta ahí semillas de temas para desarrollar luego. Estados: 🌱 capturada · 🔨 en progreso · ✅ desarrollada.

- Cuando diga **"coge tal idea de la bandeja"** (o lo veas pertinente): toma la entrada de *Pendientes*, conviértela en la(s) página(s) del vault siguiendo el esquema (§5), y al terminar **mueve la fila a la tabla *Desarrolladas*** con el enlace a lo creado.
- Cuando descubras o se mencione un tema nuevo que no toca desarrollar ahora, puedes **sembrarlo** como fila 🌱 en *Pendientes* (una línea, sin desarrollar).
- Respeta las tres secciones del fichero (Pendientes / Temas para compartir / Desarrolladas) y su formato de tabla.

---

## 7. Reglas de oro

- Imita el esquema y el tono de las páginas existentes antes que inventar formato nuevo.
- Toda página nueva: breadcrumb correcto + callouts + resumen + práctica + recursos + tags.
- Toda sección nueva: índice propio (A/B) **y** entrada en el Inicio del área **y** casilla en `Progreso de Lectura.md`.
- Wikilinks con ruta completa, numeración sin huecos, español de España.
- Ante una decisión que cambie el resultado (patrón, ubicación, alcance), pregunta breve; si es trivial, decide y continúa.

---

## 8. Git y mantenimiento

- Este vault se trabaja **directamente sobre `main`**: cuando Andrés pida "commit y push", commitea y empuja a `main` sin crear ramas ni PRs (es un vault personal, no un repo de equipo). Mensajes de commit en español, descriptivos.
- **Empuja siempre a los dos remotos**: el repo tiene `origin` (GitHub) y `gitlab` (GitLab). Cada vez que hagas push, hazlo a ambos (`git push origin main` **y** `git push gitlab main`) para mantenerlos sincronizados.
- **No commitees** ficheros ajenos al contenido: `.idea/` y `verify.py` están en `.gitignore`. Añade al `.gitignore` cualquier scratch o config de IDE que aparezca, en vez de commitearlo.
- Sí puedes editar el `.gitignore` cuando haga falta (deja de aplicar la regla antigua de "no lo toques").
