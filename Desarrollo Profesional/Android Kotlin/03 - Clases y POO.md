# 📱 03 — Clases y POO

[[Desarrollo Profesional/Android Kotlin/Android Kotlin|⬅️ Volver al índice]] | [[Desarrollo Profesional/Android Kotlin/02 - Funciones y Lambdas|← 02]] | [[Desarrollo Profesional/Android Kotlin/04 - Colecciones y Programación Funcional|04 →]]

> [!abstract] Introducción
> Kotlin reduce drásticamente el boilerplate de la orientación a objetos de Java. Una `data class` te da `equals`, `hashCode`, `toString` y `copy` gratis. Las `sealed class` modelan jerarquías cerradas perfectas para representar estados de UI. Esta página recorre las herramientas de modelado que usarás a diario en Android.

## ¿De qué vamos a hablar?

Cómo declarar clases con constructor primario, las `data class` para datos, `enum` y `sealed class` para jerarquías cerradas, `object` para singletons, y la herencia (que en Kotlin es cerrada por defecto).

### Conceptos que vamos a cubrir
- Clases y constructor primario
- `data class` y desestructuración
- Clases y métodos `open` (herencia cerrada por defecto)
- Interfaces y delegación con `by`
- `enum class` y `sealed class`
- `object` y `companion object`

---

## El Concepto

### Clases y constructor primario

El constructor primario se declara en la cabecera de la clase. Las propiedades se definen ahí mismo con `val`/`var`:

```kotlin
class Usuario(val nombre: String, var edad: Int) {
    // bloque de inicialización
    init {
        require(edad >= 0) { "La edad no puede ser negativa" }
    }

    fun cumpleAnios() { edad++ }
}

val u = Usuario("Andrés", 31)   // sin 'new' en Kotlin
println(u.nombre)               // acceso directo a la propiedad
```

### `data class`: el caballo de batalla

Para clases que solo transportan datos. El compilador genera `equals`, `hashCode`, `toString`, `copy` y funciones de desestructuración:

```kotlin
data class Punto(val x: Int, val y: Int)

val a = Punto(1, 2)
val b = Punto(1, 2)
println(a == b)            // true: equals compara por valor
println(a)                 // Punto(x=1, y=2)

val c = a.copy(y = 9)      // copia cambiando solo y → Punto(x=1, y=9)
val (x, y) = a             // desestructuración: x=1, y=2
```

`copy()` es esencial: como en Compose y MVVM trabajamos con estado **inmutable**, en vez de mutar un objeto creamos una copia modificada.

### Herencia: cerrada por defecto

En Kotlin las clases son `final` por defecto. Para poder heredar de una clase o sobrescribir un método debes marcarlos `open`:

```kotlin
open class Animal(val nombre: String) {
    open fun sonido(): String = "..."
}

class Perro(nombre: String) : Animal(nombre) {
    override fun sonido(): String = "Guau"
}
```

> [!info] Composición sobre herencia
> La herencia cerrada por defecto es deliberada: empuja a preferir composición e interfaces. En Android moderno apenas heredarás de clases tuyas; sí implementarás interfaces.

### Interfaces y delegación

```kotlin
interface Repositorio {
    fun guardar(dato: String)
    fun cargar(): String
}

// Delegación con 'by': delega la implementación en otro objeto
class RepositorioLogger(
    private val real: Repositorio,
) : Repositorio by real {
    override fun guardar(dato: String) {
        println("guardando: $dato")
        real.guardar(dato)
    }
}
```

`by` delega automáticamente todos los métodos de la interfaz al objeto indicado; solo sobrescribes los que quieras decorar.

### `enum` y `sealed class`

Un `enum` es un conjunto fijo de constantes. Una `sealed class` (o `sealed interface`) es una jerarquía **cerrada**: todas sus subclases se conocen en compilación, y pueden llevar datos distintos. Es la forma idiomática de modelar estados:

```kotlin
enum class Estado { CARGANDO, EXITO, ERROR }

sealed interface UiState {
    object Cargando : UiState
    data class Exito(val datos: List<String>) : UiState
    data class Error(val mensaje: String) : UiState
}

fun render(estado: UiState) = when (estado) {
    is UiState.Cargando -> "Spinner"
    is UiState.Exito    -> "Mostrar ${estado.datos.size} items"
    is UiState.Error    -> "Error: ${estado.mensaje}"
    // sin 'else': el compilador sabe que están todos los casos cubiertos
}
```

La **exhaustividad** del `when` sobre `sealed` es oro: si mañana añades un caso nuevo, el código deja de compilar hasta que lo manejes. Lo usaremos para el estado de UI en MVVM (página 11).

### `object` y `companion object`

`object` declara un **singleton** (una única instancia). `companion object` es el equivalente a los miembros estáticos de Java:

```kotlin
object Config {
    const val BASE_URL = "https://api.ejemplo.com"
}
Config.BASE_URL

class Usuario private constructor(val nombre: String) {
    companion object {
        fun crear(nombre: String) = Usuario(nombre)   // factory
    }
}
Usuario.crear("Ana")
```

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - El constructor primario va en la cabecera; las propiedades se declaran con `val`/`var` ahí mismo. No hay `new`.
> - `data class` genera `equals`/`hashCode`/`toString`/`copy` y desestructuración. `copy()` es clave para trabajar con estado inmutable.
> - Clases y métodos son `final` por defecto; marca `open` para heredar. Prefiere interfaces y composición (`by`).
> - `sealed class`/`sealed interface` modela jerarquías cerradas; combinada con `when` da exhaustividad comprobada en compilación — ideal para estados de UI.
> - `object` = singleton; `companion object` = miembros estáticos/factories.

### Para llevar a la práctica
- [ ] Modela una entidad de tu dominio con una `data class` y prueba `copy()`
- [ ] Crea una `sealed interface UiState` con `Cargando`, `Exito` y `Error`, y un `when` exhaustivo
- [ ] Declara un singleton de configuración con `object`
- [ ] Implementa una interfaz y decora una implementación con delegación `by`

### Recursos
- 🌐 kotlinlang.org/docs/classes.html — clases y herencia
- 🌐 kotlinlang.org/docs/sealed-classes.html — sealed classes e interfaces
- 📖 *Kotlin in Action* — capítulo 4

---
`#kotlin` `#poo` `#data-class` `#sealed-class`
