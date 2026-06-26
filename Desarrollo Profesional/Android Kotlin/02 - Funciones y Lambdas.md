# 📱 02 — Funciones y Lambdas

[[Desarrollo Profesional/Android Kotlin/Android Kotlin|⬅️ Volver al índice]] | [[Desarrollo Profesional/Android Kotlin/01 - Sintaxis Variables y Tipos|← 01]] | [[Desarrollo Profesional/Android Kotlin/03 - Clases y POO|03 →]]

> [!abstract] Introducción
> Las funciones en Kotlin son ciudadanos de primera clase: se pasan como argumentos, se devuelven y se guardan en variables. Esta página cubre desde la sintaxis básica hasta las lambdas, las funciones de orden superior, las funciones de extensión y las *scope functions* (`let`, `apply`, `run`…) — herramientas que verás constantemente en código Android.

## ¿De qué vamos a hablar?

Cómo declarar funciones legibles (con argumentos por defecto y con nombre), cómo escribir y pasar lambdas, y los idiomas de Kotlin que hacen el código tan compacto.

### Conceptos que vamos a cubrir
- Declaración de funciones y funciones de una expresión
- Argumentos por defecto y con nombre
- Lambdas y funciones de orden superior
- La convención del *trailing lambda*
- Funciones de extensión
- Scope functions: `let`, `apply`, `run`, `also`, `with`

---

## El Concepto

### Funciones básicas

```kotlin
fun sumar(a: Int, b: Int): Int {
    return a + b
}

// Función de una sola expresión: tipo de retorno inferido
fun multiplicar(a: Int, b: Int) = a * b

// Sin retorno útil: el tipo es Unit (equivalente a void), y se omite
fun saludar(nombre: String) {
    println("Hola, $nombre")
}
```

### Argumentos por defecto y con nombre

Eliminan la necesidad de sobrecargar funciones. Combinados, hacen las llamadas autoexplicativas:

```kotlin
fun crearUsuario(
    nombre: String,
    activo: Boolean = true,
    rol: String = "usuario",
) { /* ... */ }

crearUsuario("Andrés")                          // usa los defaults
crearUsuario("Ana", rol = "admin")              // argumento con nombre
crearUsuario("Leo", activo = false, rol = "invitado")
```

> [!tip] Usa argumentos con nombre para los booleanos
> `enviar(mensaje, true, false)` es ilegible. `enviar(mensaje, urgente = true, cifrado = false)` se explica solo.

### Lambdas y funciones de orden superior

Una **lambda** es una función anónima entre llaves. Una **función de orden superior** recibe o devuelve funciones.

```kotlin
// Lambda guardada en variable. Tipo: (Int, Int) -> Int
val suma = { a: Int, b: Int -> a + b }
println(suma(2, 3))   // 5

// Función que recibe otra función
fun operar(a: Int, b: Int, op: (Int, Int) -> Int): Int = op(a, b)

operar(4, 5, suma)                 // 9
operar(4, 5) { x, y -> x * y }     // 20  (trailing lambda)
```

**Convención del trailing lambda**: si el último parámetro es una función, la lambda va *fuera* de los paréntesis. Esto es la base de la sintaxis de Jetpack Compose. Dentro de una lambda de un solo parámetro, puedes usar `it`:

```kotlin
val numeros = listOf(1, 2, 3, 4)
val pares = numeros.filter { it % 2 == 0 }   // [2, 4]
```

### Funciones de extensión

Permiten **añadir** funciones a tipos existentes sin heredar ni modificar su código. Son omnipresentes en Kotlin y Android.

```kotlin
fun String.esEmail(): Boolean =
    contains("@") && contains(".")

"hola@ejemplo.com".esEmail()   // true

// Útil en Android: extender clases del framework
fun Int.dpToPx(densidad: Float): Int = (this * densidad).toInt()
```

### Scope functions

Cinco funciones de la librería estándar que ejecutan un bloque sobre un objeto. Diferencias clave: cómo se refiere al objeto (`it` o `this`) y qué devuelven.

| Función | Referencia | Devuelve | Uso típico |
|---------|-----------|----------|------------|
| `let` | `it` | resultado del bloque | ejecutar sobre no-nulos, transformar |
| `apply` | `this` | el propio objeto | configurar un objeto |
| `run` | `this` | resultado del bloque | configurar + calcular |
| `also` | `it` | el propio objeto | efectos secundarios (log) |
| `with` | `this` | resultado del bloque | agrupar llamadas a un objeto |

```kotlin
// let: ejecutar solo si no es null (lo veremos a fondo en la 05)
val longitud = nombre?.let { it.trim().length }

// apply: configurar y devolver el objeto
val intent = Intent(this, Detalle::class.java).apply {
    putExtra("id", 42)
    putExtra("titulo", "Hola")
}

// also: efecto secundario sin romper la cadena
val usuario = crearUsuario("Ana").also { Log.d("APP", "creado $it") }
```

`apply` es el rey en Android para configurar objetos (Intents, Paint, vistas) sin repetir el nombre de la variable.

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - Funciones de una expresión (`fun f() = ...`) para lógica corta; `Unit` es el "void" de Kotlin y se omite.
> - Argumentos por defecto y con nombre eliminan sobrecargas y hacen las llamadas legibles.
> - Las lambdas y funciones de orden superior son la base de la API funcional y de Compose. El *trailing lambda* va fuera de los paréntesis; `it` es el parámetro único implícito.
> - Las funciones de extensión añaden comportamiento a tipos ajenos sin herencia.
> - Scope functions: `apply` para configurar (devuelve el objeto), `let` para operar sobre no-nulos (devuelve el resultado).

### Para llevar a la práctica
- [ ] Reescribe una función con varias sobrecargas usando argumentos por defecto
- [ ] Escribe una función de extensión útil para `String` o `Int`
- [ ] Usa `apply` para configurar un objeto con varias propiedades
- [ ] Pasa una lambda como último argumento usando la sintaxis de trailing lambda

### Recursos
- 🌐 kotlinlang.org/docs/functions.html — funciones y lambdas oficiales
- 🌐 kotlinlang.org/docs/scope-functions.html — guía de las scope functions
- 📖 *Kotlin in Action* — capítulos 3, 5 y 8

---
`#kotlin` `#funciones` `#lambdas` `#extension-functions`
