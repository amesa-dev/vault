# 📱 12 — Datos: Room, Retrofit y Repositorios

[[Desarrollo Profesional/Android Kotlin/Android Kotlin|⬅️ Volver al índice]] | [[Desarrollo Profesional/Android Kotlin/11 - Arquitectura MVVM y ViewModel|← 11]] | [[Desarrollo Profesional/Android Kotlin/13 - Inyección de Dependencias y Testing|13 →]]

> [!abstract] Introducción
> Una app real lee y escribe datos: en la base de datos local y en una API remota. **Room** es la librería de persistencia sobre SQLite; **Retrofit** es el cliente HTTP estándar. Y el patrón **repositorio** los esconde tras una interfaz limpia para que el ViewModel no sepa de dónde vienen los datos. Esta página construye la capa de datos completa.

## ¿De qué vamos a hablar?

Cómo persistir con Room (entidades, DAO, base de datos), cómo llamar a una API con Retrofit, y cómo el patrón repositorio unifica ambas fuentes detrás de una interfaz.

### Conceptos que vamos a cubrir
- Room: `@Entity`, `@Dao`, `@Database`
- Room + Flow para datos reactivos
- Retrofit: interfaz de servicio y `suspend`
- DTOs vs modelos de dominio
- El patrón repositorio

---

## El Concepto

### Room: persistencia local

Room genera código a partir de anotaciones y verifica tus consultas SQL **en compilación**. Tres piezas:

```kotlin
// 1) Entidad: una tabla
@Entity(tableName = "tareas")
data class TareaEntity(
    @PrimaryKey(autoGenerate = true) val id: Int = 0,
    val titulo: String,
    val completada: Boolean = false,
)

// 2) DAO: las operaciones sobre esa tabla
@Dao
interface TareaDao {
    @Query("SELECT * FROM tareas ORDER BY id DESC")
    fun observarTodas(): Flow<List<TareaEntity>>   // reactivo: emite al cambiar

    @Insert
    suspend fun insertar(tarea: TareaEntity)

    @Update
    suspend fun actualizar(tarea: TareaEntity)

    @Delete
    suspend fun borrar(tarea: TareaEntity)
}

// 3) Base de datos
@Database(entities = [TareaEntity::class], version = 1)
abstract class AppDatabase : RoomDatabase() {
    abstract fun tareaDao(): TareaDao
}
```

Lo potente: si un `@Query` devuelve `Flow`, Room **re-emite automáticamente** cada vez que cambian los datos de la tabla. Combinado con `collectAsStateWithLifecycle` (página 09), la UI se actualiza sola al insertar o borrar. Las operaciones de escritura son `suspend`, así que corren fuera del hilo principal (página 06).

### Retrofit: cliente HTTP

Retrofit convierte una interfaz Kotlin anotada en llamadas HTTP. Con coroutines, los métodos son `suspend`:

```kotlin
// Modelo de la respuesta (DTO) — refleja el JSON de la API
data class TareaDto(val id: Int, val title: String, val completed: Boolean)

interface TareasApi {
    @GET("todos")
    suspend fun obtenerTareas(): List<TareaDto>

    @GET("todos/{id}")
    suspend fun obtenerTarea(@Path("id") id: Int): TareaDto

    @POST("todos")
    suspend fun crearTarea(@Body tarea: TareaDto): TareaDto
}

// Construcción del cliente (normalmente lo crea Hilt, página 13)
val api: TareasApi = Retrofit.Builder()
    .baseUrl("https://api.ejemplo.com/")
    .addConverterFactory(KotlinSerializationConverterFactory(...))  // JSON ↔ objetos
    .build()
    .create(TareasApi::class.java)
```

### DTO vs modelo de dominio

No uses los DTO de la API ni las entidades de Room directamente en la UI. Define un **modelo de dominio** limpio y convierte (mapea) entre capas:

```kotlin
// Modelo de dominio: lo que la app entiende, independiente de API y BD
data class Tarea(val id: Int, val titulo: String, val completada: Boolean)

fun TareaDto.toDomain() = Tarea(id, title, completed)
fun TareaEntity.toDomain() = Tarea(id, titulo, completada)
fun Tarea.toEntity() = TareaEntity(id, titulo, completada)
```

Así un cambio en el JSON de la API o en el esquema de la BD no se propaga a toda la app: solo tocas el mapeo.

### El patrón repositorio

El repositorio es la **única puerta a los datos** para el ViewModel. Esconde si vienen de red, de caché o de la BD, y decide la estrategia (p. ej. *offline-first*: la BD es la fuente de verdad y la red la refresca):

```kotlin
interface TareasRepository {
    fun observarTareas(): Flow<List<Tarea>>
    suspend fun refrescar()
    suspend fun completar(id: Int)
}

class TareasRepositoryImpl(
    private val dao: TareaDao,
    private val api: TareasApi,
) : TareasRepository {

    // La UI siempre lee de la BD local (rápida, funciona sin red)
    override fun observarTareas(): Flow<List<Tarea>> =
        dao.observarTodas().map { lista -> lista.map { it.toDomain() } }

    // Refrescar: trae de la API y actualiza la BD; la UI reacciona sola
    override suspend fun refrescar() {
        val remotas = api.obtenerTareas()
        remotas.forEach { dao.insertar(it.toDomain().toEntity()) }
    }

    override suspend fun completar(id: Int) { /* update en dao + api */ }
}
```

```
ViewModel ──► Repository ──┬──► Room  (fuente de verdad local)
                          └──► Retrofit (sincroniza con el servidor)
```

El ViewModel solo conoce la **interfaz** `TareasRepository`, no si por debajo hay Room o Retrofit. Eso permite cambiar la implementación o sustituirla por un *fake* en tests (página 13) sin tocar el ViewModel —es la inversión de dependencias de [[Desarrollo Profesional/Clean Code/Clean Code|SOLID]].

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - Room persiste sobre SQLite con `@Entity`/`@Dao`/`@Database` y verifica el SQL en compilación; un `@Query` que devuelve `Flow` re-emite al cambiar los datos.
> - Retrofit convierte una interfaz anotada (`@GET`/`@POST`, métodos `suspend`) en llamadas HTTP con (de)serialización de JSON.
> - No expongas DTOs ni entidades a la UI: define un modelo de **dominio** limpio y mapea entre capas.
> - El **repositorio** es la única puerta a los datos del ViewModel; oculta las fuentes y permite estrategias como *offline-first* (la BD como fuente de verdad).
> - El ViewModel depende de la **interfaz** del repositorio, no de su implementación: clave para sustituir fuentes y para testear.

### Para llevar a la práctica
- [ ] Define una `@Entity`, su `@Dao` con un `@Query` que devuelva `Flow` y la `@Database`
- [ ] Crea una `interface` Retrofit con un método `@GET suspend`
- [ ] Escribe funciones de mapeo `toDomain()`/`toEntity()` entre capas
- [ ] Implementa un repositorio que lea de Room y refresque desde Retrofit

### Recursos
- 🌐 developer.android.com/training/data-storage/room — Room
- 🌐 square.github.io/retrofit — Retrofit
- 🌐 developer.android.com/topic/architecture/data-layer — capa de datos y repositorios
- 🔗 [[Desarrollo Profesional/PostgreSQL/PostgreSQL|SQL]] y [[Desarrollo Profesional/RESTful/RESTful|RESTful]] — fundamentos de lo que hay debajo

---
`#android` `#room` `#retrofit` `#repositorio` `#datos`
