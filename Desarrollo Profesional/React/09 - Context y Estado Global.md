# ⚛️ 09 — Context y Estado Global

[[Desarrollo Profesional/React/React|⬅️ Volver al índice]] | [[Desarrollo Profesional/React/08 - Custom Hooks|← 08]] | [[Desarrollo Profesional/React/10 - Data Fetching|10 →]]

> [!abstract] Introducción
> Pasar props a través de muchos niveles intermedios que no las usan es el problema del **prop drilling**. La **Context API** de React permite que un componente padre exponga un valor a todo su subárbol sin pasarlo manualmente nivel a nivel. Es ideal para datos "globales" de baja frecuencia de cambio (tema, usuario autenticado, idioma). Para estado global complejo o que cambia mucho, conviene una librería como Zustand.

## ¿De qué vamos a hablar?

El problema del prop drilling, cómo crear y consumir un Context, el patrón de envolverlo en un custom hook, sus límites de rendimiento, y cuándo dar el salto a Zustand.

### Conceptos que vamos a cubrir
- Prop drilling: el problema que Context resuelve
- `createContext`, `Provider` y `useContext`
- El patrón Context + custom hook
- Límites de Context y alternativa con Zustand

---

## El Concepto

### El problema: prop drilling

```
App (tiene 'usuario')
└── Layout            ← no usa usuario, pero lo recibe y lo pasa
    └── Header        ← no usa usuario, pero lo recibe y lo pasa
        └── Avatar    ← AQUÍ se usa usuario
```

Pasar `usuario` por `Layout` y `Header` solo para que llegue a `Avatar` es ruido. Context lo evita.

### createContext + Provider + useContext

Tres piezas: creas el contexto, un `Provider` envuelve el subárbol y aporta el valor, y los hijos lo leen con `useContext`.

```tsx
import { createContext, useContext, useState } from "react";

type Tema = "claro" | "oscuro";
type TemaContexto = { tema: Tema; alternar: () => void };

// 1. Crear el contexto (con un valor inicial null y tipo)
const TemaContext = createContext<TemaContexto | null>(null);

// 2. Un Provider que aporta el valor a su subárbol
function TemaProvider({ children }: { children: React.ReactNode }) {
  const [tema, setTema] = useState<Tema>("claro");
  const alternar = () => setTema((t) => (t === "claro" ? "oscuro" : "claro"));
  return (
    <TemaContext.Provider value={{ tema, alternar }}>
      {children}
    </TemaContext.Provider>
  );
}

// 3. Un custom hook para consumirlo con seguridad
function useTema() {
  const ctx = useContext(TemaContext);
  if (!ctx) throw new Error("useTema debe usarse dentro de <TemaProvider>");
  return ctx;
}
```

Uso: envuelve la app una vez y consume donde haga falta, sin prop drilling.

```tsx
function App() {
  return (
    <TemaProvider>
      <Pagina />
    </TemaProvider>
  );
}

function BotonTema() {
  const { tema, alternar } = useTema(); // lo lee directo, sin recibir props
  return <button onClick={alternar}>Tema: {tema}</button>;
}
```

> [!tip] Envuelve siempre el Context en un custom hook
> El patrón `useTema()` con el `throw` da dos ventajas: un error claro si lo usas fuera del Provider, y un único punto de importación. Es el estándar de facto.

### Los límites de Context

Context tiene una característica clave de rendimiento: **cuando el valor del Provider cambia, todos los componentes que lo consumen se re-renderizan**. Para datos que cambian poco (tema, idioma, usuario) es perfecto. Para estado que cambia muchas veces por segundo o muy grande, puede causar re-renders innecesarios.

```tsx
// Cuidado: un único Context con MUCHAS cosas que cambian a ritmos distintos
// hace que cambiar una re-renderice consumidores de las otras.
// Solución: divide en varios Context por dominio, o usa un store externo.
```

### Cuándo saltar a Zustand

Para estado global complejo, **Zustand** es la opción ligera y popular: un store fuera del árbol de React, sin Provider, con selectores que evitan re-renders innecesarios.

```tsx
import { create } from "zustand";

type CarritoState = {
  items: string[];
  anadir: (item: string) => void;
};

const useCarrito = create<CarritoState>((set) => ({
  items: [],
  anadir: (item) => set((s) => ({ items: [...s.items, item] })),
}));

// En cualquier componente, sin Provider y con selector (solo re-renderiza si 'items' cambia)
function Carrito() {
  const items = useCarrito((s) => s.items);
  return <span>{items.length} artículos</span>;
}
```

Regla práctica: **Context** para configuración global de bajo cambio; **Zustand** (u otro store) para estado de aplicación rico y frecuente; **TanStack Query** para estado del servidor (ver [[Desarrollo Profesional/React/10 - Data Fetching|10]]).

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - **Context** resuelve el **prop drilling**: expone un valor a todo un subárbol sin pasarlo nivel a nivel.
> - Tres piezas: **`createContext`**, un **`Provider`** que aporta el valor, y **`useContext`** para leerlo.
> - Envuelve el consumo en un **custom hook** con `throw` si falta el Provider: error claro y un punto único.
> - Al cambiar el valor del Provider, **todos los consumidores re-renderizan**: ideal para datos de bajo cambio.
> - Para estado global rico/frecuente usa **Zustand**; para estado del servidor, **TanStack Query**.

### Para llevar a la práctica
- [ ] Implementa un Context de tema (claro/oscuro) con su Provider y su hook `useTema`
- [ ] Refactoriza un caso de prop drilling de 3+ niveles para usar Context
- [ ] Monta un store de Zustand para un carrito y usa selectores para evitar re-renders

### Recursos
- 🌐 react.dev/learn/passing-data-deeply-with-context — Context oficial
- 🌐 react.dev/learn/scaling-up-with-reducer-and-context — combinar reducer + context
- 🌐 zustand.docs.pmnd.rs — documentación de Zustand

---
`#react` `#context` `#estado-global` `#zustand`
