# 📱 07 — Fundamentos de Android

[[Desarrollo Profesional/Android Kotlin/Android Kotlin|⬅️ Volver al índice]] | [[Desarrollo Profesional/Android Kotlin/06 - Coroutines y Flow|← 06]] | [[Desarrollo Profesional/Android Kotlin/08 - Jetpack Compose UI Declarativa|08 →]]

> [!abstract] Introducción
> Antes de pintar pantallas conviene entender la plataforma: qué es una `Activity`, cómo Android arranca y destruye tu app, qué declara el `AndroidManifest`, y cómo funcionan los recursos y los permisos. Aunque con Compose escribirás casi todo en Kotlin, estos cimientos explican *dónde* vive tu UI y por qué el ciclo de vida importa tanto.

## ¿De qué vamos a hablar?

Los componentes de una app Android, el ciclo de vida de una `Activity` (y por qué provoca tantos bugs), el manifest, el sistema de recursos y el modelo de permisos.

### Conceptos que vamos a cubrir
- Componentes de una app: Activity, Service, Broadcast, Provider
- La `Activity` y su ciclo de vida
- El cambio de configuración (rotación) y la pérdida de estado
- `AndroidManifest.xml`
- Recursos (`res/`) y permisos en runtime

---

## El Concepto

### Los cuatro componentes

Una app Android se compone de cuatro tipos de componentes que el sistema puede arrancar:

| Componente | Para qué |
|------------|----------|
| **Activity** | Una pantalla con la que el usuario interactúa |
| **Service** | Trabajo en segundo plano sin UI (reproducir música, sincronizar) |
| **BroadcastReceiver** | Reacciona a eventos del sistema (batería baja, conectividad) |
| **ContentProvider** | Comparte datos entre apps (contactos, fotos) |

En el día a día con Compose trabajarás sobre todo con **una sola Activity** que hospeda toda la UI.

### La Activity y su ciclo de vida

Una `Activity` es el punto de entrada de la UI. El sistema la lleva por una serie de estados según el usuario y la memoria disponible:

```
        onCreate()      ← se crea: aquí montas la UI
            ↓
        onStart()       ← visible
            ↓
        onResume()      ← en primer plano, interactiva
            ↓
       [ usuario usa la app ]
            ↓
        onPause()       ← pierde el foco (llega otra pantalla)
            ↓
        onStop()        ← ya no visible
            ↓
        onDestroy()     ← se destruye
```

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {            // aquí enchufamos Jetpack Compose
            MiApp()
        }
    }
}
```

### El gran "gotcha": cambios de configuración

Cuando rotas el dispositivo (o cambias idioma, tema oscuro…), Android **destruye y recrea** la Activity entera. Si guardaste estado en una variable normal de la Activity, **se pierde**.

```kotlin
// MAL: este contador vuelve a 0 al rotar la pantalla
class MainActivity : ComponentActivity() {
    var contador = 0
}
```

La solución moderna: guardar el estado en un **ViewModel** (página 11), que sobrevive a los cambios de configuración, y en Compose usar `rememberSaveable` (página 09). Entender este problema es la razón de ser de media arquitectura de Android.

> [!info] Una app, muchas muertes
> Android puede matar tu proceso en segundo plano para liberar memoria y recrearlo después. Diseña asumiendo que tu UI es desechable y que el estado debe poder reconstruirse.

### `AndroidManifest.xml`

Es el documento de identidad de la app. Declara componentes, permisos y metadatos:

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android">

    <uses-permission android:name="android.permission.INTERNET" />

    <application
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:theme="@style/Theme.MiApp">

        <activity
            android:name=".MainActivity"
            android:exported="true">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
    </application>
</manifest>
```

El `intent-filter` con `MAIN`/`LAUNCHER` marca qué Activity aparece como icono en el menú.

### Recursos: `res/`

Android separa los textos, colores e imágenes del código, en `res/`. Esto permite traducir y adaptar a cada dispositivo sin tocar Kotlin:

```xml
<!-- res/values/strings.xml -->
<resources>
    <string name="app_name">Mi App</string>
    <string name="saludo">Hola, %1$s</string>
</resources>
```

```kotlin
// En Compose se accede con stringResource
Text(text = stringResource(R.string.saludo, nombre))
```

Carpetas con sufijo se eligen automáticamente: `values-en/` (inglés), `drawable-night/` (modo oscuro), `values-es/` (español). El sistema escoge la variante según el dispositivo.

### Permisos en runtime

Para acceso sensible (cámara, ubicación, contactos) no basta declararlo en el manifest: hay que **pedirlo en ejecución** y el usuario puede negarlo:

```kotlin
// Con la API de Activity Result (en Compose hay un helper equivalente)
val launcher = rememberLauncherForActivityResult(
    ActivityResultContracts.RequestPermission()
) { concedido ->
    if (concedido) abrirCamara() else mostrarMensaje()
}

launcher.launch(Manifest.permission.CAMERA)
```

Pide el permiso **en el momento** en que lo necesitas y maneja siempre el caso de que lo nieguen.

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - Una app tiene cuatro componentes (Activity, Service, BroadcastReceiver, ContentProvider); con Compose trabajas sobre todo con una Activity.
> - La `Activity` tiene un ciclo de vida (`onCreate`→`onResume`→`onPause`→`onDestroy`); en `onCreate` montas la UI con `setContent { }`.
> - Un cambio de configuración (rotar) **destruye y recrea** la Activity: el estado en variables se pierde. Por eso existen ViewModel y `rememberSaveable`.
> - El `AndroidManifest.xml` declara componentes y permisos; el `intent-filter` MAIN/LAUNCHER define la pantalla de arranque.
> - Los recursos viven en `res/` (separados del código, traducibles); los permisos sensibles se piden en runtime y pueden denegarse.

### Para llevar a la práctica
- [ ] Añade un `Log` en `onCreate`, `onResume` y `onPause` y observa el ciclo al rotar
- [ ] Comprueba en logs cómo un estado en variable de la Activity se pierde al rotar
- [ ] Mueve un texto fijo de tu UI a `res/values/strings.xml` y léelo con `stringResource`
- [ ] Declara el permiso de INTERNET en el manifest

### Recursos
- 🌐 developer.android.com/guide/components/activities/activity-lifecycle — ciclo de vida
- 🌐 developer.android.com/guide/topics/manifest/manifest-intro — el manifest
- 🌐 developer.android.com/training/permissions/requesting — permisos en runtime

---
`#android` `#activity` `#ciclo-de-vida` `#manifest` `#permisos`
