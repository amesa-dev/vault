# ⚛️ 00 — Introducción y Setup

[[Desarrollo Profesional/React/React|⬅️ Volver al índice]] | [[Desarrollo Profesional/React/01 - Componentes y JSX|01 →]]

> [!abstract] Introducción
> React es una librería para construir interfaces de usuario a base de **componentes**: piezas reutilizables que describen *qué* debe verse en pantalla en función de unos datos. No es un framework completo (no trae router ni data fetching de serie) — es la capa de vista, y a su alrededor existe un ecosistema. Su idea central es el **renderizado declarativo**: tú describes el estado final de la UI y React se encarga de actualizar el DOM de forma eficiente.

## ¿De qué vamos a hablar?

Qué problema resuelve React, su modelo mental (UI = f(estado)), cómo montar un proyecto moderno con Vite y TypeScript, y la anatomía de la app mínima.

### Conceptos que vamos a cubrir
- El modelo declarativo: la UI como función del estado
- Componentes y el árbol de componentes
- Setup con Vite + TypeScript
- El Virtual DOM y la reconciliación, en una frase

---

## El Concepto

### El modelo declarativo

En JavaScript imperativo tú manipulas el DOM paso a paso: crea un nodo, ponle texto, añádelo, actualízalo cuando cambie algo. En React describes el resultado y dejas que él calcule los cambios.

```tsx
// IMPERATIVO (vanilla JS) — tú das las instrucciones, una a una
const btn = document.createElement("button");
btn.textContent = "Clicks: 0";
let count = 0;
btn.onclick = () => { count++; btn.textContent = `Clicks: ${count}`; };
document.body.append(btn);

// DECLARATIVO (React) — describes el resultado según el estado
function Contador() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(count + 1)}>Clicks: {count}</button>;
}
```

La regla mental que lo resume todo: **`UI = f(estado)`**. La interfaz es el resultado de aplicar tus componentes a los datos actuales. Cuando el estado cambia, React vuelve a llamar a la función y actualiza solo lo que difiere.

### El árbol de componentes

Una app de React es un árbol. Un componente raíz que contiene otros, que a su vez contienen otros. Los datos fluyen **de padres a hijos** mediante props.

```
App
├── Header
│   └── NavBar
├── ListaTareas
│   ├── Tarea
│   ├── Tarea
│   └── Tarea
└── Footer
```

### Setup con Vite + TypeScript

[[Desarrollo Profesional/TypeScript/TypeScript|TypeScript]] es hoy el estándar en React: te da autocompletado y seguridad en props, estado y eventos. **Vite** es la herramienta de build moderna (rápida, con hot reload instantáneo).

```bash
# Crear el proyecto
npm create vite@latest mi-app -- --template react-ts
cd mi-app
npm install
npm run dev        # servidor de desarrollo con HMR en localhost:5173
```

Estructura mínima que genera:

```
mi-app/
├── index.html          ← punto de entrada HTML, contiene <div id="root">
├── src/
│   ├── main.tsx        ← arranca React y lo monta en #root
│   ├── App.tsx         ← componente raíz
│   └── ...
├── tsconfig.json
└── package.json
```

```tsx
// src/main.tsx — el bootstrap de la app
import { StrictMode } from "react";
import { createRoot } from "react-dom/client";
import App from "./App";

createRoot(document.getElementById("root")!).render(
  <StrictMode>
    <App />
  </StrictMode>,
);
```

`StrictMode` no afecta a producción: en desarrollo activa comprobaciones extra (entre ellas, montar-desmontar dos veces para detectar efectos mal escritos — lo veremos en [[Desarrollo Profesional/React/06 - useEffect y Efectos|06]]).

```tsx
// src/App.tsx — tu primer componente
function App() {
  return <h1>Hola, React 👋</h1>;
}

export default App;
```

### Virtual DOM y reconciliación, en una frase

React mantiene una representación ligera de la UI en memoria (el *Virtual DOM*). Cuando el estado cambia, genera un árbol nuevo, lo **compara** con el anterior (reconciliación) y aplica al DOM real **solo las diferencias**. Por eso es rápido sin que tú optimices nada manualmente.

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - React construye UIs a base de **componentes** reutilizables y es **declarativo**: describes el resultado, no los pasos.
> - La regla mental es **`UI = f(estado)`**: cuando el estado cambia, React recalcula y actualiza el DOM por ti.
> - Los datos fluyen **de padres a hijos** por props, formando un árbol de componentes.
> - **Vite + TypeScript** es el setup moderno; `createRoot` monta la app en el `<div id="root">`.
> - El **Virtual DOM** y la reconciliación hacen que las actualizaciones sean eficientes sin esfuerzo manual.

### Para llevar a la práctica
- [ ] Crea un proyecto con `npm create vite@latest -- --template react-ts` y arráncalo
- [ ] Cambia el `<h1>` de `App.tsx` y observa el hot reload
- [ ] Dibuja en papel el árbol de componentes de una web que uses a diario

### Recursos
- 🌐 react.dev/learn — el tutorial oficial, interactivo
- 🌐 vite.dev/guide — guía de Vite
- 🌐 react.dev/learn/thinking-in-react — "pensar en React", lectura imprescindible

---
`#react` `#frontend` `#setup` `#vite`
