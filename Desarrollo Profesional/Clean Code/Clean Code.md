# 🧹 Clean Code (Código Limpio)

[[Desarrollo Profesional/Inicio Profesional|⬅️ Volver a Desarrollo Profesional]]

> [!abstract] Clean Code
> Concepto popularizado por Robert C. Martin ("Uncle Bob"). Código limpio es aquel que es fácil de leer, fácil de entender y fácil de mantener por cualquier desarrollador (incluido tu yo del futuro).

---

## 🔑 Reglas Fundamentales

### 1. Nombres Significativos
- **Usa nombres que revelen la intención:** Evita variables como `x`, `temp`, `data`.
- **Usa nombres pronunciables y buscables:** `diasDesdeUltimaModificacion` en lugar de `dsum`.
- **Evita la desinformación:** No llames `listaDeUsuarios` a algo que no es un array (usa `usuarios` o `grupoUsuarios`).

### 2. Funciones
- **Pequeñas:** Deben tener pocas líneas de código.
- **Hacer una sola cosa (Single Responsibility):** Si una función hace más de una tarea, divídela.
- **Pocos argumentos:** El número ideal de argumentos para una función es cero (niladica), seguido de uno (monádica) y dos (diádica). Evita tres o más.
- **Sin efectos secundarios:** No debe cambiar el estado global o modificar datos inesperadamente.

### 3. Principio DRY (Don't Repeat Yourself)
Evita la duplicación de código. Si ves que estás copiando y pegando lógica, abstráela en una función o clase reutilizable.

---

## 🚫 Comentarios y Malos Olores (Code Smells)

- **El mejor comentario es el que no se escribe:** En lugar de explicar un código confuso con un comentario, reescribe el código para que sea autoexplicativo.
- **Comentarios redundantes o de diario:** Elimina comentarios que solo explican lo obvio o que guardan un registro de cambios históricos (eso lo hace Git).
- **Código muerto:** Borra funciones o variables que ya no se usan. No las dejes comentadas.

---

## 🧩 Principios SOLID (Resumen)

| Principio | Siglas | Concepto clave |
| --- | --- | --- |
| **Single Responsibility** | SRP | Una clase debe tener una sola razón para cambiar. |
| **Open/Closed** | OCP | Abierto para la extensión, cerrado para la modificación. |
| **Liskov Substitution** | LSP | Las subclases deben ser sustituibles por sus clases base. |
| **Interface Segregation** | ISP | Es mejor tener interfaces específicas que una de propósito general. |
| **Dependency Inversion** | DIP | Depende de abstracciones, no de clases concretas. |

---
`#cleancode` `#buenaspracticas` `#diseno` `#apuntes`
