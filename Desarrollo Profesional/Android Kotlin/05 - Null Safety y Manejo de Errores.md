# 📱 05 — Null Safety y Manejo de Errores

[[Desarrollo Profesional/Android Kotlin/Android Kotlin|⬅️ Volver al índice]] | [[Desarrollo Profesional/Android Kotlin/04 - Colecciones y Programación Funcional|← 04]] | [[Desarrollo Profesional/Android Kotlin/06 - Coroutines y Flow|06 →]]

> [!abstract] Introducción
> El "error del billón de dólares" —la `NullPointerException`— es, en Kotlin, en gran parte cosa del pasado. El sistema de tipos distingue `String` (nunca nulo) de `String?` (puede ser nulo) y te obliga a manejar el caso nulo en compilación. Esta página te da las herramientas para tratar la nulabilidad con elegancia y para modelar errores de forma explícita.

## ¿De qué vamos a hablar?

Tipos anulables, los operadores `?.`, `?:` y `!!`, los *smart casts*, y dos formas de manejar fallos: excepciones y el patrón `Result` con `sealed class`.

### Conceptos que vamos a cubrir
- Tipos anulables (`T?`) vs no anulables (`T`)
- Safe call `?.`, Elvis `?:` y el peligroso `!!`
- Smart casts y `let` para no-nulos
- Excepciones en Kotlin (`try`/`catch` como expresión)
- Modelar fallos con un `Result` sellado

---

## El Concepto

### Anulable o no anulable: lo decide el tipo

Por defecto, un tipo **no admite** `null`. Para permitirlo, añade `?`:

```kotlin
val nombre: String = "Ana"
// nombre = null            // ❌ error de compilación

val apodo: String? = null   // ✅ explícitamente anulable
// println(apodo.length)    // ❌ error: apodo podría ser null
```

El compilador no te deja desreferenciar un anulable sin tratarlo antes. Esto traslada el fallo de *runtime* a *compilación*.

### Safe call `?.` y Elvis `?:`

```kotlin
val longitud: Int? = apodo?.length       // null si apodo es null; si no, su length

// Elvis: valor por defecto cuando algo es null
val n: Int = apodo?.length ?: 0          // 0 si apodo es null

// Encadenable
val ciudad = usuario?.direccion?.ciudad ?: "desconocida"

// Elvis con return/throw para "early exit"
fun procesar(u: Usuario?) {
    val usuario = u ?: return            // sale si es null
    // a partir de aquí 'usuario' es no-nulo
}
```

### El operador `!!`: evítalo

`!!` afirma "esto no es null" y lanza `NullPointerException` si lo es. Es renunciar a la red de seguridad:

```kotlin
val longitud = apodo!!.length   // 💥 NPE si apodo es null
```

> [!warning] `!!` es un *code smell*
> Casi siempre hay una alternativa mejor: `?.`, `?:`, `requireNotNull(x)` (que da un mensaje claro) o reestructurar para que el tipo no sea anulable. Reserva `!!` para casos donde es realmente imposible que sea nulo.

### Smart casts y `let`

Tras comprobar que algo no es null (o tras un `is`), el compilador lo trata como no-nulo automáticamente —el **smart cast**:

```kotlin
fun saludar(nombre: String?) {
    if (nombre != null) {
        println(nombre.uppercase())   // smart cast: aquí es String, no String?
    }
}
```

`let` ejecuta un bloque solo si el receptor no es null —muy idiomático:

```kotlin
apodo?.let {
    println("El apodo es $it")   // se ejecuta solo si apodo != null
}
```

### Excepciones

Se lanzan y capturan como en Java, pero en Kotlin **no hay excepciones comprobadas** (no se declaran en la firma) y `try` es una expresión:

```kotlin
fun parsear(texto: String): Int? = try {
    texto.toInt()
} catch (e: NumberFormatException) {
    null
}

// Funciones de la stdlib que ya devuelven null en vez de lanzar
val n: Int? = "abc".toIntOrNull()      // null, sin try/catch

// Precondiciones: lanzan con mensajes claros
fun retirar(cantidad: Int) {
    require(cantidad > 0) { "La cantidad debe ser positiva" }   // IllegalArgumentException
    check(saldo >= cantidad) { "Saldo insuficiente" }           // IllegalStateException
}
```

### Modelar fallos con un `Result` sellado

En arquitecturas limpias (lo veremos en MVVM) suele preferirse **devolver** el error en vez de lanzarlo, para forzar a la capa superior a tratarlo. Una `sealed interface` lo expresa muy bien:

```kotlin
sealed interface Resultado<out T> {
    data class Exito<T>(val datos: T) : Resultado<T>
    data class Error(val mensaje: String) : Resultado<Nothing>
}

fun cargarUsuario(id: Int): Resultado<Usuario> =
    try {
        Resultado.Exito(api.obtener(id))
    } catch (e: Exception) {
        Resultado.Error(e.message ?: "Error desconocido")
    }

// El llamador está obligado a tratar ambos casos (when exhaustivo)
when (val r = cargarUsuario(42)) {
    is Resultado.Exito -> mostrar(r.datos)
    is Resultado.Error -> mostrarError(r.mensaje)
}
```

> [!info] `kotlin.Result` ya existe
> La stdlib trae un `Result<T>` con `runCatching { ... }`. Para estados de UI suele preferirse un sellado propio porque permite añadir un estado `Cargando` y casa con el `when` exhaustivo.

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - El tipo decide la nulabilidad: `T` nunca es null, `T?` puede serlo. El compilador te obliga a tratar el caso nulo.
> - `?.` (safe call) y `?:` (Elvis, con valor por defecto o `return`/`throw`) son tu día a día. Evita `!!`.
> - El smart cast trata como no-nulo tras comprobarlo; `?.let { }` ejecuta solo si no es null.
> - Kotlin no tiene excepciones comprobadas; `try` es expresión. Usa `toIntOrNull()` y `require`/`check` para errores claros.
> - Para arquitectura limpia, modela el fallo con un `sealed` `Result` y trátalo con `when` exhaustivo en lugar de propagar excepciones.

### Para llevar a la práctica
- [ ] Sustituye un `!!` de tu código por `?:` con un valor por defecto sensato
- [ ] Convierte una cadena anulable con `?.let { }` en lugar de un `if (x != null)`
- [ ] Reemplaza un `try/catch` que parsea por `toIntOrNull()`
- [ ] Modela el resultado de una operación con una `sealed interface` `Exito`/`Error`

### Recursos
- 🌐 kotlinlang.org/docs/null-safety.html — null safety oficial
- 🌐 kotlinlang.org/docs/exceptions.html — excepciones en Kotlin
- 📖 *Kotlin in Action* — capítulo 6 (sistema de tipos y nulabilidad)

---
`#kotlin` `#null-safety` `#errores` `#sealed-class`
