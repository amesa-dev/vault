# ⚛️ 01 — Componentes y JSX

[[Desarrollo Profesional/React/React|⬅️ Volver al índice]] | [[Desarrollo Profesional/React/00 - Introducción y Setup|← 00]] | [[Desarrollo Profesional/React/02 - Props y Composición|02 →]]

> [!abstract] Introducción
> Un componente es una **función de JavaScript que devuelve JSX**. Es la unidad básica de React: encapsula marcado, estilo y lógica en una pieza con nombre, reutilizable y testeable. JSX es la sintaxis que parece HTML dentro de JavaScript — pero no es HTML, es azúcar sintáctico que se compila a llamadas de función.

## ¿De qué vamos a hablar?

Cómo se define un componente de función, qué es realmente JSX y sus reglas, cómo incrustar expresiones, y cómo agrupar elementos sin ensuciar el DOM.

### Conceptos que vamos a cubrir
- Componentes de función y la convención de PascalCase
- JSX: qué es y a qué compila
- Reglas de JSX: un solo nodo raíz, atributos camelCase, cierre obligatorio
- Expresiones con `{}` y renderizado de Fragments

---

## El Concepto

### Un componente es una función

Por convención, los componentes empiezan por **mayúscula** (PascalCase). React usa esa mayúscula para distinguir tus componentes (`<Avatar />`) de las etiquetas HTML nativas (`<div />`).

```tsx
function Saludo() {
  return <p>Hola desde un componente</p>;
}

// Se usa como si fuera una etiqueta HTML
function App() {
  return (
    <main>
      <Saludo />
      <Saludo />
    </main>
  );
}
```

### Qué es JSX realmente

JSX **no es HTML ni una string**. Es sintaxis que el compilador (Babel/SWC, vía Vite) transforma en llamadas a `React.createElement`, que devuelven objetos describiendo la UI.

```tsx
// Lo que escribes:
const elemento = <h1 className="titulo">Hola</h1>;

// A lo que compila (aproximadamente):
const elemento = React.createElement("h1", { className: "titulo" }, "Hola");
```

Por eso JSX puede ir donde va una expresión: asignarse a variables, devolverse de funciones, meterse en arrays.

### Las reglas de JSX

**1. Un único nodo raíz.** Una función solo puede devolver un elemento. Si necesitas varios hermanos, envuélvelos.

```tsx
// MAL — dos nodos raíz
function Perfil() {
  return (
    <h1>Ana</h1>
    <p>Ingeniera</p>   // ❌ error de compilación
  );
}

// BIEN — envueltos en un Fragment (no añade nodo al DOM)
function Perfil() {
  return (
    <>
      <h1>Ana</h1>
      <p>Ingeniera</p>
    </>
  );
}
```

**2. Atributos en camelCase.** Como JSX es JavaScript, los atributos siguen su convención: `class` → `className`, `for` → `htmlFor`, `onclick` → `onClick`, `tabindex` → `tabIndex`.

**3. Todas las etiquetas se cierran.** Incluso las vacías: `<img />`, `<br />`, `<input />`.

### Expresiones con `{}`

Las llaves abren una ventana de JavaScript dentro de JSX. Dentro va **cualquier expresión** (algo que produce un valor): variables, llamadas a función, operadores ternarios, `.map()`… pero no sentencias (`if`, `for`).

```tsx
function Tarjeta() {
  const nombre = "Marco";
  const ahora = new Date().toLocaleTimeString();
  const precio = 29.99;

  return (
    <div className="tarjeta">
      <h2>Hola, {nombre}</h2>
      <p>Son las {ahora}</p>
      <p>Total: {precio.toFixed(2)} €</p>
      <p>Acceso: {precio > 0 ? "de pago" : "gratis"}</p>
    </div>
  );
}
```

Las llaves también sirven para pasar valores no-string a atributos:

```tsx
<img src={urlAvatar} alt={`Foto de ${nombre}`} width={64} />
<button disabled={cargando}>Enviar</button>
```

### Fragments

Cuando solo quieres agrupar sin añadir un `<div>` extra al DOM, usa el Fragment `<>...</>`. Si necesitas pasarle una `key` (en listas), usa la forma larga `<React.Fragment key={id}>`.

```tsx
import { Fragment } from "react";

function FilasDefinicion({ items }: { items: { termino: string; def: string }[] }) {
  return (
    <dl>
      {items.map((item) => (
        <Fragment key={item.termino}>
          <dt>{item.termino}</dt>
          <dd>{item.def}</dd>
        </Fragment>
      ))}
    </dl>
  );
}
```

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - Un componente es una **función en PascalCase que devuelve JSX**; React usa la mayúscula para distinguirlo del HTML nativo.
> - **JSX no es HTML**: compila a `createElement` y puede usarse como cualquier expresión de JavaScript.
> - Reglas: **un solo nodo raíz**, atributos en **camelCase** (`className`, `onClick`), y **todas las etiquetas cierran**.
> - Las **llaves `{}`** incrustan expresiones (no sentencias) en el marcado.
> - Los **Fragments** (`<>...</>`) agrupan elementos sin añadir nodos al DOM.

### Para llevar a la práctica
- [ ] Crea un componente `Tarjeta` que muestre nombre, rol y un dato calculado con `{}`
- [ ] Provoca a propósito el error de "dos nodos raíz" y arréglalo con un Fragment
- [ ] Convierte un trozo de HTML real a JSX (renombra `class`, cierra etiquetas)

### Recursos
- 🌐 react.dev/learn/writing-markup-with-jsx — JSX paso a paso
- 🌐 react.dev/learn/javascript-in-jsx-with-curly-braces — expresiones con `{}`
- 🌐 transform.tools/html-to-jsx — convertidor HTML→JSX

---
`#react` `#jsx` `#componentes` `#frontend`
