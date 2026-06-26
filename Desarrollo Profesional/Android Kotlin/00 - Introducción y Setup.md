# 📱 00 — Introducción y Setup

[[Desarrollo Profesional/Android Kotlin/Android Kotlin|⬅️ Volver al índice]] | [[Desarrollo Profesional/Android Kotlin/01 - Sintaxis Variables y Tipos|01 →]]

> [!abstract] Introducción
> Kotlin es un lenguaje moderno, conciso y con seguridad frente a `null`, que corre sobre la JVM y es desde 2019 el **lenguaje oficial de Android**. Esta primera página sitúa el terreno: qué es Kotlin, cómo se relaciona con Java y Android, qué es Gradle, y cómo montar Android Studio para que el resto del curso te resulte cómodo.

## ¿De qué vamos a hablar?

Antes de escribir código, conviene entender qué ejecuta tu app, cómo se compila y qué piezas componen un proyecto Android. Si vienes de Python o JavaScript, el modelo de compilación a bytecode de la JVM y el sistema de build con Gradle son lo más novedoso.

### Conceptos que vamos a cubrir
- Qué es Kotlin y por qué Android lo adoptó
- La JVM, el bytecode y la relación con Java
- Android Studio y el emulador
- Gradle: el sistema de build (Kotlin DSL)
- Anatomía de un proyecto Android

---

## El Concepto

### Kotlin = un lenguaje JVM moderno

Kotlin lo creó JetBrains (los de IntelliJ) para corregir las asperezas de Java manteniendo total interoperabilidad. Compila a **bytecode de la JVM**, igual que Java, por lo que ambos lenguajes conviven en el mismo proyecto y se llaman entre sí sin fricción.

```
código.kt → kotlinc (compilador) → bytecode (.class / .dex) → ejecuta la JVM / Android Runtime (ART)
```

Sus tres ventajas frente a Java, que verás una y otra vez en el curso:

- **Conciso**: una `data class` reemplaza decenas de líneas de boilerplate de Java.
- **Null-safe**: el sistema de tipos distingue `String` de `String?`, eliminando la mayoría de `NullPointerException` en compilación.
- **Funcional**: lambdas, funciones de extensión y una librería de colecciones muy expresiva.

> [!info] En Android, ¿Java o Kotlin?
> Toda la documentación, librerías nuevas (Jetpack Compose, Coroutines) y ejemplos oficiales son Kotlin-first. Aprende Kotlin; leerás Java solo en proyectos legacy.

### Hola Kotlin

```kotlin
fun main() {
    println("Hola, Kotlin")
}
```

Sin clase envolvente, sin `public static void`, sin punto y coma obligatorio. Puedes probar fragmentos como este en el playground de kotlinlang.org sin instalar nada.

### Android Studio y el emulador

Android Studio es el IDE oficial (basado en IntelliJ). Incluye el editor, el **emulador** (un teléfono virtual) y el **SDK** de Android. El flujo mínimo:

1. Descarga Android Studio de developer.android.com/studio.
2. `New Project` → plantilla **Empty Activity** (trae Compose ya configurado).
3. Crea un dispositivo virtual en el **Device Manager** (p. ej. un Pixel con la última API).
4. Pulsa ▶️ **Run** y la app se instala en el emulador.

### Gradle: el sistema de build

**Gradle** compila, gestiona dependencias y empaqueta tu app en un `.apk`/`.aab`. La configuración vive en ficheros `build.gradle.kts` (Kotlin DSL). Lo esencial:

```kotlin
// app/build.gradle.kts
android {
    namespace = "com.ejemplo.miapp"
    compileSdk = 35

    defaultConfig {
        applicationId = "com.ejemplo.miapp"
        minSdk = 24          // versión mínima de Android soportada
        targetSdk = 35       // versión con la que se prueba/optimiza
        versionCode = 1
        versionName = "1.0"
    }
}

dependencies {
    implementation("androidx.core:core-ktx:1.13.1")
    implementation(platform("androidx.compose:compose-bom:2024.09.00"))
    implementation("androidx.compose.material3:material3")
}
```

Las versiones suelen centralizarse en un **version catalog** (`gradle/libs.versions.toml`) para no repetirlas. No memorices nada de esto: Android Studio lo genera por ti. Lo importante es saber **dónde** se declaran SDK y dependencias.

### Anatomía de un proyecto Android

```
MiApp/
├── app/
│   ├── build.gradle.kts          # config del módulo de la app
│   └── src/main/
│       ├── AndroidManifest.xml   # declara la app, permisos, componentes
│       ├── java/com/ejemplo/miapp/
│       │   └── MainActivity.kt   # punto de entrada de la UI
│       └── res/                  # recursos: strings, colores, drawables
│           ├── values/
│           └── drawable/
├── build.gradle.kts              # config del proyecto raíz
├── settings.gradle.kts           # módulos y repositorios
└── gradle/libs.versions.toml     # version catalog
```

| Pieza | Qué es |
|-------|--------|
| `AndroidManifest.xml` | Identidad de la app: nombre, icono, permisos, componentes |
| `MainActivity.kt` | La `Activity` que arranca; aquí montaremos la UI de Compose |
| `res/` | Recursos no-código: textos traducibles, colores, imágenes |
| Gradle | Compila, resuelve dependencias y genera el `.apk`/`.aab` |

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - Kotlin compila a bytecode de la JVM y es 100% interoperable con Java; es el lenguaje oficial de Android.
> - Sus señas de identidad: conciso, null-safe y funcional. Por eso reduce el boilerplate de Java.
> - Android Studio es el IDE oficial e incluye SDK y emulador. La plantilla *Empty Activity* trae Compose listo.
> - Gradle (con `build.gradle.kts`) gestiona el build y las dependencias; `minSdk`/`targetSdk`/`compileSdk` controlan las versiones de Android.

### Para llevar a la práctica
- [ ] Instala Android Studio y crea un proyecto *Empty Activity*
- [ ] Configura un emulador en el Device Manager y ejecuta la app por defecto
- [ ] Abre `app/build.gradle.kts` e identifica `minSdk`, `targetSdk` y las dependencias de Compose
- [ ] Prueba `fun main() { println("Hola") }` en el playground de kotlinlang.org

### Recursos
- 🌐 developer.android.com/studio — descarga y guía de instalación
- 🌐 kotlinlang.org/docs/getting-started.html — primeros pasos con Kotlin
- 🌐 developer.android.com/courses/android-basics-compose — curso oficial gratuito
- 📖 *Kotlin in Action* — Jemerov & Isakova (capítulo 1 para el contexto)

---
`#android` `#kotlin` `#setup` `#gradle` `#android-studio`
