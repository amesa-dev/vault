# 📱 14 — Trabajo en Segundo Plano y Notificaciones

[[Desarrollo Profesional/Android Kotlin/Android Kotlin|⬅️ Volver al índice]] | [[Desarrollo Profesional/Android Kotlin/13 - Inyección de Dependencias y Testing|← 13]]

> [!abstract] Introducción
> Una app real no solo reacciona a toques: también hace cosas cuando nadie la mira —sincronizar datos, subir fotos, mandar un recordatorio. Pero Android es **muy** restrictivo con el segundo plano para ahorrar batería: no puedes simplemente lanzar un hilo y confiar en que siga vivo. Esta página cubre la pieza correcta para cada caso —**WorkManager** para trabajo diferible, servicios en primer plano para trabajo en curso, y **notificaciones** para avisar al usuario.

## ¿De qué vamos a hablar?

Por qué el segundo plano en Android es un campo minado, cuándo usar cada herramienta, cómo programar trabajo fiable con WorkManager y cómo mostrar notificaciones (incluido el permiso de Android 13+).

### Conceptos que vamos a cubrir
- Las restricciones de segundo plano y qué herramienta toca
- WorkManager: `Worker`, `WorkRequest`, restricciones
- Trabajo periódico y encadenado
- Notificaciones: canales y construcción
- El permiso `POST_NOTIFICATIONS` (Android 13+)

---

## El Concepto

### Por qué no puedes "lanzar un hilo y ya"

Desde Android 8, el sistema **mata** los procesos en segundo plano agresivamente y limita lo que pueden hacer (Doze, límites de ejecución). Un `CoroutineScope` propio o un hilo suelto muere cuando el sistema recoge tu proceso. Hay que usar las APIs que el sistema **respeta** y reanuda. La elección depende de la naturaleza del trabajo:

| Necesidad | Herramienta |
|-----------|-------------|
| Trabajo diferible y **garantizado** (sincronizar, subir backup) | **WorkManager** |
| Trabajo en curso que el usuario **ve** (reproducir música, ruta GPS) | Foreground Service + notificación |
| Tarea ligada a una pantalla viva (cargar datos) | `viewModelScope` (página 06) |
| Avisar al usuario de algo | Notificación |

La regla: si el trabajo **debe completarse** aunque la app se cierre o el móvil se reinicie, es WorkManager.

### WorkManager: trabajo diferible y garantizado

WorkManager persiste el trabajo y lo ejecuta respetando las restricciones del sistema; sobrevive a cierres de la app y reinicios. Defines un `Worker` con la tarea:

```kotlin
class SyncWorker(
    context: Context,
    params: WorkerParameters,
) : CoroutineWorker(context, params) {   // CoroutineWorker: doWork es suspend

    override suspend fun doWork(): Result = try {
        repositorio.refrescar()          // tu lógica (página 12), en Dispatchers.IO
        Result.success()
    } catch (e: Exception) {
        Result.retry()                   // WorkManager lo reintentará con backoff
    }
}
```

`doWork()` devuelve `success`, `failure` o `retry`. Si devuelves `retry`, WorkManager reprograma con *backoff* exponencial (mismo principio que [[Desarrollo Profesional/Patrones de Resiliencia/Patrones de Resiliencia|Patrones de Resiliencia]]).

Lo encolas con una `WorkRequest`, opcionalmente con **restricciones** (solo con red, solo cargando…):

```kotlin
val restricciones = Constraints.Builder()
    .setRequiredNetworkType(NetworkType.CONNECTED)   // solo con conexión
    .setRequiresCharging(true)                       // solo cargando
    .build()

val peticion = OneTimeWorkRequestBuilder<SyncWorker>()
    .setConstraints(restricciones)
    .build()

WorkManager.getInstance(context).enqueue(peticion)
```

### Trabajo periódico y encadenado

```kotlin
// Periódico: cada 6 horas (mínimo permitido: 15 minutos)
val sync = PeriodicWorkRequestBuilder<SyncWorker>(6, TimeUnit.HOURS).build()

WorkManager.getInstance(context).enqueueUniquePeriodicWork(
    "sync_diaria",
    ExistingPeriodicWorkPolicy.KEEP,   // no duplicar si ya existe
    sync,
)

// Encadenar: comprimir → subir → limpiar, en orden
WorkManager.getInstance(context)
    .beginWith(comprimir)
    .then(subir)
    .then(limpiar)
    .enqueue()
```

`enqueueUniquePeriodicWork` con una clave evita duplicar el trabajo si vuelves a encolarlo (p. ej. al reabrir la app). Puedes observar el estado de un trabajo como `Flow`/`LiveData` para reflejarlo en la UI.

### Notificaciones: canales primero

Desde Android 8 toda notificación pertenece a un **canal** (el usuario controla sonido/importancia por canal). Crea el canal una vez al arrancar:

```kotlin
fun crearCanal(context: Context) {
    val canal = NotificationChannel(
        "recordatorios",                       // id del canal
        "Recordatorios",                       // nombre visible
        NotificationManager.IMPORTANCE_DEFAULT,
    )
    context.getSystemService(NotificationManager::class.java)
        .createNotificationChannel(canal)
}
```

Y construyes la notificación con `NotificationCompat` (compatible hacia atrás):

```kotlin
fun mostrar(context: Context) {
    val notif = NotificationCompat.Builder(context, "recordatorios")
        .setSmallIcon(R.drawable.ic_notification)
        .setContentTitle("Tarea pendiente")
        .setContentText("Tienes una tarea sin completar")
        .setPriority(NotificationCompat.PRIORITY_DEFAULT)
        .setAutoCancel(true)                   // se cierra al pulsarla
        .build()

    NotificationManagerCompat.from(context).notify(1, notif)   // 1 = id
}
```

### El permiso `POST_NOTIFICATIONS` (Android 13+)

Desde Android 13, mostrar notificaciones requiere **permiso en runtime** (página 07). Decláralo en el manifest y pídelo:

```xml
<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
```

```kotlin
// En un composable: pedir el permiso antes de notificar
val launcher = rememberLauncherForActivityResult(
    ActivityResultContracts.RequestPermission()
) { concedido -> if (concedido) mostrar(context) }

Button(onClick = { launcher.launch(Manifest.permission.POST_NOTIFICATIONS) }) {
    Text("Activar avisos")
}
```

> [!warning] No abuses del segundo plano
> Cada trabajo periódico y cada notificación gasta batería y paciencia del usuario. Pide permiso de notificaciones **en contexto** (cuando aporta valor, no al arrancar), usa restricciones en WorkManager para no sincronizar sin red o sin batería, y agrupa avisos. Una app que vibra cada cinco minutos se desinstala.

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - Android restringe mucho el segundo plano: no lances hilos sueltos; usa la API que el sistema respeta y reanuda.
> - **WorkManager** para trabajo diferible y garantizado (sobrevive a cierres y reinicios): `CoroutineWorker` con `success`/`failure`/`retry`, restricciones (red, carga) y trabajo periódico/encadenado.
> - Foreground Service para trabajo en curso que el usuario ve; `viewModelScope` para lo ligado a una pantalla viva.
> - Toda notificación va en un **canal** (Android 8+); constrúyela con `NotificationCompat`.
> - Android 13+ exige el permiso en runtime `POST_NOTIFICATIONS`: decláralo y pídelo en contexto.

### Para llevar a la práctica
- [ ] Crea un `CoroutineWorker` que llame a `repositorio.refrescar()` y encólalo
- [ ] Programa un trabajo periódico con `enqueueUniquePeriodicWork` y una restricción de red
- [ ] Crea un canal de notificación y muestra una notificación con `NotificationCompat`
- [ ] Pide `POST_NOTIFICATIONS` en runtime antes de notificar (Android 13+)

### Recursos
- 🌐 developer.android.com/topic/libraries/architecture/workmanager — WorkManager
- 🌐 developer.android.com/develop/ui/views/notifications — notificaciones
- 🌐 developer.android.com/develop/background-work/background-tasks — guía de tareas en segundo plano (qué usar cuándo)
- 🔗 [[Desarrollo Profesional/Patrones de Resiliencia/Patrones de Resiliencia|Patrones de Resiliencia]] — el backoff de los reintentos

---
`#android` `#workmanager` `#notificaciones` `#segundo-plano`
