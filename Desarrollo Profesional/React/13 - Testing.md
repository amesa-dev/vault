# ⚛️ 13 — Testing

[[Desarrollo Profesional/React/React|⬅️ Volver al índice]] | [[Desarrollo Profesional/React/12 - Performance y Optimización|← 12]]

> [!abstract] Introducción
> Testear componentes da confianza para cambiarlos sin romperlos. La filosofía moderna, encarnada en **React Testing Library (RTL)**, es clara: **testea como usa la app un usuario real**, no los detalles internos. No compruebes "el estado vale 1"; comprueba "aparece el texto correcto al hacer clic". Combinada con **Vitest** (el runner rápido del ecosistema Vite), es el estándar actual.

## ¿De qué vamos a hablar?

El stack Vitest + RTL, la filosofía de testear comportamiento, cómo buscar elementos por rol, simular interacciones con `user-event`, y un ejemplo completo.

### Conceptos que vamos a cubrir
- El stack: Vitest + React Testing Library
- La filosofía: testear comportamiento, no implementación
- Queries accesibles (`getByRole`, `getByText`)
- `user-event` para simular al usuario

---

## El Concepto

### El stack y un primer test

Con un proyecto Vite, instalas `vitest`, `@testing-library/react`, `@testing-library/user-event` y `jsdom`. Un test renderiza el componente y hace aserciones sobre lo que el usuario vería.

```tsx
// Boton.tsx
export function Boton({ onClick }: { onClick: () => void }) {
  return <button onClick={onClick}>Enviar</button>;
}

// Boton.test.tsx
import { render, screen } from "@testing-library/react";
import { describe, it, expect, vi } from "vitest";
import userEvent from "@testing-library/user-event";
import { Boton } from "./Boton";

describe("Boton", () => {
  it("llama a onClick al pulsarse", async () => {
    const onClick = vi.fn(); // mock/spy de Vitest
    render(<Boton onClick={onClick} />);

    await userEvent.click(screen.getByRole("button", { name: "Enviar" }));

    expect(onClick).toHaveBeenCalledOnce();
  });
});
```

### Testear comportamiento, no implementación

La diferencia clave de RTL frente a enfoques antiguos:

```tsx
// MAL — testear detalles internos (estado, nombres de métodos)
// Si refactorizas el componente sin cambiar lo que ve el usuario, el test se rompe.
expect(componente.state.contador).toBe(1);

// BIEN — testear lo que el usuario percibe
// Sobrevive a refactors internos; solo se rompe si cambia el comportamiento visible.
expect(screen.getByText("Has pulsado 1 vez")).toBeInTheDocument();
```

Esto hace los tests **resistentes a refactors** y, de paso, te empuja a UIs accesibles (porque buscas por rol y texto, como un lector de pantalla).

### Buscar elementos: queries accesibles

RTL prioriza buscar como lo haría una persona. Orden de preferencia: por **rol** (`getByRole`), por **etiqueta/texto** (`getByLabelText`, `getByText`), y como último recurso `getByTestId`.

```tsx
screen.getByRole("button", { name: "Guardar" }); // un botón con ese texto accesible
screen.getByRole("heading", { level: 1 });          // el <h1>
screen.getByLabelText("Email");                     // el input asociado a esa label
screen.getByText("Bienvenida");                     // texto suelto

// Variantes: queryBy* (no lanza, devuelve null) y findBy* (async, espera a que aparezca)
expect(screen.queryByText("Error")).not.toBeInTheDocument();
expect(await screen.findByText("Cargado")).toBeInTheDocument();
```

### Un ejemplo completo con interacción y estado

```tsx
// Contador.tsx
import { useState } from "react";
export function Contador() {
  const [n, setN] = useState(0);
  return <button onClick={() => setN(n + 1)}>Pulsado {n} veces</button>;
}

// Contador.test.tsx
import { render, screen } from "@testing-library/react";
import { it, expect } from "vitest";
import userEvent from "@testing-library/user-event";
import { Contador } from "./Contador";

it("incrementa al pulsar", async () => {
  render(<Contador />);
  const boton = screen.getByRole("button");

  expect(boton).toHaveTextContent("Pulsado 0 veces");
  await userEvent.click(boton);
  await userEvent.click(boton);
  expect(boton).toHaveTextContent("Pulsado 2 veces"); // comprobamos lo que se ve
});
```

> [!tip] La pirámide aplicada a React
> Muchos tests pequeños de componentes y hooks (rápidos), algunos de integración (varios componentes juntos) y unos pocos end-to-end con **Playwright** o **Cypress** para los flujos críticos. Encaja con la [[Desarrollo Profesional/Estrategia de Testing/Estrategia de Testing|Estrategia de Testing]] general del vault.

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - El stack moderno es **Vitest** (runner rápido de Vite) + **React Testing Library** + **`user-event`**.
> - Filosofía RTL: **testea comportamiento visible, no implementación**; los tests sobreviven a refactors.
> - Busca elementos con **queries accesibles**: `getByRole` primero, luego texto/label; `getByTestId` como último recurso.
> - Variantes: **`queryBy*`** (no lanza), **`findBy*`** (async, espera). Simula al usuario con **`userEvent`**.
> - Aplica la **pirámide**: muchos tests de componente, algunos de integración, pocos e2e (Playwright/Cypress).

### Para llevar a la práctica
- [ ] Configura Vitest + RTL en un proyecto Vite y escribe el test del botón
- [ ] Testea un contador comprobando el texto visible, no el estado interno
- [ ] Usa `findByText` para testear un componente que carga datos de forma asíncrona

### Recursos
- 🌐 testing-library.com/docs/react-testing-library/intro — RTL oficial
- 🌐 vitest.dev — Vitest
- 🌐 testing-library.com/docs/queries/about/#priority — prioridad de queries accesibles

---
`#react` `#testing` `#vitest` `#testing-library`
