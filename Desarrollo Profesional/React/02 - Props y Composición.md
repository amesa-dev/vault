# ⚛️ 02 — Props y Composición

[[Desarrollo Profesional/React/React|⬅️ Volver al índice]] | [[Desarrollo Profesional/React/01 - Componentes y JSX|← 01]] | [[Desarrollo Profesional/React/03 - Estado con useState|03 →]]

> [!abstract] Introducción
> Las **props** (properties) son los argumentos que un componente recibe de su padre. Son la vía por la que fluyen los datos en React: siempre de arriba hacia abajo, y siempre de **solo lectura**. Con TypeScript, tipar las props convierte cada componente en un contrato claro y verificado. La **composición** —componentes que envuelven a otros— es la forma idiomática de reutilizar en React, en lugar de la herencia.

## ¿De qué vamos a hablar?

Cómo se pasan y tipan las props, por qué son inmutables, el caso especial de `children`, valores por defecto, y por qué React prefiere composición a herencia.

### Conceptos que vamos a cubrir
- Pasar y recibir props; tipado con TypeScript
- Inmutabilidad: las props son de solo lectura
- La prop especial `children`
- Valores por defecto y desestructuración
- Composición vs herencia

---

## El Concepto

### Pasar y tipar props

El padre pasa props como atributos; el hijo las recibe como un objeto. Con TypeScript defines su forma con un `type` o `interface`.

```tsx
type AvatarProps = {
  nombre: string;
  urlImagen: string;
  tamano?: number; // opcional (el ? lo marca)
};

function Avatar({ nombre, urlImagen, tamano = 48 }: AvatarProps) {
  return (
    <img
      src={urlImagen}
      alt={`Foto de ${nombre}`}
      width={tamano}
      height={tamano}
    />
  );
}

// Uso
<Avatar nombre="Lucía" urlImagen="/lucia.jpg" tamano={96} />
<Avatar nombre="Pedro" urlImagen="/pedro.jpg" /> // usa tamano = 48
```

Desestructurar las props en la firma (`{ nombre, urlImagen }`) es lo idiomático: queda claro de un vistazo qué recibe el componente.

### Las props son de solo lectura

Un componente **nunca** debe modificar sus props. React asume que, para los mismos props, un componente renderiza lo mismo (es una función pura). Mutarlas rompe ese contrato y provoca bugs.

```tsx
// MAL — mutar una prop
function Lista({ items }: { items: string[] }) {
  items.push("nuevo"); // ❌ efecto secundario sobre datos del padre
  return <ul>{items.map((i) => <li key={i}>{i}</li>)}</ul>;
}

// BIEN — derivar datos nuevos sin tocar los props
function Lista({ items }: { items: string[] }) {
  const ordenados = [...items].sort(); // copia
  return <ul>{ordenados.map((i) => <li key={i}>{i}</li>)}</ul>;
}
```

Si un dato necesita cambiar con el tiempo, no es una prop: es **estado** (lo veremos en [[Desarrollo Profesional/React/03 - Estado con useState|03]]).

### `children`: el contenido entre etiquetas

`children` es una prop especial: contiene lo que el padre pone **entre** las etiquetas de apertura y cierre. Es la base de los componentes "contenedor".

```tsx
type CardProps = {
  titulo: string;
  children: React.ReactNode; // cualquier cosa renderizable
};

function Card({ titulo, children }: CardProps) {
  return (
    <section className="card">
      <h2>{titulo}</h2>
      <div className="card-body">{children}</div>
    </section>
  );
}

// Uso — todo lo de dentro llega como children
<Card titulo="Perfil">
  <p>Ingeniera de software</p>
  <button>Seguir</button>
</Card>
```

`React.ReactNode` es el tipo correcto para `children`: admite JSX, strings, números, arrays y `null`.

### Composición vs herencia

En programación orientada a objetos reutilizarías con herencia. React **no usa herencia entre componentes**: reutiliza **componiendo**. Un componente genérico (`Card`, `Modal`, `Layout`) recibe por `children` o por props el contenido específico.

```tsx
// Un Dialog genérico que se especializa por composición
function Dialog({ titulo, children, footer }: {
  titulo: string;
  children: React.ReactNode;
  footer?: React.ReactNode;
}) {
  return (
    <div className="dialog">
      <header>{titulo}</header>
      <div className="content">{children}</div>
      {footer && <footer>{footer}</footer>}
    </div>
  );
}

// Especialización: un diálogo de confirmación, sin heredar nada
function ConfirmarBorrado({ onConfirmar }: { onConfirmar: () => void }) {
  return (
    <Dialog
      titulo="¿Borrar elemento?"
      footer={<button onClick={onConfirmar}>Sí, borrar</button>}
    >
      <p>Esta acción no se puede deshacer.</p>
    </Dialog>
  );
}
```

> [!tip] "Pasar componentes como props"
> Cuando pasas JSX por una prop (como `footer` arriba), estás invirtiendo el control: el componente genérico decide *dónde* va, y el que lo usa decide *qué* va. Es el patrón más potente de composición en React.

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - Las **props** son los argumentos de un componente: fluyen de **padre a hijo** y son de **solo lectura**.
> - **Tipa las props** con TypeScript (`type`/`interface`) para tener un contrato verificado; desestructúralas en la firma.
> - Un dato que cambia con el tiempo no es prop, es **estado**.
> - **`children`** (tipo `React.ReactNode`) es lo que va entre las etiquetas: la base de los componentes contenedor.
> - React reutiliza por **composición**, no por herencia: componentes genéricos especializados vía `children` y props de tipo JSX.

### Para llevar a la práctica
- [ ] Crea un componente `Card` reutilizable con `titulo` y `children`, y úsalo dos veces con contenidos distintos
- [ ] Añade una prop opcional con valor por defecto y comprueba que TypeScript te avisa si pasas un tipo erróneo
- [ ] Refactoriza dos componentes parecidos para que compartan uno genérico por composición

### Recursos
- 🌐 react.dev/learn/passing-props-to-a-component — props en detalle
- 🌐 react.dev/learn/passing-data-deeply-with-context — cuándo las props se quedan cortas (preludio de Context)
- 🌐 react.dev/learn/keeping-components-pure — pureza y por qué no mutar props

---
`#react` `#props` `#composicion` `#typescript`
