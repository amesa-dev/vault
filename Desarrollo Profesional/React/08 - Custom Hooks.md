# ⚛️ 08 — Custom Hooks

[[Desarrollo Profesional/React/React|⬅️ Volver al índice]] | [[Desarrollo Profesional/React/07 - Hooks Avanzados|← 07]] | [[Desarrollo Profesional/React/09 - Context y Estado Global|09 →]]

> [!abstract] Introducción
> Un **custom hook** es una función que empieza por `use` y que llama a otros hooks. Es el mecanismo de React para **extraer y reutilizar lógica con estado** entre componentes: la suscripción a un evento, la lectura de `localStorage`, una llamada a una API. No comparte estado entre componentes —cada uso es independiente—, comparte la *lógica*. Es el corazón de la reutilización en React moderno.

## ¿De qué vamos a hablar?

Qué es un custom hook y por qué existe, las reglas de los hooks que debes respetar, y varios ejemplos prácticos que verás (o escribirás) constantemente.

### Conceptos que vamos a cubrir
- Las reglas de los hooks
- Extraer lógica repetida a un custom hook
- Ejemplos: `useToggle`, `useLocalStorage`, `useDebounce`
- Qué comparten (lógica) y qué no (estado)

---

## El Concepto

### Las reglas de los hooks

Antes de crear hooks, dos reglas innegociables que el linter vigila:

1. **Solo se llaman en el nivel superior.** Nunca dentro de `if`, bucles o funciones anidadas. React identifica cada hook por su orden de llamada, que debe ser estable entre renders.
2. **Solo desde componentes o desde otros hooks.** No desde funciones normales.

```tsx
// MAL — hook dentro de una condición: el orden cambia entre renders
function Componente({ activo }: { activo: boolean }) {
  if (activo) {
    const [x, setX] = useState(0); // ❌
  }
}

// BIEN — siempre en el nivel superior; condiciona dentro
function Componente({ activo }: { activo: boolean }) {
  const [x, setX] = useState(0);
  if (!activo) return null;
}
```

### Extraer lógica a un custom hook

Si dos componentes repiten la misma lógica con estado, extráela. La regla: el nombre **empieza por `use`** (así el linter aplica las reglas de los hooks).

```tsx
// useToggle: encapsula un booleano y su alternancia
import { useState, useCallback } from "react";

function useToggle(inicial = false) {
  const [valor, setValor] = useState(inicial);
  const alternar = useCallback(() => setValor((v) => !v), []);
  return [valor, alternar] as const; // 'as const' → tupla tipada [boolean, () => void]
}

// Uso — cada componente tiene SU propio estado, comparten la lógica
function Panel() {
  const [abierto, alternarAbierto] = useToggle();
  return (
    <div>
      <button onClick={alternarAbierto}>{abierto ? "Cerrar" : "Abrir"}</button>
      {abierto && <p>Contenido</p>}
    </div>
  );
}
```

### Ejemplos que usarás siempre

**`useLocalStorage`** — estado sincronizado con `localStorage`:

```tsx
import { useState } from "react";

function useLocalStorage<T>(clave: string, inicial: T) {
  const [valor, setValor] = useState<T>(() => {
    const guardado = localStorage.getItem(clave);
    return guardado ? (JSON.parse(guardado) as T) : inicial;
  });

  function actualizar(nuevo: T) {
    setValor(nuevo);
    localStorage.setItem(clave, JSON.stringify(nuevo));
  }

  return [valor, actualizar] as const;
}
```

**`useDebounce`** — retrasa un valor (típico en buscadores, para no disparar en cada tecla):

```tsx
import { useState, useEffect } from "react";

function useDebounce<T>(valor: T, ms = 300): T {
  const [debounced, setDebounced] = useState(valor);

  useEffect(() => {
    const id = setTimeout(() => setDebounced(valor), ms);
    return () => clearTimeout(id); // cancela si 'valor' cambia antes de cumplirse el plazo
  }, [valor, ms]);

  return debounced;
}

// Uso típico en un buscador
function Buscador() {
  const [texto, setTexto] = useState("");
  const textoDebounced = useDebounce(texto, 400);
  // ... aquí lanzarías la búsqueda solo cuando textoDebounced cambie
  return <input value={texto} onChange={(e) => setTexto(e.target.value)} />;
}
```

> [!tip] Lógica fuera, JSX dentro
> Un buen custom hook deja el componente legible: el componente describe *qué* se ve, el hook encapsula *cómo* funciona la lógica. Si un componente acumula muchos `useState`/`useEffect` entrelazados, suele haber uno o dos custom hooks esperando a nacer.

### Comparten lógica, no estado

Importante: dos componentes que usan `useToggle` tienen **estados independientes**. Un custom hook reutiliza el *código*, no los *datos*. Para compartir datos entre componentes necesitas Context o un store global (ver [[Desarrollo Profesional/React/09 - Context y Estado Global|09]]).

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - Un **custom hook** es una función `useAlgo` que llama a otros hooks para **reutilizar lógica con estado**.
> - **Reglas de los hooks**: solo en el nivel superior (nunca en `if`/bucles) y solo desde componentes u otros hooks.
> - El prefijo **`use`** no es decorativo: activa las reglas del linter.
> - Patrones habituales: `useToggle`, `useLocalStorage`, `useDebounce`, `useFetch`…
> - Comparten **lógica, no estado**: cada componente que usa el hook tiene su propia copia del estado.

### Para llevar a la práctica
- [ ] Extrae un `useToggle` y úsalo en dos componentes distintos; confirma que sus estados son independientes
- [ ] Escribe un `useLocalStorage` y persiste el tema (claro/oscuro) de tu app
- [ ] Implementa `useDebounce` en un buscador para no disparar la búsqueda en cada tecla

### Recursos
- 🌐 react.dev/learn/reusing-logic-with-custom-hooks — custom hooks oficial
- 🌐 react.dev/reference/rules/rules-of-hooks — las reglas de los hooks
- 🌐 usehooks.com — colección de custom hooks listos para copiar y estudiar

---
`#react` `#custom-hooks` `#reutilizacion` `#hooks`
