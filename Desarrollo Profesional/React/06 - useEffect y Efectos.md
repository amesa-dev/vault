# ⚛️ 06 — useEffect y Efectos

[[Desarrollo Profesional/React/React|⬅️ Volver al índice]] | [[Desarrollo Profesional/React/05 - Eventos y Formularios|← 05]] | [[Desarrollo Profesional/React/07 - Hooks Avanzados|07 →]]

> [!abstract] Introducción
> Un **efecto** es código que sincroniza tu componente con un sistema **externo** a React: una suscripción, un temporizador, una API del navegador. `useEffect` ejecuta ese código *después* del render. Es uno de los hooks más usados y, a la vez, el peor entendido: la clave es saber **cuándo NO usarlo**. Muchos efectos son código que no debería ser un efecto.

## ¿De qué vamos a hablar?

Qué es un efecto y para qué sirve, el array de dependencias, la función de limpieza (cleanup), el doble montaje de StrictMode, y los antipatrones más comunes.

### Conceptos que vamos a cubrir
- `useEffect`: sincronización con sistemas externos
- El array de dependencias y sus tres modos
- Cleanup: limpiar suscripciones y timers
- Cuándo NO necesitas un efecto

---

## El Concepto

### Qué es un efecto

`useEffect(fn, deps)` ejecuta `fn` después de que React pinte el render. Sirve para "salir" de React: hablar con el mundo exterior.

```tsx
import { useState, useEffect } from "react";

function Reloj() {
  const [hora, setHora] = useState(new Date());

  useEffect(() => {
    const id = setInterval(() => setHora(new Date()), 1000); // sistema externo: timer
    return () => clearInterval(id); // cleanup: lo paramos al desmontar
  }, []); // [] → solo al montar

  return <p>{hora.toLocaleTimeString()}</p>;
}
```

### El array de dependencias

El segundo argumento controla **cuándo** se reejecuta el efecto:

```tsx
useEffect(() => { /* ... */ });            // sin array: tras CADA render (raro, casi siempre un bug)
useEffect(() => { /* ... */ }, []);         // []: solo al montar (y cleanup al desmontar)
useEffect(() => { /* ... */ }, [userId]);   // [userId]: al montar y cada vez que userId cambie
```

Regla de oro: **el array debe contener todos los valores reactivos** (props, estado) que el efecto usa. El linter `eslint-plugin-react-hooks` te avisa. Omitir dependencias produce bugs sutiles con valores obsoletos.

### Cleanup: limpiar lo que abriste

Si un efecto crea algo que persiste (timer, suscripción, conexión), debe **devolver una función** que lo limpie. React la ejecuta antes de re-ejecutar el efecto y al desmontar el componente.

```tsx
function SalaChat({ salaId }: { salaId: string }) {
  useEffect(() => {
    const conexion = crearConexion(salaId);
    conexion.conectar();
    return () => conexion.desconectar(); // se desconecta de la sala vieja antes de conectar a la nueva
  }, [salaId]);

  return <h1>Sala {salaId}</h1>;
}
```

> [!info] El doble montaje de StrictMode
> En desarrollo, `StrictMode` monta cada componente, lo desmonta y lo vuelve a montar. Verás tus efectos ejecutarse **dos veces**. No es un bug: es una prueba de que tu cleanup funciona. Si algo se rompe con el doble montaje, te falta limpiar bien.

### Cuándo NO necesitas un efecto

El error más común es usar efectos para cosas que React ya resuelve. Dos antipatrones frecuentes:

```tsx
// MAL — estado derivado en un efecto (re-render extra, datos desincronizados)
const [nombre, setNombre] = useState("");
const [apellido, setApellido] = useState("");
const [completo, setCompleto] = useState("");
useEffect(() => {
  setCompleto(nombre + " " + apellido);
}, [nombre, apellido]);

// BIEN — calcúlalo durante el render, sin estado ni efecto
const completo = nombre + " " + apellido;
```

```tsx
// MAL — responder a un evento del usuario dentro de un efecto
useEffect(() => {
  if (enviado) enviarAnalitica("compra");
}, [enviado]);

// BIEN — la lógica de un evento va en el handler del evento
function onComprar() {
  enviarAnalitica("compra");
  // ...
}
```

Pregúntate: **¿esto sincroniza con un sistema externo?** Si no (es un cálculo o una reacción a un evento), probablemente no es un efecto. Para data fetching, lo idiomático hoy es una librería como TanStack Query (ver [[Desarrollo Profesional/React/10 - Data Fetching|10]]) en lugar de `useEffect` a mano.

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - Un **efecto** sincroniza el componente con un **sistema externo** (timers, suscripciones, APIs del navegador) y corre **después** del render.
> - El **array de dependencias** decide cuándo se reejecuta: sin array (cada render), `[]` (solo montar), `[deps]` (cuando cambian). Incluye **todos** los valores reactivos que uses.
> - Devuelve una **función de cleanup** para limpiar lo que abriste; corre antes de reejecutar y al desmontar.
> - **StrictMode** monta dos veces en desarrollo: es una prueba de tu cleanup, no un bug.
> - **Muchos efectos sobran**: el estado derivado se calcula en el render y la lógica de eventos va en sus handlers.

### Para llevar a la práctica
- [ ] Crea un reloj con `setInterval` y su cleanup; comprueba que no se acumulan timers
- [ ] Sustituye un "estado derivado en efecto" por un cálculo directo en el render
- [ ] Activa el linter de hooks y deja que te corrija un array de dependencias incompleto

### Recursos
- 🌐 react.dev/learn/synchronizing-with-effects — efectos y cleanup
- 🌐 react.dev/learn/you-might-not-need-an-effect — lectura imprescindible sobre antipatrones
- 🌐 react.dev/reference/react/useEffect — referencia del hook

---
`#react` `#useeffect` `#efectos` `#hooks`
