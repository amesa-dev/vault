# 📱 09 — Estado y Recomposición

[[Desarrollo Profesional/Android Kotlin/Android Kotlin|⬅️ Volver al índice]] | [[Desarrollo Profesional/Android Kotlin/08 - Jetpack Compose UI Declarativa|← 08]] | [[Desarrollo Profesional/Android Kotlin/10 - Navegación|10 →]]

> [!abstract] Introducción
> El estado es el corazón de Compose. La UI es una **función del estado**: cuando el estado cambia, Compose vuelve a ejecutar (recompone) los composables afectados y la pantalla se actualiza sola. Esta página explica `remember`, `mutableStateOf`, el patrón de *state hoisting* (subir el estado) y cómo lanzar efectos secundarios sin romper la recomposición.

## ¿De qué vamos a hablar?

Qué es la recomposición, cómo se declara y recuerda el estado, por qué hay que "subir" el estado, cómo sobrevivir a la rotación con `rememberSaveable`, y cómo ejecutar efectos con `LaunchedEffect`.

### Conceptos que vamos a cubrir
- Recomposición: UI = f(estado)
- `mutableStateOf` y `remember`
- `rememberSaveable` para sobrevivir cambios de configuración
- State hoisting: estado arriba, eventos abajo
- Observar `StateFlow` desde Compose
- Efectos secundarios: `LaunchedEffect`

---

## El Concepto

### La UI es función del estado

En Compose no manipulas widgets ("pon este texto", "oculta este botón"). Declaras cómo se ve la UI **para el estado actual**, y cuando el estado cambia, Compose recompone:

```
estado cambia → Compose re-ejecuta los composables que lo leen → UI actualizada
```

### `mutableStateOf` + `remember`

`mutableStateOf` crea un estado **observable**: Compose detecta sus lecturas y recompone al cambiar. `remember` lo **conserva** entre recomposiciones (sin él, se reiniciaría en cada render):

```kotlin
@Composable
fun Contador() {
    var cuenta by remember { mutableStateOf(0) }   // 'by' para leer/escribir directo

    Button(onClick = { cuenta++ }) {
        Text("Has pulsado $cuenta veces")
    }
}
```

Al pulsar, `cuenta++` cambia el estado → Compose recompone el `Button` → el texto se actualiza. Sin `remember`, `cuenta` volvería a 0 en cada recomposición.

> [!warning] `remember` no sobrevive a la rotación
> `remember` se mantiene entre recomposiciones, pero se pierde si la Activity se recrea (rotar). Para estado de UI sencillo que deba sobrevivir, usa `rememberSaveable`.

```kotlin
var cuenta by rememberSaveable { mutableStateOf(0) }   // sobrevive a la rotación
```

### State hoisting: estado arriba, eventos abajo

El patrón clave de Compose: un composable que **no posee** su estado es *stateless* —recibe el valor y notifica los cambios mediante un callback. Subir el estado al padre (hoisting) lo hace reutilizable y testeable:

```kotlin
// Stateless: no tiene estado propio; recibe valor y emite eventos
@Composable
fun Contador(cuenta: Int, onIncrementar: () -> Unit) {
    Button(onClick = onIncrementar) {
        Text("Cuenta: $cuenta")
    }
}

// El padre posee el estado y lo pasa hacia abajo
@Composable
fun Pantalla() {
    var cuenta by remember { mutableStateOf(0) }
    Contador(cuenta = cuenta, onIncrementar = { cuenta++ })
}
```

Esto es **unidirectional data flow**: el estado fluye hacia abajo (`cuenta`), los eventos hacia arriba (`onIncrementar`). Misma idea que props + callbacks en React.

### Estado inmutable: por qué `copy()`

Para estado complejo, usa una `data class` y reemplázala entera con `copy()` (página 03) en vez de mutar campos. Compose detecta el cambio de referencia y recompone:

```kotlin
data class FormState(val nombre: String = "", val email: String = "")

var form by remember { mutableStateOf(FormState()) }
// al escribir en el campo nombre:
form = form.copy(nombre = nuevoTexto)
```

### Observar `StateFlow` desde Compose

El estado real de la app suele vivir en un ViewModel como `StateFlow` (página 06). Compose lo observa y se recompone con `collectAsStateWithLifecycle()`:

```kotlin
@Composable
fun PantallaTareas(viewModel: TareasViewModel) {
    val estado by viewModel.uiState.collectAsStateWithLifecycle()

    when (estado) {
        is UiState.Cargando -> CircularProgressIndicator()
        is UiState.Exito    -> ListaTareas(estado.datos)
        is UiState.Error    -> Text(estado.mensaje)
    }
}
```

`collectAsStateWithLifecycle` deja de observar cuando la pantalla no está visible —eficiente y consciente del ciclo de vida. Lo enlazaremos con el ViewModel en la página 11.

### Efectos secundarios: `LaunchedEffect`

Si necesitas lanzar una coroutine cuando un composable aparece (cargar datos, arrancar una animación), usa `LaunchedEffect`. Se ejecuta al entrar en composición y se cancela al salir:

```kotlin
@Composable
fun Pantalla(id: Int) {
    LaunchedEffect(id) {          // se relanza si 'id' cambia
        viewModel.cargar(id)     // llamada suspend
    }
    // ...
}
```

La *key* (`id`) funciona como el array de dependencias de `useEffect` en React: si cambia, el efecto se cancela y se relanza.

> [!tip] No llames a funciones suspend directamente en el cuerpo del composable
> El cuerpo de un composable se ejecuta muchas veces (en cada recomposición). Los efectos y las coroutines van en `LaunchedEffect`, no sueltos.

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - En Compose, UI = f(estado): al cambiar el estado, los composables que lo leen se recomponen.
> - `mutableStateOf` crea estado observable; `remember` lo conserva entre recomposiciones; `rememberSaveable` además sobrevive a la rotación.
> - *State hoisting*: sube el estado al padre y deja los hijos *stateless* (valor + callback). Estado abajo, eventos arriba (UDF).
> - Para estado complejo usa `data class` + `copy()`: el cambio de referencia dispara la recomposición.
> - Observa el `StateFlow` del ViewModel con `collectAsStateWithLifecycle()`. Lanza coroutines/efectos con `LaunchedEffect(key)`, nunca sueltos en el cuerpo.

### Para llevar a la práctica
- [ ] Construye un contador con `remember { mutableStateOf(0) }` y luego pásalo a `rememberSaveable`
- [ ] Refactoriza un composable con estado a *stateless* + hoisting (valor + callback)
- [ ] Gestiona un formulario con una `data class` y `copy()`
- [ ] Carga datos al entrar en una pantalla con `LaunchedEffect`

### Recursos
- 🌐 developer.android.com/jetpack/compose/state — estado en Compose
- 🌐 developer.android.com/jetpack/compose/side-effects — efectos secundarios
- 🌐 developer.android.com/jetpack/compose/mental-model — modelo mental de Compose

---
`#android` `#compose` `#estado` `#recomposicion`
