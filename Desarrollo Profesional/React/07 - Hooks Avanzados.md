# ⚛️ 07 — Hooks Avanzados

[[Desarrollo Profesional/React/React|⬅️ Volver al índice]] | [[Desarrollo Profesional/React/06 - useEffect y Efectos|← 06]] | [[Desarrollo Profesional/React/08 - Custom Hooks|08 →]]

> [!abstract] Introducción
> Más allá de `useState` y `useEffect`, React trae hooks para casos específicos: guardar valores que no provocan render (`useRef`), evitar recálculos caros (`useMemo`), estabilizar funciones (`useCallback`) y gestionar estado complejo con transiciones claras (`useReducer`). Saber cuándo usar cada uno —y cuándo no— marca la diferencia entre código limpio y código sobre-optimizado.

## ¿De qué vamos a hablar?

Los cuatro hooks que más aparecen tras los básicos: `useRef`, `useMemo`, `useCallback` y `useReducer`, con el criterio de cuándo merece la pena cada uno.

### Conceptos que vamos a cubrir
- `useRef`: valores mutables que no re-renderizan y referencias al DOM
- `useMemo`: memorizar cálculos costosos
- `useCallback`: memorizar funciones
- `useReducer`: estado complejo con un reducer

---

## El Concepto

### useRef

`useRef` guarda un valor mutable en `.current` que **persiste entre renders pero NO provoca re-render** al cambiar. Dos usos: acceder a un nodo del DOM, y guardar datos que no afectan a la UI (un id de timer, el valor previo).

```tsx
import { useRef, useEffect } from "react";

function CampoAutoFoco() {
  const inputRef = useRef<HTMLInputElement>(null);

  useEffect(() => {
    inputRef.current?.focus(); // acceso directo al nodo del DOM
  }, []);

  return <input ref={inputRef} />;
}
```

```tsx
// Guardar un valor entre renders sin re-renderizar
const contadorClicks = useRef(0);
function onClick() {
  contadorClicks.current++; // no dispara render; útil para métricas internas
}
```

### useMemo

`useMemo(fn, deps)` **memoriza el resultado** de un cálculo y solo lo recalcula cuando cambian sus dependencias. Úsalo para cálculos genuinamente caros, no para todo.

```tsx
import { useMemo } from "react";

function ListaFiltrada({ productos, filtro }: { productos: Producto[]; filtro: string }) {
  const visibles = useMemo(
    () => productos.filter((p) => p.nombre.includes(filtro)), // recalcula solo si cambian productos o filtro
    [productos, filtro],
  );
  return <ul>{visibles.map((p) => <li key={p.id}>{p.nombre}</li>)}</ul>;
}
```

### useCallback

`useCallback(fn, deps)` memoriza **la propia función**, devolviendo la misma referencia entre renders mientras no cambien las dependencias. Importa cuando pasas la función a un hijo memorizado con `React.memo` (ver [[Desarrollo Profesional/React/12 - Performance y Optimización|12]]) o como dependencia de otro hook.

```tsx
import { useCallback } from "react";

function Padre({ items }: { items: Item[] }) {
  // sin useCallback, 'onSelect' sería una función nueva en cada render,
  // rompiendo la memoización de un hijo que dependa de ella
  const onSelect = useCallback((id: number) => {
    console.log("seleccionado", id);
  }, []);

  return <ListaMemo items={items} onSelect={onSelect} />;
}
```

> [!tip] No memorices por defecto
> `useMemo` y `useCallback` tienen coste (memoria y comparación de deps). Úsalos cuando haya un problema real de rendimiento medido, no "por si acaso". El [React Compiler](https://react.dev/learn/react-compiler) automatiza gran parte de esto en versiones modernas.

### useReducer

Cuando el estado tiene **varias sub-partes que cambian juntas** o transiciones complejas, `useReducer` centraliza la lógica en una función pura `(estado, acción) => nuevoEstado`, al estilo Redux pero local.

```tsx
import { useReducer } from "react";

type Estado = { cantidad: number; cargando: boolean };
type Accion =
  | { type: "incrementar" }
  | { type: "reset" }
  | { type: "set"; valor: number };

function reducer(estado: Estado, accion: Accion): Estado {
  switch (accion.type) {
    case "incrementar": return { ...estado, cantidad: estado.cantidad + 1 };
    case "reset":       return { ...estado, cantidad: 0 };
    case "set":         return { ...estado, cantidad: accion.valor };
  }
}

function Contador() {
  const [estado, dispatch] = useReducer(reducer, { cantidad: 0, cargando: false });

  return (
    <div>
      <p>{estado.cantidad}</p>
      <button onClick={() => dispatch({ type: "incrementar" })}>+1</button>
      <button onClick={() => dispatch({ type: "reset" })}>Reset</button>
    </div>
  );
}
```

`useReducer` brilla cuando las acciones describen "qué pasó" (`dispatch({ type: "incrementar" })`) en vez de "cómo cambiar el estado": la lógica queda en un único sitio testeable.

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - **`useRef`** guarda un valor mutable en `.current` que persiste sin re-renderizar; sirve para acceder al DOM o guardar datos no visuales.
> - **`useMemo`** memoriza el **resultado** de un cálculo caro; recalcula solo si cambian sus deps.
> - **`useCallback`** memoriza la **función** misma; importa al pasarla a hijos con `React.memo` o como dependencia.
> - **No memorices por defecto**: tiene coste; optimiza con datos, no por intuición.
> - **`useReducer`** centraliza estado complejo en un reducer puro `(estado, acción) => estado`, con acciones que describen qué pasó.

### Para llevar a la práctica
- [ ] Usa `useRef` para enfocar un input al montar y para contar clicks sin re-render
- [ ] Mide con el Profiler un filtrado caro y envuélvelo en `useMemo`
- [ ] Reescribe un `useState` con varias sub-partes como un `useReducer` con acciones tipadas

### Recursos
- 🌐 react.dev/reference/react/useRef — useRef
- 🌐 react.dev/reference/react/useMemo — useMemo y cuándo usarlo
- 🌐 react.dev/learn/extracting-state-logic-into-a-reducer — useReducer paso a paso

---
`#react` `#hooks` `#usememo` `#usereducer`
