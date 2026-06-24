# ⚛️ 11 — React Router

[[Desarrollo Profesional/React/React|⬅️ Volver al índice]] | [[Desarrollo Profesional/React/10 - Data Fetching|← 10]] | [[Desarrollo Profesional/React/12 - Performance y Optimización|12 →]]

> [!abstract] Introducción
> React es solo la capa de vista: no trae enrutado. **React Router** es la librería estándar para construir SPAs con varias "páginas" sin recargar el navegador. Define qué componente se muestra según la URL, gestiona parámetros, navegación programática y, en sus versiones modernas, carga de datos por ruta con `loaders`.

## ¿De qué vamos a hablar?

Cómo declarar rutas, navegar con `Link` y de forma programática, leer parámetros y query strings, rutas anidadas con layouts, y la idea de `loader` para cargar datos.

### Conceptos que vamos a cubrir
- Declarar rutas y renderizar la app enrutada
- `Link` y navegación programática con `useNavigate`
- Parámetros de ruta (`useParams`) y query (`useSearchParams`)
- Rutas anidadas, `Outlet` y `loaders`

---

## El Concepto

### Declarar rutas

Defines un mapa de rutas con `createBrowserRouter` y lo renderizas con `RouterProvider`:

```tsx
import { createBrowserRouter, RouterProvider } from "react-router-dom";

const router = createBrowserRouter([
  { path: "/", element: <Inicio /> },
  { path: "/productos", element: <Productos /> },
  { path: "/productos/:id", element: <DetalleProducto /> }, // :id es un parámetro
  { path: "*", element: <NoEncontrado /> }, // catch-all 404
]);

function App() {
  return <RouterProvider router={router} />;
}
```

### Navegar: Link y useNavigate

Para enlaces declarativos, **`Link`** (no uses `<a href>`: recargaría la página). Para navegar desde código (tras un login, por ejemplo), **`useNavigate`**:

```tsx
import { Link, useNavigate } from "react-router-dom";

function Menu() {
  return (
    <nav>
      <Link to="/">Inicio</Link>
      <Link to="/productos">Productos</Link>
    </nav>
  );
}

function FormLogin() {
  const navigate = useNavigate();
  function onSuccess() {
    navigate("/dashboard"); // navegación programática
  }
  // ...
}
```

### Parámetros de ruta y query string

`useParams` lee los segmentos dinámicos (`:id`); `useSearchParams` lee y escribe la query (`?orden=precio`):

```tsx
import { useParams, useSearchParams } from "react-router-dom";

function DetalleProducto() {
  const { id } = useParams();             // de /productos/:id
  return <h1>Producto {id}</h1>;
}

function Productos() {
  const [params, setParams] = useSearchParams(); // de /productos?orden=precio
  const orden = params.get("orden") ?? "nombre";
  return (
    <button onClick={() => setParams({ orden: "precio" })}>
      Ordenar (actual: {orden})
    </button>
  );
}
```

### Rutas anidadas, Outlet y loaders

Las rutas anidadas comparten un **layout** común. El padre renderiza `<Outlet />` donde van los hijos. Y cada ruta puede tener un **`loader`** que carga datos *antes* de renderizar (evitando spinners en cascada).

```tsx
import { Outlet, useLoaderData } from "react-router-dom";

// Layout padre con navegación común
function LayoutTienda() {
  return (
    <div>
      <nav>/* menú común */</nav>
      <Outlet /> {/* aquí se inyecta la ruta hija activa */}
    </div>
  );
}

const router = createBrowserRouter([
  {
    path: "/tienda",
    element: <LayoutTienda />,
    children: [
      {
        path: "productos/:id",
        element: <DetalleProducto />,
        loader: async ({ params }) => {
          const r = await fetch(`/api/productos/${params.id}`);
          return r.json(); // disponible vía useLoaderData()
        },
      },
    ],
  },
]);

function DetalleProducto() {
  const producto = useLoaderData() as Producto; // datos ya cargados por el loader
  return <h1>{producto.nombre}</h1>;
}
```

> [!tip] Router framework vs librería, y la relación con TanStack Query
> React Router moderno puede usarse como simple librería de rutas o como "framework" (con SSR, loaders y actions). Si ya usas [[Desarrollo Profesional/React/10 - Data Fetching|TanStack Query]], muchos prefieren cargar datos dentro de los componentes con `useQuery` y dejar al router solo el enrutado. Ambos enfoques son válidos; elige uno y sé consistente.

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - React no trae enrutado: **React Router** mapea URLs a componentes en una SPA sin recargar.
> - Define rutas con **`createBrowserRouter`** + **`RouterProvider`**; usa `*` para el 404.
> - Navega con **`Link`** (declarativo) y **`useNavigate`** (programático); nunca `<a href>` interno.
> - Lee parámetros con **`useParams`** (`:id`) y la query con **`useSearchParams`** (`?clave=valor`).
> - Rutas **anidadas** comparten layout vía **`<Outlet />`**; los **`loaders`** cargan datos antes de renderizar.

### Para llevar a la práctica
- [ ] Crea una app con 3 rutas (inicio, lista, detalle con `:id`) y una 404
- [ ] Añade un layout con menú común usando rutas anidadas y `<Outlet />`
- [ ] Lee y modifica un parámetro de query (orden o filtro) con `useSearchParams`

### Recursos
- 🌐 reactrouter.com — documentación oficial
- 🌐 reactrouter.com/start/library/routing — guía de inicio como librería
- 🌐 react.dev/learn/start-a-new-react-project — frameworks (Next.js, React Router) recomendados

---
`#react` `#react-router` `#routing` `#spa`
