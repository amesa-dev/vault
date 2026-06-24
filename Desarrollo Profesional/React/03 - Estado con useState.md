# ⚛️ 03 — Estado con useState

[[Desarrollo Profesional/React/React|⬅️ Volver al índice]] | [[Desarrollo Profesional/React/02 - Props y Composición|← 02]] | [[Desarrollo Profesional/React/04 - Renderizado Condicional y Listas|04 →]]

> [!abstract] Introducción
> El **estado** es la memoria de un componente: datos que cambian con el tiempo y que, al cambiar, provocan un nuevo renderizado. Mientras las props vienen de fuera y son inmutables, el estado es interno y mutable a través de un *setter*. El hook `useState` es la puerta de entrada a la interactividad en React.

## ¿De qué vamos a hablar?

Qué es el estado y en qué se diferencia de una variable normal, cómo declararlo con `useState`, por qué hay que actualizarlo de forma inmutable, las actualizaciones basadas en el valor anterior, y el batching.

### Conceptos que vamos a cubrir
- `useState`: declarar estado y su setter
- Por qué una variable normal no funciona
- Inmutabilidad al actualizar objetos y arrays
- Actualización funcional `setX(prev => ...)`
- Batching: varios setState, un solo render

---

## El Concepto

### Declarar estado con useState

`useState(valorInicial)` devuelve una pareja: el valor actual y una función para actualizarlo. La convención es `[algo, setAlgo]`.

```tsx
import { useState } from "react";

function Contador() {
  const [count, setCount] = useState(0); // estado inicial: 0

  return (
    <button onClick={() => setCount(count + 1)}>
      Has pulsado {count} veces
    </button>
  );
}
```

Con TypeScript, el tipo se infiere del valor inicial (`number` aquí). Si el inicial no basta (p. ej. arranca en `null`), tipa explícitamente: `useState<Usuario | null>(null)`.

### ¿Por qué no una variable normal?

Una variable local se reinicia en cada render y, sobre todo, **no le dice a React que vuelva a renderizar**. El estado persiste entre renders y dispara la actualización.

```tsx
// MAL — una variable normal: cambia, pero la UI no se entera
function Roto() {
  let count = 0;
  return <button onClick={() => { count++; console.log(count); }}>{count}</button>;
  // count sube en consola, pero en pantalla siempre pone 0
}

// BIEN — estado: cambiar dispara un re-render y la UI se actualiza
function Funciona() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(count + 1)}>{count}</button>;
}
```

### Inmutabilidad: nunca mutes el estado

React detecta cambios comparando **referencias**. Si mutas un objeto o array existente, la referencia no cambia y React puede no re-renderizar. Crea siempre un valor **nuevo**.

```tsx
const [usuario, setUsuario] = useState({ nombre: "Ana", edad: 30 });

// MAL — mutación directa: misma referencia, React no se entera
usuario.edad = 31;
setUsuario(usuario);

// BIEN — objeto nuevo con spread
setUsuario({ ...usuario, edad: 31 });

const [tareas, setTareas] = useState<string[]>([]);

// MAL — push muta el array existente
tareas.push("nueva");
setTareas(tareas);

// BIEN — array nuevo
setTareas([...tareas, "nueva"]);
setTareas(tareas.filter((t) => t !== "vieja")); // eliminar
setTareas(tareas.map((t) => (t === "a" ? "A" : t))); // modificar
```

### Actualización funcional

Cuando el nuevo estado depende del anterior, pasa una **función** al setter en vez de un valor. React te garantiza el valor más reciente, evitando bugs con actualizaciones encadenadas o asíncronas.

```tsx
// MAL — con clausuras "viejas", esto sube solo 1, no 3
function tripleIncremento() {
  setCount(count + 1);
  setCount(count + 1);
  setCount(count + 1);
}

// BIEN — cada actualización parte del valor previo real → sube 3
function tripleIncremento() {
  setCount((prev) => prev + 1);
  setCount((prev) => prev + 1);
  setCount((prev) => prev + 1);
}
```

### Batching: varios setState, un render

React **agrupa** (batches) las actualizaciones de estado que ocurren en el mismo evento y hace **un solo re-render** al final, por eficiencia. Por eso `count` no cambia "a mitad" de la función: el nuevo valor estará disponible en el **siguiente** render, no en la línea siguiente.

```tsx
function manejarClick() {
  setCount(count + 1);
  console.log(count); // ⚠️ imprime el valor VIEJO; el nuevo llega al re-render
}
```

> [!tip] El estado es una foto, no una variable viva
> Durante un render, los valores de estado son una "foto" fija. Pedir un cambio (`setCount`) no altera la foto actual: programa un render nuevo con una foto nueva. Interiorizar esto evita la mayoría de los bugs de principiante.

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - El **estado** es la memoria del componente; al cambiar, provoca un **re-render**. `useState` devuelve `[valor, setter]`.
> - Una **variable normal no sirve**: no persiste entre renders ni avisa a React.
> - **Nunca mutes** el estado: crea objetos/arrays nuevos (spread, `map`, `filter`). React compara por referencia.
> - Usa la **forma funcional** `setX(prev => ...)` cuando el nuevo valor depende del anterior.
> - React hace **batching**: el estado actualizado se ve en el siguiente render, no en la línea siguiente.

### Para llevar a la práctica
- [ ] Crea un contador y rompe a propósito el triple incremento; arréglalo con la forma funcional
- [ ] Monta una lista de tareas con añadir y eliminar usando solo operaciones inmutables
- [ ] Intenta mutar un objeto de estado directamente y observa que la UI no se actualiza

### Recursos
- 🌐 react.dev/learn/state-a-components-memory — el estado como memoria
- 🌐 react.dev/learn/state-as-a-snapshot — el estado como "foto"
- 🌐 react.dev/learn/updating-objects-in-state — actualizar objetos y arrays sin mutar

---
`#react` `#estado` `#usestate` `#hooks`
