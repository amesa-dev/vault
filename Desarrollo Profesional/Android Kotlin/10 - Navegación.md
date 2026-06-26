# 📱 10 — Navegación

[[Desarrollo Profesional/Android Kotlin/Android Kotlin|⬅️ Volver al índice]] | [[Desarrollo Profesional/Android Kotlin/09 - Estado y Recomposición|← 09]] | [[Desarrollo Profesional/Android Kotlin/11 - Arquitectura MVVM y ViewModel|11 →]]

> [!abstract] Introducción
> Casi toda app tiene varias pantallas y hay que moverse entre ellas pasando datos y respetando el botón Atrás. **Navigation Compose** resuelve esto con un grafo de destinos declarado en Kotlin. Esta página cubre cómo definir rutas, navegar entre ellas, pasar argumentos y gestionar el *back stack*.

## ¿De qué vamos a hablar?

El `NavController` y el `NavHost`, cómo se definen rutas type-safe, cómo navegar pasando argumentos, y cómo controlar la pila de navegación (volver, no acumular pantallas).

### Conceptos que vamos a cubrir
- `NavController` y `NavHost`
- Definir destinos y rutas
- Navegar y pasar argumentos
- El back stack y `popUpTo`
- Rutas type-safe con objetos serializables

---

## El Concepto

### Las piezas: `NavController` y `NavHost`

- **`NavController`**: el objeto que ejecuta la navegación (navegar, volver). Se crea con `rememberNavController()`.
- **`NavHost`**: el contenedor que mapea cada **ruta** a un composable. Es el grafo de navegación.

```kotlin
@Composable
fun MiApp() {
    val navController = rememberNavController()

    NavHost(navController = navController, startDestination = "lista") {
        composable("lista") {
            PantallaLista(
                onTareaClick = { id -> navController.navigate("detalle/$id") }
            )
        }
        composable("detalle/{id}") { backStackEntry ->
            val id = backStackEntry.arguments?.getString("id")?.toInt() ?: 0
            PantallaDetalle(id)
        }
    }
}
```

`startDestination` es la pantalla inicial. Cada `composable("ruta")` registra un destino.

### Navegar y pasar argumentos

Para navegar, llamas a `navController.navigate("ruta")`. Los argumentos viajan en la propia ruta (como una URL):

```kotlin
// Definir la ruta con un parámetro tipado
composable(
    route = "detalle/{id}",
    arguments = listOf(navArgument("id") { type = NavType.IntType })
) { entry ->
    val id = entry.arguments?.getInt("id") ?: 0
    PantallaDetalle(id)
}

// Navegar pasando el valor
navController.navigate("detalle/42")
```

> [!tip] No pases objetos grandes por la ruta
> Las rutas son para **identificadores** (un `id`), no para objetos completos. Pasa el `id` y deja que la pantalla destino cargue el dato desde su ViewModel/repositorio.

### El back stack y `popUpTo`

Cada `navigate` apila un destino. El botón Atrás del sistema hace `popBackStack()` automáticamente. A veces quieres controlar la pila:

```kotlin
// Volver atrás manualmente
navController.popBackStack()

// Navegar limpiando la pila hasta un destino (típico tras un login)
navController.navigate("home") {
    popUpTo("login") { inclusive = true }   // elimina login del back stack
}

// Evitar duplicar la misma pantalla al pulsar repetido (p. ej. bottom bar)
navController.navigate("perfil") {
    launchSingleTop = true
}
```

```
Pila tras login → navigate("home") con popUpTo("login", inclusive):

   [login] [home]   →   navigate   →   [home]
   (login eliminado: Atrás no vuelve al login)
```

### Rutas type-safe (recomendado hoy)

Las rutas como String son frágiles (un typo no falla hasta runtime). Navigation Compose permite definir destinos como **objetos serializables** con verificación en compilación:

```kotlin
@Serializable
data class Detalle(val id: Int)      // ruta tipada con su argumento

@Serializable
object Lista                          // ruta sin argumentos

NavHost(navController, startDestination = Lista) {
    composable<Lista> {
        PantallaLista(onClick = { id -> navController.navigate(Detalle(id)) })
    }
    composable<Detalle> { entry ->
        val detalle: Detalle = entry.toRoute()
        PantallaDetalle(detalle.id)
    }
}
```

Con este enfoque el compilador comprueba los argumentos: si la ruta espera un `Int` y pasas otra cosa, no compila. Es la forma preferida en proyectos nuevos.

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - `rememberNavController()` crea el `NavController`; el `NavHost` mapea cada ruta a un composable, empezando por `startDestination`.
> - Navegas con `navController.navigate("ruta")`; los argumentos viajan en la ruta como en una URL y se leen del `backStackEntry`.
> - Pasa **identificadores**, no objetos grandes: el destino carga el dato desde su ViewModel/repositorio.
> - El back stack se gestiona solo con Atrás; usa `popUpTo`/`inclusive` (p. ej. tras login) y `launchSingleTop` para no duplicar pantallas.
> - Prefiere **rutas type-safe** con objetos `@Serializable` y `composable<T>`: el compilador verifica los argumentos.

### Para llevar a la práctica
- [ ] Monta un `NavHost` con dos pantallas y navega de una a otra
- [ ] Pasa un `id` como argumento y léelo en la pantalla destino
- [ ] Implementa un flujo de login con `popUpTo(..., inclusive = true)`
- [ ] Convierte tus rutas String a rutas type-safe con objetos `@Serializable`

### Recursos
- 🌐 developer.android.com/jetpack/compose/navigation — Navigation Compose
- 🌐 developer.android.com/guide/navigation/design/type-safety — rutas type-safe
- 🌐 developer.android.com/guide/navigation/backstack — el back stack

---
`#android` `#compose` `#navegacion` `#navigation`
