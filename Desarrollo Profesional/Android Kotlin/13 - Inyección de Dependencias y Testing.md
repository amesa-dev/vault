# 📱 13 — Inyección de Dependencias y Testing

[[Desarrollo Profesional/Android Kotlin/Android Kotlin|⬅️ Volver al índice]] | [[Desarrollo Profesional/Android Kotlin/12 - Datos Room Retrofit y Repositorios|← 12]] | [[Desarrollo Profesional/Android Kotlin/14 - Trabajo en Segundo Plano y Notificaciones|14 →]]

> [!abstract] Introducción
> Llegamos al último escalón de una app profesional: cómo se ensamblan todas las piezas y cómo se prueban. La **inyección de dependencias** (con Hilt) hace que el ViewModel reciba su repositorio en vez de crearlo, lo que desacopla y facilita los tests. Y el **testing** —unitario y de UI— te da la confianza para refactorizar sin miedo. Esta página cierra el curso conectando arquitectura, DI y pruebas.

## ¿De qué vamos a hablar?

Por qué inyectar dependencias, cómo Hilt automatiza el cableado en Android, y cómo escribir tests del ViewModel, del repositorio y de la UI de Compose.

### Conceptos que vamos a cubrir
- Inyección de dependencias: el porqué
- Hilt: `@HiltAndroidApp`, `@Module`, `@Inject`, `@HiltViewModel`
- Tests unitarios del ViewModel con fakes
- Probar coroutines y Flow
- Tests de UI de Compose

---

## El Concepto

### El porqué de la inyección de dependencias

Comparar "crear" con "recibir" dependencias:

```kotlin
// MAL: el ViewModel crea sus dependencias → acoplado, imposible de testear aislado
class TareasViewModel : ViewModel() {
    private val repo = TareasRepositoryImpl(/* dao, api reales... */)
}

// BIEN: las recibe → desacoplado, en tests le pasas un fake
class TareasViewModel(
    private val repo: TareasRepository,   // depende de la interfaz
) : ViewModel()
```

La **inyección de dependencias** (DI) consiste en que cada objeto recibe lo que necesita desde fuera. Ya lo aplicamos en las páginas 11 y 12; ahora falta quién monta todo el grafo. Hacerlo a mano es tedioso, y por eso usamos Hilt.

### Hilt: DI automática para Android

Hilt (sobre Dagger) genera el cableado y conoce el ciclo de vida de Android (Application, Activity, ViewModel). Pasos:

```kotlin
// 1) Marca la Application
@HiltAndroidApp
class MiApp : Application()

// 2) Enseña a Hilt a construir lo que no controlas (Retrofit, Room)
@Module
@InstallIn(SingletonComponent::class)
object DataModule {

    @Provides @Singleton
    fun provideDatabase(@ApplicationContext ctx: Context): AppDatabase =
        Room.databaseBuilder(ctx, AppDatabase::class.java, "app.db").build()

    @Provides
    fun provideDao(db: AppDatabase): TareaDao = db.tareaDao()

    @Provides @Singleton
    fun provideApi(): TareasApi =
        Retrofit.Builder().baseUrl("https://api.ejemplo.com/").build()
            .create(TareasApi::class.java)
}

// 3) Vincula interfaz → implementación
@Module
@InstallIn(SingletonComponent::class)
abstract class RepoModule {
    @Binds
    abstract fun bindRepo(impl: TareasRepositoryImpl): TareasRepository
}

// 4) Marca lo tuyo con @Inject y @HiltViewModel
class TareasRepositoryImpl @Inject constructor(
    private val dao: TareaDao,
    private val api: TareasApi,
) : TareasRepository { /* ... */ }

@HiltViewModel
class TareasViewModel @Inject constructor(
    private val repo: TareasRepository,
) : ViewModel() { /* ... */ }
```

En la UI, obtienes el ViewModel ya cableado con `hiltViewModel()` (la Activity se marca con `@AndroidEntryPoint`):

```kotlin
@Composable
fun PantallaTareas(viewModel: TareasViewModel = hiltViewModel()) { /* ... */ }
```

Hilt resuelve toda la cadena (ViewModel → Repository → Dao + Api → Database) sin que la escribas a mano.

### Tests unitarios del ViewModel

Como el ViewModel recibe una **interfaz**, en el test le pasas un **fake** sin red ni base de datos:

```kotlin
class FakeRepo(private val datos: List<Tarea>) : TareasRepository {
    override fun observarTareas() = flowOf(datos)
    override suspend fun refrescar() {}
    override suspend fun completar(id: Int) {}
}

class TareasViewModelTest {
    @get:Rule val dispatcherRule = MainDispatcherRule()   // sustituye Dispatchers.Main

    @Test
    fun `carga muestra los datos del repositorio`() = runTest {
        val vm = TareasViewModel(FakeRepo(listOf(Tarea(1, "Test", false))))

        val estado = vm.uiState.value
        assertTrue(estado is TareasUiState.Exito)
        assertEquals(1, (estado as TareasUiState.Exito).tareas.size)
    }
}
```

`runTest` (de `kotlinx-coroutines-test`) ejecuta coroutines de forma controlada y salta los `delay`. La regla `MainDispatcherRule` reemplaza `Dispatchers.Main` por uno de test, ya que en JVM pura no existe el hilo principal de Android.

> [!tip] La pirámide de tests también aplica en Android
> Muchos tests unitarios rápidos (ViewModel, repositorio, mapeos), algunos de integración, y pocos de UI/end-to-end. Igual que en [[Desarrollo Profesional/Estrategia de Testing/Estrategia de Testing|Estrategia de Testing]].

### Tests de UI de Compose

Compose trae su propia API de testing que busca y actúa sobre nodos por texto o por `testTag`:

```kotlin
class PantallaTest {
    @get:Rule val composeRule = createComposeRule()

    @Test
    fun muestra_el_titulo_y_responde_al_click() {
        var pulsado = false
        composeRule.setContent {
            Contador(cuenta = 0, onIncrementar = { pulsado = true })
        }

        composeRule.onNodeWithText("Cuenta: 0").assertIsDisplayed()
        composeRule.onNodeWithText("Cuenta: 0").performClick()
        assertTrue(pulsado)
    }
}
```

Como diseñamos composables *stateless* (página 09), probarlos es trivial: les pasas un estado y verificas lo que pintan y los eventos que emiten, sin levantar la app entera.

### El cuadro completo

```
@HiltAndroidApp App
        │ Hilt cablea todo el grafo
        ▼
@AndroidEntryPoint Activity → setContent { NavHost }
        │                         │
        ▼                         ▼
 hiltViewModel()           composables (UI = f(estado))
        │ observa StateFlow
        ▼
   ViewModel ──► Repository ──► Room + Retrofit
   (test: fake)   (interfaz)     (datos)
```

Cada flecha es un punto de desacople: estado observable, interfaces e inyección. Eso es lo que hace una app Android moderna mantenible y testeable.

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - La DI hace que los objetos **reciban** sus dependencias (vía interfaces) en vez de crearlas: desacopla y permite tests con fakes.
> - Hilt automatiza el grafo: `@HiltAndroidApp` en la Application, `@Module`/`@Provides`/`@Binds` para enseñar a construir, `@Inject`/`@HiltViewModel`, y `hiltViewModel()` en la UI.
> - Testea el ViewModel con un fake del repositorio y `runTest` + `MainDispatcherRule` para coroutines y `StateFlow`.
> - Los composables *stateless* se prueban con `createComposeRule()` buscando nodos por texto/`testTag` y simulando clics.
> - Aplica la pirámide de tests: muchos unitarios, pocos de UI. El diseño en capas con interfaces es lo que lo hace posible.

### Para llevar a la práctica
- [ ] Configura Hilt: `@HiltAndroidApp`, un `@Module` con `@Provides` y `@HiltViewModel`
- [ ] Sustituye `viewModel()` por `hiltViewModel()` en una pantalla
- [ ] Escribe un test del ViewModel con un repositorio fake y `runTest`
- [ ] Escribe un test de UI de un composable stateless con `createComposeRule()`

### Recursos
- 🌐 developer.android.com/training/dependency-injection/hilt-android — Hilt
- 🌐 developer.android.com/jetpack/compose/testing — testing de Compose
- 🌐 developer.android.com/kotlin/coroutines/test — testear coroutines
- 🔗 [[Desarrollo Profesional/Estrategia de Testing/Estrategia de Testing|Estrategia de Testing]] y [[Desarrollo Profesional/Clean Code/Clean Code|Clean Code (SOLID)]] — los principios detrás

---
`#android` `#hilt` `#inyeccion-de-dependencias` `#testing`
