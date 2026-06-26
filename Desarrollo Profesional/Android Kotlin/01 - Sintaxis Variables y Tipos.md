# 📱 01 — Sintaxis, Variables y Tipos

[[Desarrollo Profesional/Android Kotlin/Android Kotlin|⬅️ Volver al índice]] | [[Desarrollo Profesional/Android Kotlin/00 - Introducción y Setup|← 00]] | [[Desarrollo Profesional/Android Kotlin/02 - Funciones y Lambdas|02 →]]

> [!abstract] Introducción
> La sintaxis de Kotlin es la mitad de larga que la de Java y mucho más segura. Esta página cubre lo básico que usarás en cada línea: cómo se declaran variables (y por qué `val` es tu opción por defecto), el sistema de tipos, y cómo en Kotlin estructuras como `if` y `when` son **expresiones** que devuelven un valor.

## ¿De qué vamos a hablar?

Repasamos variables, tipos primitivos, inferencia, strings con plantillas, y el control de flujo. La idea central a interiorizar: en Kotlin casi todo es una expresión.

### Conceptos que vamos a cubrir
- `val` vs `var`: inmutabilidad por defecto
- Tipos básicos e inferencia de tipos
- String templates y raw strings
- `if`, `when` y bucles como expresiones
- Rangos y la conversión explícita de tipos

---

## El Concepto

### `val` vs `var`

`val` declara una referencia de **solo lectura** (no reasignable); `var` una **mutable**. Empieza siempre por `val` y cambia a `var` solo cuando necesites reasignar. Esto hace tu código más fácil de razonar.

```kotlin
val nombre = "Andrés"   // inmutable: no se puede reasignar
var edad = 30           // mutable
edad = 31               // ✅ OK

// nombre = "Otro"      // ❌ error de compilación
```

> [!tip] `val` no significa "constante profunda"
> `val lista = mutableListOf(1, 2)` impide reasignar `lista`, pero sí puedes hacer `lista.add(3)`. `val` congela la *referencia*, no el contenido.

### Tipos básicos e inferencia

Kotlin infiere el tipo cuando es obvio, pero puedes anotarlo:

```kotlin
val entero: Int = 42
val largo = 42L              // Long
val decimal = 3.14          // Double
val flotante = 3.14f        // Float
val booleano = true         // Boolean
val caracter = 'A'          // Char
val texto: String = "hola"  // String
```

No hay conversión implícita entre tipos numéricos (a diferencia de Java). Hay que convertir explícitamente:

```kotlin
val i = 10
val l: Long = i.toLong()   // # BIEN: conversión explícita
// val l2: Long = i        // ❌ error: Int no es Long automáticamente
```

### String templates

Interpolas variables con `$` y expresiones con `${...}`:

```kotlin
val nombre = "Andrés"
val edad = 31
println("$nombre tiene $edad años")          // Andrés tiene 31 años
println("El año que viene tendrá ${edad + 1}")

// Raw string (multilínea, sin escapes)
val json = """
    {
        "nombre": "$nombre",
        "edad": $edad
    }
""".trimIndent()
```

### `if` y `when` son expresiones

En Kotlin `if` **devuelve un valor**, así que no existe el operador ternario: no hace falta.

```kotlin
val max = if (a > b) a else b   // if como expresión
```

`when` es el `switch` de Kotlin, pero mucho más potente. También es una expresión:

```kotlin
val tipo = when (codigo) {
    200 -> "OK"
    in 400..499 -> "Error de cliente"   // rangos
    in 500..599 -> "Error de servidor"
    else -> "Desconocido"
}

// when sin argumento: como cadena de if/else
val mensaje = when {
    edad < 18 -> "menor"
    edad < 65 -> "adulto"
    else -> "jubilado"
}
```

Cuando `when` cubre todos los casos de un `enum` o `sealed class`, el compilador no exige `else` (lo veremos en la página 03). Esa exhaustividad es una de las grandes fortalezas del lenguaje.

### Bucles y rangos

```kotlin
for (i in 1..5) println(i)          // 1 2 3 4 5 (rango cerrado)
for (i in 1 until 5) println(i)     // 1 2 3 4 (excluye el 5)
for (i in 10 downTo 1 step 2) {}    // 10 8 6 4 2
for (letra in "abc") println(letra) // itera caracteres

val numeros = listOf(10, 20, 30)
for ((indice, valor) in numeros.withIndex()) {
    println("$indice -> $valor")
}

var n = 5
while (n > 0) { n-- }
```

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - `val` (solo lectura) por defecto; `var` solo cuando necesites reasignar. Hace el código más predecible.
> - Kotlin infiere tipos, pero no convierte números implícitamente: usa `.toLong()`, `.toInt()`, etc.
> - String templates con `$variable` y `${expresión}`; raw strings con `"""..."""`.
> - `if` y `when` son **expresiones** que devuelven valor — por eso no hay operador ternario. `when` soporta rangos y condiciones.

### Para llevar a la práctica
- [ ] Reescribe un `switch` mental tuyo como `when` con rangos (`in 1..10`)
- [ ] Convierte una variable `var` que no reasignas a `val`
- [ ] Usa un string template para construir un mensaje con una operación dentro de `${}`
- [ ] Itera una lista con `withIndex()` y desestructuración

### Recursos
- 🌐 kotlinlang.org/docs/basic-syntax.html — sintaxis básica oficial
- 🌐 kotlinlang.org/docs/control-flow.html — `if`, `when` y bucles a fondo
- 📖 *Kotlin in Action* — capítulos 2 y 3

---
`#kotlin` `#sintaxis` `#tipos` `#control-de-flujo`
