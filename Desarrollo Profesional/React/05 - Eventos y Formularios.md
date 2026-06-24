# ⚛️ 05 — Eventos y Formularios

[[Desarrollo Profesional/React/React|⬅️ Volver al índice]] | [[Desarrollo Profesional/React/04 - Renderizado Condicional y Listas|← 04]] | [[Desarrollo Profesional/React/06 - useEffect y Efectos|06 →]]

> [!abstract] Introducción
> La interactividad nace de los **eventos**: clics, pulsaciones de tecla, envíos de formulario. React envuelve los eventos del DOM en *eventos sintéticos* con una API consistente entre navegadores. En los formularios, el patrón idiomático es el **controlled component**: el estado de React es la única fuente de verdad del valor de cada campo.

## ¿De qué vamos a hablar?

Cómo manejar eventos, el tipado de los handlers con TypeScript, los componentes controlados, gestionar varios campos a la vez, y el envío de un formulario con validación básica.

### Conceptos que vamos a cubrir
- Event handlers y eventos sintéticos
- Tipado de eventos con TypeScript
- Controlled components: `value` + `onChange`
- Un formulario con varios campos y su envío

---

## El Concepto

### Manejar eventos

Pasas una **función** (no su resultado) al atributo del evento, en camelCase:

```tsx
function Boton() {
  function manejarClick() {
    console.log("clic");
  }
  return <button onClick={manejarClick}>Pulsa</button>;
  // OJO: onClick={manejarClick}  ✅   no  onClick={manejarClick()}  ❌ (lo llamaría al renderizar)
}
```

Para pasar argumentos, envuelve en una flecha:

```tsx
<button onClick={() => borrar(producto.id)}>Borrar</button>
```

### Tipar eventos con TypeScript

Los handlers reciben un evento sintético. TypeScript tiene tipos precisos para cada uno:

```tsx
function Formulario() {
  // evento de cambio en un input
  function onChange(e: React.ChangeEvent<HTMLInputElement>) {
    console.log(e.target.value);
  }
  // evento de envío del formulario
  function onSubmit(e: React.FormEvent<HTMLFormElement>) {
    e.preventDefault(); // evita la recarga de página por defecto
  }
  // evento de clic
  function onClick(e: React.MouseEvent<HTMLButtonElement>) {}

  return null;
}
```

### Controlled components

En un componente controlado, el `value` del input lo dicta el estado, y cada cambio lo actualiza con `onChange`. El estado es la **única fuente de verdad**.

```tsx
import { useState } from "react";

function CampoNombre() {
  const [nombre, setNombre] = useState("");

  return (
    <div>
      <input
        value={nombre}
        onChange={(e) => setNombre(e.target.value)}
        placeholder="Tu nombre"
      />
      <p>Hola, {nombre || "desconocido"}</p>
    </div>
  );
}
```

El flujo es circular: el usuario teclea → `onChange` actualiza el estado → React re-renderiza → el input muestra el nuevo `value`.

### Un formulario con varios campos

Para varios campos, un objeto de estado y un handler genérico que usa el `name` de cada input:

```tsx
type Datos = { email: string; password: string };

function Registro() {
  const [datos, setDatos] = useState<Datos>({ email: "", password: "" });
  const [error, setError] = useState<string | null>(null);

  function onChange(e: React.ChangeEvent<HTMLInputElement>) {
    const { name, value } = e.target;
    setDatos((prev) => ({ ...prev, [name]: value })); // clave computada por name
  }

  function onSubmit(e: React.FormEvent<HTMLFormElement>) {
    e.preventDefault();
    if (!datos.email.includes("@")) {
      setError("Email no válido");
      return;
    }
    setError(null);
    console.log("Enviando", datos);
  }

  return (
    <form onSubmit={onSubmit}>
      <input name="email" value={datos.email} onChange={onChange} />
      <input name="password" type="password" value={datos.password} onChange={onChange} />
      {error && <p className="error">{error}</p>}
      <button type="submit">Crear cuenta</button>
    </form>
  );
}
```

> [!tip] Para formularios complejos, no reinventes la rueda
> En formularios grandes (muchos campos, validación rica, errores por campo) usa **React Hook Form** + un validador como **Zod**. Reducen el boilerplate y mejoran el rendimiento al evitar re-renders por tecla. Para 1–3 campos, el patrón controlado manual sobra.

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - Pasa **la función** al handler (`onClick={fn}`), no su llamada (`onClick={fn()}`); envuelve en flecha para pasar argumentos.
> - Tipa los eventos con los tipos de React: `ChangeEvent<HTMLInputElement>`, `FormEvent<HTMLFormElement>`, etc.
> - En un **controlled component**, el `value` viene del estado y `onChange` lo actualiza: el estado es la fuente de verdad.
> - Para varios campos, un **objeto de estado** y un handler genérico con `[name]: value` (clave computada).
> - En el `onSubmit`, llama a **`e.preventDefault()`** para evitar la recarga; valida antes de enviar.

### Para llevar a la práctica
- [ ] Crea un input controlado que muestre en vivo lo que escribes
- [ ] Monta un formulario de login con dos campos, handler genérico y validación de email
- [ ] Prueba React Hook Form + Zod en un formulario de varios campos y compara el boilerplate

### Recursos
- 🌐 react.dev/learn/responding-to-events — eventos en React
- 🌐 react.dev/learn/reacting-to-input-with-state — pensar formularios con estado
- 🌐 react-hook-form.com — la librería estándar para formularios complejos

---
`#react` `#eventos` `#formularios` `#typescript`
