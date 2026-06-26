# 📱 Android / Kotlin — Curso Completo

[[Desarrollo Profesional/Inicio Profesional|⬅️ Volver a Desarrollo Profesional]]

> [!abstract] Sobre este curso
> Curso de desarrollo Android moderno con **Kotlin** de principiante a avanzado. Primero dominamos Kotlin como lenguaje (sintaxis, null safety, coroutines), y luego construimos apps reales con **Jetpack Compose** (la UI declarativa actual de Android), arquitectura **MVVM**, persistencia con **Room**, red con **Retrofit**, inyección de dependencias con **Hilt** y testing. Nada de XML legacy ni `AsyncTask`: el Android de hoy. Cada página es autocontenida y sirve como ruta secuencial o referencia rápida.

---

## 📚 Índice del Curso

### Fundamentos de Kotlin
1. [[Desarrollo Profesional/Android Kotlin/00 - Introducción y Setup|00 - Introducción y Setup]] — qué es Kotlin/Android, JVM, Android Studio, Gradle, estructura del proyecto
2. [[Desarrollo Profesional/Android Kotlin/01 - Sintaxis Variables y Tipos|01 - Sintaxis, Variables y Tipos]] — val/var, tipos, `if`/`when` como expresión, strings, rangos
3. [[Desarrollo Profesional/Android Kotlin/02 - Funciones y Lambdas|02 - Funciones y Lambdas]] — args con nombre/default, lambdas, higher-order, extension functions, scope functions
4. [[Desarrollo Profesional/Android Kotlin/03 - Clases y POO|03 - Clases y POO]] — class, data class, sealed, enum, object, interfaces, delegación
5. [[Desarrollo Profesional/Android Kotlin/04 - Colecciones y Programación Funcional|04 - Colecciones y Programación Funcional]] — List/Map/Set, map/filter/fold, sequences

### Kotlin intermedio
6. [[Desarrollo Profesional/Android Kotlin/05 - Null Safety y Manejo de Errores|05 - Null Safety y Manejo de Errores]] — `?.`, `?:`, `!!`, sealed `Result`, excepciones
7. [[Desarrollo Profesional/Android Kotlin/06 - Coroutines y Flow|06 - Coroutines y Flow]] — suspend, scopes, dispatchers, `Flow`, `StateFlow`

### Android con Jetpack Compose
8. [[Desarrollo Profesional/Android Kotlin/07 - Fundamentos de Android|07 - Fundamentos de Android]] — Activity, ciclo de vida, manifest, recursos, permisos
9. [[Desarrollo Profesional/Android Kotlin/08 - Jetpack Compose UI Declarativa|08 - Jetpack Compose: UI Declarativa]] — composables, modifiers, layouts, listas, Material 3
10. [[Desarrollo Profesional/Android Kotlin/09 - Estado y Recomposición|09 - Estado y Recomposición]] — `remember`, `mutableStateOf`, state hoisting, `LaunchedEffect`

### Construir una app real
11. [[Desarrollo Profesional/Android Kotlin/10 - Navegación|10 - Navegación]] — Navigation Compose, rutas, argumentos, back stack
12. [[Desarrollo Profesional/Android Kotlin/11 - Arquitectura MVVM y ViewModel|11 - Arquitectura MVVM y ViewModel]] — ViewModel, UI State, capas, unidirectional data flow
13. [[Desarrollo Profesional/Android Kotlin/12 - Datos Room Retrofit y Repositorios|12 - Datos: Room, Retrofit y Repositorios]] — persistencia, red, patrón repositorio
14. [[Desarrollo Profesional/Android Kotlin/13 - Inyección de Dependencias y Testing|13 - Inyección de Dependencias y Testing]] — Hilt, tests unitarios, tests de UI
15. [[Desarrollo Profesional/Android Kotlin/14 - Trabajo en Segundo Plano y Notificaciones|14 - Trabajo en Segundo Plano y Notificaciones]] — WorkManager, foreground services, notificaciones

---

## 🗺️ Ruta Recomendada

```
Kotlin base   → 00 → 01 → 02 → 03 → 04
Kotlin async  → 05 → 06
Compose       → 07 → 08 → 09
App real      → 10 → 11 → 12 → 13 → 14
```

> [!tip] Aprende construyendo
> A partir de la página 08, monta una app pequeña (una lista de tareas, un lector de noticias) y ve incorporando cada concepto: primero la UI con Compose, luego el estado, la navegación, el ViewModel y por último datos reales con Room y Retrofit. Al llegar a la 13 tendrás una app con arquitectura profesional.

---

## 🔗 Recursos Generales
- 🌐 developer.android.com — documentación oficial (la mejor referencia, con codelabs guiados)
- 🌐 kotlinlang.org/docs — documentación oficial de Kotlin, muy buena y con playground
- 📖 *Kotlin in Action* — Dmitry Jemerov & Svetlana Isakova (la referencia del lenguaje)
- 🌐 *Now in Android* (github.com/android/nowinandroid) — app de muestra oficial con arquitectura moderna
- 🌐 developer.android.com/courses — *Android Basics with Compose*, curso oficial gratuito

---
`#android` `#kotlin` `#compose` `#mobile` `#curso`
