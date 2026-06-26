# 📱 06 — Coroutines y Flow

[[Desarrollo Profesional/Android Kotlin/Android Kotlin|⬅️ Volver al índice]] | [[Desarrollo Profesional/Android Kotlin/05 - Null Safety y Manejo de Errores|← 05]] | [[Desarrollo Profesional/Android Kotlin/07 - Fundamentos de Android|07 →]]

> [!abstract] Introducción
> En Android **no puedes bloquear el hilo principal**: si una llamada de red o de base de datos lo congela, la app se cuelga (un ANR). Las **coroutines** son la solución de Kotlin para la asincronía: código asíncrono que se *lee* como secuencial, sin callbacks anidados. Y **Flow** es su contrapartida para flujos de datos que emiten valores con el tiempo. Son la base de todo el Android moderno.

## ¿De qué vamos a hablar?

Qué es una función `suspend`, cómo se lanzan coroutines en un *scope*, cómo se eligen hilos con *dispatchers*, y cómo `Flow` y `StateFlow` modelan datos que cambian en el tiempo.

### Conceptos que vamos a cubrir
- Funciones `suspend` y por qué no bloquean
- Scopes y *structured concurrency*
- Dispatchers: en qué hilo corre cada cosa
- Concurrencia con `async`/`await`
- `Flow`, `StateFlow` y operadores

---

## El Concepto

### Funciones `suspend`

Una función `suspend` puede **pausarse** sin bloquear el hilo y reanudarse después. Solo se llama desde otra `suspend` o desde una coroutine:

```kotlin
suspend fun cargarUsuario(id: Int): Usuario {
    delay(1000)              // suspende 1s SIN bloquear el hilo
    return api.obtener(id)   // llamada de red (también suspend)
}
```

Mientras una coroutine está suspendida (esperando red, p. ej.), el hilo queda libre para otras tareas. Por eso miles de coroutines caben donde no cabrían miles de hilos.

```
Callback hell (MAL)              Coroutines (BIEN)
login(user) { token ->          val token = login(user)
  getPerfil(token) { perfil ->  val perfil = getPerfil(token)
    getAmigos(perfil) { ... }   val amigos = getAmigos(perfil)
  }                             // lineal, legible, con try/catch normal
}
```

### Scopes y concurrencia estructurada

Una coroutine vive dentro de un **scope**, que define su tiempo de vida. Si el scope se cancela, todas sus coroutines se cancelan —eso es *structured concurrency*, y evita fugas. En Android usarás scopes ya integrados:

```kotlin
// viewModelScope: se cancela solo cuando el ViewModel muere (página 11)
viewModelScope.launch {
    val usuario = cargarUsuario(42)   // suspend
    _estado.value = UiState.Exito(usuario)
}
```

- `launch` lanza una coroutine que no devuelve resultado (dispara y olvida).
- `runBlocking` bloquea el hilo hasta terminar (solo para `main`/tests, **nunca** en Android de producción).

### Dispatchers: el hilo donde corre

Un *dispatcher* decide en qué hilo se ejecuta la coroutine:

| Dispatcher | Para qué |
|------------|----------|
| `Dispatchers.Main` | UI; tocar la pantalla. Es el hilo principal |
| `Dispatchers.IO` | red, disco, base de datos (operaciones de E/S) |
| `Dispatchers.Default` | cálculo intensivo de CPU |

```kotlin
suspend fun procesar() {
    val datos = withContext(Dispatchers.IO) {   // cambia a hilo de E/S
        leerDeDisco()
    }
    // de vuelta en el contexto original
    actualizarUi(datos)
}
```

> [!tip] Las librerías ya cambian de hilo por ti
> Retrofit y Room (página 12) usan `Dispatchers.IO` internamente para sus funciones `suspend`. Normalmente solo necesitas `withContext(Dispatchers.IO)` para tu propio código de E/S.

### Concurrencia con `async`/`await`

Para lanzar varias tareas **en paralelo** y esperar sus resultados:

```kotlin
suspend fun cargarPantalla() = coroutineScope {
    val perfil = async { api.perfil() }      // arranca ya
    val amigos = async { api.amigos() }      // arranca en paralelo
    Pantalla(perfil.await(), amigos.await()) // espera ambos
}
```

Las dos llamadas corren a la vez; el total tarda lo que la más lenta, no la suma.

### `Flow`: streams asíncronos

Una `suspend` devuelve **un** valor. Un `Flow` emite **varios** a lo largo del tiempo (resultados de una búsqueda, lecturas de un sensor, filas de la base de datos que cambian):

```kotlin
fun temperaturas(): Flow<Int> = flow {
    while (true) {
        emit(leerSensor())   // emite un valor
        delay(1000)
    }
}

// Se "colecciona" dentro de una coroutine
viewModelScope.launch {
    temperaturas()
        .filter { it > 0 }
        .map { "$it ºC" }
        .collect { valor -> println(valor) }
}
```

`Flow` es **frío**: no produce nada hasta que alguien lo colecciona, y soporta los mismos operadores que las colecciones (`map`, `filter`…).

### `StateFlow`: estado observable

`StateFlow` es un `Flow` **caliente** que siempre tiene un valor actual. Es la herramienta estándar para exponer el estado de la UI desde un ViewModel (página 11):

```kotlin
private val _contador = MutableStateFlow(0)
val contador: StateFlow<Int> = _contador.asStateFlow()

fun incrementar() {
    _contador.value += 1   // emite el nuevo valor a quien observe
}
```

La UI de Compose observa este `StateFlow` con `collectAsStateWithLifecycle()` y se recompone cuando cambia (lo enlazamos en la página 09).

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - Una función `suspend` se pausa sin bloquear el hilo; convierte el "callback hell" en código secuencial legible.
> - Las coroutines viven en un *scope* (`viewModelScope`) que las cancela en bloque: concurrencia estructurada, sin fugas.
> - Los dispatchers eligen el hilo: `Main` (UI), `IO` (red/disco/BD), `Default` (CPU). `withContext(Dispatchers.IO)` para tu E/S.
> - `async`/`await` ejecutan tareas en paralelo y esperan sus resultados.
> - `Flow` emite varios valores en el tiempo (frío); `StateFlow` mantiene un valor actual observable: la base del estado de UI en MVVM.

### Para llevar a la práctica
- [ ] Escribe una función `suspend` con `delay` y lánzala desde un `launch`
- [ ] Carga dos recursos en paralelo con `async`/`await` y compara el tiempo
- [ ] Crea un `Flow` con `flow { emit(...) }` y colecciónalo con un `map` en medio
- [ ] Expón un contador como `StateFlow` con un `MutableStateFlow` privado

### Recursos
- 🌐 developer.android.com/kotlin/coroutines — coroutines en Android (guía oficial)
- 🌐 kotlinlang.org/docs/flow.html — `Flow` en profundidad
- 🌐 developer.android.com/kotlin/flow/stateflow-and-sharedflow — `StateFlow`
- 📖 *Kotlin Coroutines: Deep Dive* — Marcin Moskała

---
`#kotlin` `#coroutines` `#flow` `#asincronia` `#android`
