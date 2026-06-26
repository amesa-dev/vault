# 📱 08 — Jetpack Compose: UI Declarativa

[[Desarrollo Profesional/Android Kotlin/Android Kotlin|⬅️ Volver al índice]] | [[Desarrollo Profesional/Android Kotlin/07 - Fundamentos de Android|← 07]] | [[Desarrollo Profesional/Android Kotlin/09 - Estado y Recomposición|09 →]]

> [!abstract] Introducción
> Jetpack Compose es el kit de UI moderno de Android: en lugar de definir pantallas en XML y manipularlas con código (el modelo viejo), **describes** cómo debe verse la UI para un estado dado, y Compose se encarga de actualizarla. Si vienes de React, te sonará: es UI declarativa basada en funciones. Esta página cubre los composables, los modifiers y los layouts esenciales.

## ¿De qué vamos a hablar?

Qué es un composable, cómo se componen, el sistema de `Modifier` para tamaño/espaciado/estilo, los layouts fundamentales (`Column`, `Row`, `Box`), las listas eficientes con `LazyColumn` y Material 3.

### Conceptos que vamos a cubrir
- Funciones `@Composable`
- Composables básicos: `Text`, `Button`, `Image`, `TextField`
- El `Modifier` y por qué el orden importa
- Layouts: `Column`, `Row`, `Box`
- Listas eficientes: `LazyColumn`
- Material 3 y `@Preview`

---

## El Concepto

### Una función `@Composable` describe UI

Un composable es una función anotada con `@Composable` que emite UI. No devuelve una vista: la *declara*. Por convención van en PascalCase:

```kotlin
@Composable
fun Saludo(nombre: String) {
    Text(text = "Hola, $nombre")
}
```

Compones la UI **llamando** a unos composables dentro de otros —exactamente la sintaxis de trailing lambda de la página 02:

```kotlin
@Composable
fun Tarjeta() {
    Column {                       // contenedor vertical
        Text("Título")
        Text("Subtítulo")
        Button(onClick = { /* acción */ }) {
            Text("Pulsar")
        }
    }
}
```

### El sistema de `Modifier`

Los `Modifier` configuran tamaño, espaciado, fondo, clics… Se **encadenan**, y **el orden importa** porque cada uno envuelve al anterior:

```kotlin
Text(
    text = "Hola",
    modifier = Modifier
        .padding(16.dp)            // primero margen exterior
        .background(Color.Yellow)  // luego el fondo (no cubre ese padding)
        .padding(8.dp)             // padding interior, dentro del amarillo
)

// El orden cambia el resultado:
Modifier.padding(16.dp).background(Color.Yellow)   // fondo NO incluye el padding
Modifier.background(Color.Yellow).padding(16.dp)   // fondo SÍ incluye el padding
```

Modifiers muy frecuentes: `fillMaxWidth()`, `fillMaxSize()`, `size(48.dp)`, `padding()`, `clickable { }`, `background()`. Las dimensiones se expresan en `.dp` (density-independent pixels) y los tamaños de texto en `.sp`.

### Layouts fundamentales

Tres contenedores cubren casi todo:

```kotlin
Column { /* hijos apilados verticalmente */ }
Row    { /* hijos en fila horizontal */ }
Box    { /* hijos superpuestos (uno encima de otro) */ }
```

La alineación y la distribución se controlan con parámetros:

```kotlin
Column(
    modifier = Modifier.fillMaxSize(),
    verticalArrangement = Arrangement.Center,        // centra en vertical
    horizontalAlignment = Alignment.CenterHorizontally,
) {
    Text("Centrado")
    Spacer(Modifier.height(8.dp))                    // hueco entre elementos
    Button(onClick = {}) { Text("OK") }
}
```

### Listas eficientes: `LazyColumn`

Para listas largas **nunca** uses `Column` con un bucle (renderizaría todos los elementos a la vez). `LazyColumn` solo compone los elementos visibles, como un RecyclerView:

```kotlin
@Composable
fun ListaTareas(tareas: List<Tarea>) {
    LazyColumn {
        items(tareas, key = { it.id }) { tarea ->
            Text(
                text = tarea.titulo,
                modifier = Modifier
                    .fillMaxWidth()
                    .padding(16.dp),
            )
        }
    }
}
```

El parámetro `key` ayuda a Compose a identificar elementos al reordenar o filtrar (igual que las `key` de React).

### Material 3 y `@Preview`

La plantilla trae **Material 3**, el sistema de diseño de Google. Usa sus componentes (`Scaffold`, `TopAppBar`, `Card`, `Button`) para una UI coherente:

```kotlin
@Composable
fun Pantalla() {
    Scaffold(
        topBar = { TopAppBar(title = { Text("Mi App") }) }
    ) { padding ->
        Column(Modifier.padding(padding)) {
            Card { Text("Contenido", Modifier.padding(16.dp)) }
        }
    }
}
```

`@Preview` renderiza un composable en Android Studio **sin emulador** —iteración instantánea:

```kotlin
@Preview(showBackground = true)
@Composable
fun PantallaPreview() {
    MiAppTheme {        // envuelve siempre en tu tema
        Pantalla()
    }
}
```

> [!tip] Composables pequeños y reutilizables
> Igual que en React con los componentes: divide la UI en composables pequeños, cada uno con una responsabilidad. Son más fáciles de previsualizar, testear y reutilizar.

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - Un `@Composable` describe la UI de forma declarativa; compones llamando unos dentro de otros con la sintaxis de trailing lambda.
> - Los `Modifier` se encadenan y **el orden importa**: cada uno envuelve al anterior. Tamaños en `.dp`, texto en `.sp`.
> - `Column` (vertical), `Row` (horizontal) y `Box` (superpuesto) cubren la mayoría de layouts; alineas con `Arrangement`/`Alignment`.
> - Para listas largas usa `LazyColumn`/`LazyRow` (solo componen lo visible) con `key` estable, nunca `Column` + bucle.
> - Material 3 (`Scaffold`, `Card`, `TopAppBar`) da componentes listos; `@Preview` renderiza sin emulador.

### Para llevar a la práctica
- [ ] Monta una pantalla con `Scaffold` + `TopAppBar` y una `Column` centrada
- [ ] Experimenta con el orden de `.padding()` y `.background()` y observa el cambio
- [ ] Renderiza una lista con `LazyColumn` e `items(..., key = { })`
- [ ] Añade un `@Preview` envuelto en tu tema y itera sin emulador

### Recursos
- 🌐 developer.android.com/jetpack/compose/tutorial — tutorial oficial de Compose
- 🌐 developer.android.com/jetpack/compose/modifiers — guía de modifiers
- 🌐 developer.android.com/jetpack/compose/lists — listas con Lazy*
- 🌐 m3.material.io — guía de Material 3

---
`#android` `#compose` `#ui` `#material3`
