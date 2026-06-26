# 📱 11 — Arquitectura MVVM y ViewModel

[[Desarrollo Profesional/Android Kotlin/Android Kotlin|⬅️ Volver al índice]] | [[Desarrollo Profesional/Android Kotlin/10 - Navegación|← 10]] | [[Desarrollo Profesional/Android Kotlin/12 - Datos Room Retrofit y Repositorios|12 →]]

> [!abstract] Introducción
> Una pantalla con toda la lógica metida en el composable es imposible de testear y se rompe al rotar. La arquitectura recomendada por Google separa responsabilidades en capas, con el **ViewModel** como pieza central: contiene el estado y la lógica de la UI, sobrevive a los cambios de configuración y expone el estado como un `StateFlow`. Esta página une todo lo anterior en una arquitectura coherente.

## ¿De qué vamos a hablar?

El patrón MVVM en Android, qué es y qué no es un ViewModel, cómo se modela el `UiState`, el flujo unidireccional de datos, y las capas de una app moderna (UI → dominio → datos).

### Conceptos que vamos a cubrir
- MVVM y la arquitectura por capas de Google
- El `ViewModel` y `viewModelScope`
- Modelar el `UiState` con `StateFlow`
- Unidirectional Data Flow (UDF)
- Conectar ViewModel y Compose

---

## El Concepto

### Las capas

La guía de arquitectura de Android propone separar la app en capas con dependencias en una sola dirección:

```
┌─────────────────────────────────────────┐
│  UI Layer                                 │
│   Composables  ──observa──►  ViewModel    │
│                  ◄──eventos──             │
├─────────────────────────────────────────┤
│  Domain Layer (opcional)                  │
│   Use Cases (lógica de negocio reutilizable)
├─────────────────────────────────────────┤
│  Data Layer                               │
│   Repository ──► fuentes (Room, Retrofit) │
└─────────────────────────────────────────┘
```

- **UI**: composables (pintan) + ViewModel (estado y lógica de presentación).
- **Dominio** (opcional): *use cases* que encapsulan reglas de negocio.
- **Datos**: repositorios que coordinan fuentes (página 12).

La regla de oro: las capas superiores dependen de las inferiores, **nunca al revés**. El composable conoce el ViewModel, pero el ViewModel no sabe nada de Compose.

### El ViewModel

Un `ViewModel` mantiene el estado y la lógica de la UI, y **sobrevive a los cambios de configuración** (rotar no lo destruye). Resuelve el problema de la página 07:

```kotlin
class TareasViewModel(
    private val repositorio: TareasRepository,
) : ViewModel() {

    // Estado privado mutable / público de solo lectura
    private val _uiState = MutableStateFlow<TareasUiState>(TareasUiState.Cargando)
    val uiState: StateFlow<TareasUiState> = _uiState.asStateFlow()

    init {
        cargarTareas()
    }

    fun cargarTareas() {
        viewModelScope.launch {            // se cancela solo al morir el ViewModel
            _uiState.value = TareasUiState.Cargando
            _uiState.value = try {
                TareasUiState.Exito(repositorio.obtenerTareas())
            } catch (e: Exception) {
                TareasUiState.Error(e.message ?: "Error")
            }
        }
    }

    fun completar(id: Int) {
        viewModelScope.launch { repositorio.completar(id) }
    }
}
```

> [!warning] Qué NO meter en un ViewModel
> Nunca guardes referencias a composables, `Context` de Activity o vistas: provocan fugas de memoria porque el ViewModel vive más que la pantalla. El ViewModel solo conoce datos y casos de uso.

### Modelar el `UiState`

Representa **todo** lo que la pantalla necesita pintar en un único objeto, normalmente una `sealed interface` (página 03) o una `data class`:

```kotlin
sealed interface TareasUiState {
    data object Cargando : TareasUiState
    data class Exito(val tareas: List<Tarea>) : TareasUiState
    data class Error(val mensaje: String) : TareasUiState
}
```

Así la UI solo hace un `when` exhaustivo sobre el estado y se acabaron los "¿está cargando? ¿hay error? ¿hay datos?" sueltos. Para pantallas con varios campos, una sola `data class` con todos los flags también es habitual:

```kotlin
data class FormUiState(
    val texto: String = "",
    val enviando: Boolean = false,
    val error: String? = null,
)
```

### Unidirectional Data Flow

El flujo es un círculo en una sola dirección, idéntico en espíritu al state hoisting de la página 09 pero a escala de pantalla:

```
Usuario actúa → evento al ViewModel → ViewModel actualiza el StateFlow
      ▲                                              │
      └────────  Compose recompone con el nuevo estado  ◄──┘
```

- **Estado** baja: del ViewModel a la UI (vía `StateFlow`).
- **Eventos** suben: de la UI al ViewModel (llamando a sus funciones).

### Conectar ViewModel y Compose

Compose obtiene el ViewModel con `viewModel()` (o `hiltViewModel()`, página 13) y observa su estado:

```kotlin
@Composable
fun PantallaTareas(viewModel: TareasViewModel = viewModel()) {
    val estado by viewModel.uiState.collectAsStateWithLifecycle()

    when (val s = estado) {
        is TareasUiState.Cargando -> CircularProgressIndicator()
        is TareasUiState.Exito    -> ListaTareas(
            tareas = s.tareas,
            onCompletar = viewModel::completar,   // evento sube al ViewModel
        )
        is TareasUiState.Error    -> Text(s.mensaje)
    }
}
```

Fíjate: el composable no tiene lógica de negocio. Observa estado, lo pinta y reenvía eventos. Eso lo hace fácil de previsualizar y testear, y el ViewModel se testea sin tocar la UI.

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - Separa la app en capas (UI → dominio → datos) con dependencias en una sola dirección; el ViewModel no conoce Compose.
> - El `ViewModel` guarda estado y lógica de presentación, sobrevive a la rotación y lanza trabajo en `viewModelScope`. Nunca guardes Context/vistas en él.
> - Expón el estado como un `StateFlow` (privado mutable + público de solo lectura) y modélalo con `sealed interface`/`data class` (`UiState`).
> - UDF: el estado baja a la UI, los eventos suben al ViewModel.
> - El composable observa con `collectAsStateWithLifecycle()`, hace un `when` exhaustivo y reenvía eventos: sin lógica de negocio.

### Para llevar a la práctica
- [ ] Crea un `ViewModel` que exponga un `StateFlow<UiState>` con `Cargando`/`Exito`/`Error`
- [ ] Mueve la lógica de una pantalla del composable al ViewModel
- [ ] Conecta la UI con `viewModel()` + `collectAsStateWithLifecycle()` y un `when` exhaustivo
- [ ] Reenvía un evento de la UI con una referencia a método (`viewModel::accion`)

### Recursos
- 🌐 developer.android.com/topic/architecture — guía de arquitectura de apps
- 🌐 developer.android.com/topic/libraries/architecture/viewmodel — ViewModel
- 🌐 developer.android.com/topic/architecture/ui-layer — capa de UI y UI State
- 🔗 [[Desarrollo Profesional/Clean Code/Clean Code|Clean Code]] y [[Desarrollo Profesional/Arquitectura Hexagonal/Arquitectura Hexagonal|Arquitectura Hexagonal]] — los principios de fondo

---
`#android` `#mvvm` `#viewmodel` `#arquitectura` `#stateflow`
