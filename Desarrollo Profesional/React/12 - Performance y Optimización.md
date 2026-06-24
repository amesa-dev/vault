# ⚛️ 12 — Performance y Optimización

[[Desarrollo Profesional/React/React|⬅️ Volver al índice]] | [[Desarrollo Profesional/React/11 - React Router|← 11]] | [[Desarrollo Profesional/React/13 - Testing|13 →]]

> [!abstract] Introducción
> React es rápido por defecto, pero apps grandes pueden sufrir **re-renders innecesarios** o **bundles pesados**. Optimizar bien empieza por **medir** (no adivinar) y entender por qué un componente se vuelve a renderizar. Luego se aplican herramientas concretas: `React.memo`, memoización, `key`, y la división del código con `lazy`. La regla rectora: primero corrección y legibilidad, después rendimiento donde haya un problema real.

## ¿De qué vamos a hablar?

Por qué se re-renderiza un componente, cómo evitar re-renders con `React.memo` y memoización, el code splitting con `lazy`/`Suspense`, y cómo perfilar.

### Conceptos que vamos a cubrir
- Qué dispara un re-render
- `React.memo` + `useMemo`/`useCallback`
- Code splitting con `lazy` y `Suspense`
- Medir con React DevTools Profiler

---

## El Concepto

### Qué dispara un re-render

Un componente se re-renderiza cuando: cambia su **estado**, cambia su **contexto** consumido, o **su padre se re-renderiza**. Lo último sorprende: por defecto, si un padre renderiza, **todos sus hijos** renderizan, aunque sus props no cambien. Normalmente es barato; en árboles grandes o cálculos pesados, importa.

```tsx
// Cada vez que Padre renderiza (p.ej. al teclear), Hijo también lo hace,
// aunque 'datos' no haya cambiado.
function Padre() {
  const [texto, setTexto] = useState("");
  return (
    <>
      <input value={texto} onChange={(e) => setTexto(e.target.value)} />
      <Hijo datos={datos} /> {/* renderiza en cada tecla */}
    </>
  );
}
```

### React.memo + memoización

`React.memo` evita que un componente re-renderice si sus props **no han cambiado** (comparación superficial). Para que funcione con props de tipo objeto o función, esas props deben estar **memorizadas** (`useMemo`/`useCallback`, ver [[Desarrollo Profesional/React/07 - Hooks Avanzados|07]]).

```tsx
import { memo, useCallback } from "react";

const Hijo = memo(function Hijo({ datos, onSelect }: Props) {
  console.log("render Hijo");
  return <ul>{datos.map((d) => <li key={d.id} onClick={() => onSelect(d.id)}>{d.nombre}</li>)}</ul>;
});

function Padre({ datos }: { datos: Item[] }) {
  const [texto, setTexto] = useState("");
  // sin useCallback, onSelect sería nueva en cada render y romperia el memo
  const onSelect = useCallback((id: number) => console.log(id), []);
  return (
    <>
      <input value={texto} onChange={(e) => setTexto(e.target.value)} />
      <Hijo datos={datos} onSelect={onSelect} /> {/* ya NO renderiza al teclear */}
    </>
  );
}
```

> [!tip] El React Compiler está cambiando esto
> En React moderno, el **React Compiler** memoriza automáticamente componentes y valores, haciendo innecesarios muchos `memo`/`useMemo`/`useCallback` manuales. Aun así, conviene entender el modelo: explica por qué algo se re-renderiza y qué hace el compilador por ti.

### Code splitting con lazy y Suspense

No cargues toda la app de golpe. `lazy` carga un componente **solo cuando se necesita** (p. ej. una ruta), y `Suspense` muestra un fallback mientras llega. Reduce el tamaño del bundle inicial.

```tsx
import { lazy, Suspense } from "react";

const Dashboard = lazy(() => import("./Dashboard")); // se descarga al usarse

function App() {
  return (
    <Suspense fallback={<p>Cargando…</p>}>
      <Dashboard />
    </Suspense>
  );
}
```

Lo habitual es aplicar `lazy` por ruta, de modo que cada página se descargue al navegar a ella.

### Medir antes de optimizar

No optimices a ciegas. El **Profiler** de React DevTools graba los renders y te dice qué componente renderizó, cuántas veces y por qué. Optimiza solo lo que el Profiler señale como caro.

```
Flujo correcto:
  1. ¿Hay un problema real percibido? (jank, lentitud)
  2. Mídelo con el Profiler / Lighthouse
  3. Identifica el componente o cálculo culpable
  4. Aplica memo / useMemo / lazy SOLO ahí
  5. Vuelve a medir para confirmar la mejora
```

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - Un componente re-renderiza por **estado**, **contexto** o porque **su padre lo hizo** (los hijos renderizan por defecto).
> - **`React.memo`** evita re-renders si las props no cambian; requiere **memorizar** props de tipo objeto/función con `useMemo`/`useCallback`.
> - El **React Compiler** automatiza gran parte de la memoización en versiones modernas.
> - **`lazy` + `Suspense`** dividen el código y reducen el bundle inicial; aplícalo por ruta.
> - **Mide con el Profiler antes de optimizar**: primero corrección y claridad, después rendimiento donde duela.

### Para llevar a la práctica
- [ ] Usa el Profiler para ver un hijo que re-renderiza al teclear en el padre
- [ ] Aplica `React.memo` + `useCallback` y confirma que deja de renderizar
- [ ] Convierte las rutas de una app a `lazy` + `Suspense` y compara el tamaño del bundle

### Recursos
- 🌐 react.dev/reference/react/memo — React.memo
- 🌐 react.dev/learn/react-compiler — el compilador y la memoización automática
- 🌐 react.dev/reference/react/lazy — code splitting con lazy y Suspense

---
`#react` `#performance` `#optimizacion` `#code-splitting`
