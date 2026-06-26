# 📱 04 — Colecciones y Programación Funcional

[[Desarrollo Profesional/Android Kotlin/Android Kotlin|⬅️ Volver al índice]] | [[Desarrollo Profesional/Android Kotlin/03 - Clases y POO|← 03]] | [[Desarrollo Profesional/Android Kotlin/05 - Null Safety y Manejo de Errores|05 →]]

> [!abstract] Introducción
> La librería de colecciones de Kotlin es una de las mejores que existen: declarativa, encadenable y con una clara separación entre colecciones mutables e inmutables. Aprender `map`, `filter`, `groupBy` y compañía te ahorra bucles enteros y hace tu código mucho más legible. Esta página es pura productividad diaria.

## ¿De qué vamos a hablar?

Los tipos de colección, la distinción mutable/inmutable, las operaciones funcionales que sustituyen a los bucles, y cuándo usar `Sequence` para no crear listas intermedias.

### Conceptos que vamos a cubrir
- `List`, `Set`, `Map` y sus variantes mutables
- Transformaciones: `map`, `filter`, `flatMap`
- Agregación: `fold`, `reduce`, `sumOf`, `count`
- Agrupación y búsqueda: `groupBy`, `associateBy`, `find`, `any`/`all`
- `Sequence`: evaluación perezosa

---

## El Concepto

### Colecciones inmutables por defecto

Kotlin distingue entre interfaces de **solo lectura** (`List`, `Set`, `Map`) y **mutables** (`MutableList`…). Por defecto, crea inmutables:

```kotlin
val nums = listOf(1, 2, 3)              // List (solo lectura)
val mut = mutableListOf(1, 2, 3)        // MutableList
mut.add(4)

val mapa = mapOf("a" to 1, "b" to 2)    // Map
val conjunto = setOf(1, 2, 2, 3)        // Set → {1, 2, 3}

println(mapa["a"])                       // 1
for ((clave, valor) in mapa) { /* ... */ }
```

> [!tip] Inmutable por defecto, igual que con `val`
> Expón `List` en tus APIs y usa `MutableList` solo internamente. Reduce errores y encaja con el estado inmutable de Compose.

### Transformaciones

Estas operaciones devuelven una **nueva** colección sin tocar la original:

```kotlin
val nums = listOf(1, 2, 3, 4, 5)

nums.map { it * 2 }              // [2, 4, 6, 8, 10]
nums.filter { it % 2 == 0 }     // [2, 4]
nums.map { it * 2 }.filter { it > 4 }   // encadenado → [6, 8, 10]

// flatMap: aplana una colección de colecciones
val listas = listOf(listOf(1, 2), listOf(3, 4))
listas.flatMap { it }           // [1, 2, 3, 4]

val nombres = listOf("ana", "leo")
nombres.mapIndexed { i, n -> "$i: $n" }  // ["0: ana", "1: leo"]
```

### Agregación

```kotlin
val nums = listOf(1, 2, 3, 4)

nums.sum()                          // 10
nums.sumOf { it * it }              // 30  (suma de cuadrados)
nums.count { it > 2 }              // 2
nums.maxOrNull()                    // 4
nums.average()                      // 2.5

// fold: acumula con un valor inicial
nums.fold(0) { acc, n -> acc + n }  // 10
// reduce: como fold pero usa el primer elemento como inicial
nums.reduce { acc, n -> acc * n }   // 24
```

### Agrupación, transformación a mapas y búsqueda

```kotlin
data class Persona(val nombre: String, val ciudad: String)
val personas = listOf(
    Persona("Ana", "Madrid"),
    Persona("Leo", "Madrid"),
    Persona("Eva", "Sevilla"),
)

// groupBy: Map<Ciudad, List<Persona>>
personas.groupBy { it.ciudad }
// {Madrid=[Ana, Leo], Sevilla=[Eva]}

// associateBy: Map<clave, elemento> (índice por clave)
personas.associateBy { it.nombre }

// búsqueda y predicados
personas.find { it.ciudad == "Sevilla" }    // Persona(Eva, Sevilla) o null
personas.any { it.ciudad == "Madrid" }      // true
personas.all { it.nombre.length == 3 }      // true
personas.sortedBy { it.nombre }
```

Compara el estilo declarativo con un bucle imperativo:

```kotlin
// MAL: imperativo, mutable, ruidoso
val resultado = mutableListOf<String>()
for (p in personas) {
    if (p.ciudad == "Madrid") resultado.add(p.nombre.uppercase())
}

// BIEN: declarativo, una línea, sin mutación
val resultado = personas
    .filter { it.ciudad == "Madrid" }
    .map { it.nombre.uppercase() }
```

### `Sequence`: evaluación perezosa

Cada operación de colección crea una lista intermedia. Para cadenas largas o colecciones grandes, `asSequence()` evalúa **perezosamente**, elemento a elemento, sin listas intermedias:

```kotlin
val resultado = (1..1_000_000).asSequence()
    .map { it * 2 }          // no se ejecuta aún
    .filter { it % 3 == 0 }  // no se ejecuta aún
    .take(5)                 // operación terminal: tira del flujo
    .toList()                // [6, 12, 18, 24, 30]
```

Regla práctica: listas pequeñas → operaciones normales (más simples). Colecciones grandes o muchas etapas encadenadas → `Sequence`.

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - Las colecciones son de solo lectura por defecto (`listOf`, `mapOf`); usa las `mutable*` solo cuando haga falta.
> - `map`/`filter`/`flatMap` transforman devolviendo colecciones nuevas; encadénalas en vez de escribir bucles.
> - `fold`/`reduce`/`sumOf`/`count` agregan; `groupBy`/`associateBy` reestructuran; `find`/`any`/`all` buscan.
> - El estilo declarativo sustituye bucles imperativos con estado mutable: más corto y más seguro.
> - `asSequence()` evalúa de forma perezosa, sin colecciones intermedias: úsalo en datos grandes o cadenas largas.

### Para llevar a la práctica
- [ ] Reescribe un bucle `for` con acumulador como cadena `filter`/`map`
- [ ] Agrupa una lista de objetos por una propiedad con `groupBy`
- [ ] Usa `associateBy` para crear un índice (`Map`) por id
- [ ] Convierte una cadena larga de operaciones a `asSequence()` y observa la diferencia

### Recursos
- 🌐 kotlinlang.org/docs/collections-overview.html — colecciones oficiales
- 🌐 kotlinlang.org/docs/sequences.html — sequences y evaluación perezosa
- 📖 *Kotlin in Action* — capítulos 5 y 6

---
`#kotlin` `#colecciones` `#programacion-funcional` `#sequences`
