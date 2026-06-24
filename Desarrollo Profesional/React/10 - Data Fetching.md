# ⚛️ 10 — Data Fetching

[[Desarrollo Profesional/React/React|⬅️ Volver al índice]] | [[Desarrollo Profesional/React/09 - Context y Estado Global|← 09]] | [[Desarrollo Profesional/React/11 - React Router|11 →]]

> [!abstract] Introducción
> Casi toda app necesita datos de un servidor. Hacerlo a mano con `useEffect` + `useState` parece fácil, pero pronto aparecen carga, errores, caché, refetch, condiciones de carrera… El **estado del servidor** es distinto del estado local: es asíncrono, se comparte y puede quedar obsoleto. **TanStack Query** (antes React Query) es la herramienta estándar para gestionarlo bien.

## ¿De qué vamos a hablar?

Por qué el data fetching manual se complica, qué problemas resuelve TanStack Query, cómo hacer queries y mutations, y la idea de caché e invalidación.

### Conceptos que vamos a cubrir
- El problema del fetching manual con useEffect
- `useQuery`: carga, error y caché automáticos
- `useMutation` e invalidación de caché
- Estado del servidor vs estado del cliente

---

## El Concepto

### El problema del fetching manual

```tsx
// El patrón "a mano": funciona, pero le faltan muchas cosas
function Usuarios() {
  const [datos, setDatos] = useState<Usuario[]>([]);
  const [cargando, setCargando] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  useEffect(() => {
    fetch("/api/usuarios")
      .then((r) => r.json())
      .then(setDatos)
      .catch(setError)
      .finally(() => setCargando(false));
  }, []);
  // ❌ sin caché, sin reintentos, sin refetch al volver a la pestaña,
  //    sin protección ante condiciones de carrera, repetido en cada componente
}
```

### useQuery: lo esencial resuelto

TanStack Query encapsula todo eso. Defines una **query key** (identidad del dato en caché) y una **query function** (cómo obtenerlo); él te da `data`, `isLoading`, `isError` y caché.

```tsx
import { QueryClient, QueryClientProvider, useQuery } from "@tanstack/react-query";

const queryClient = new QueryClient();

// Se envuelve la app una vez
function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <Usuarios />
    </QueryClientProvider>
  );
}

async function getUsuarios(): Promise<Usuario[]> {
  const r = await fetch("/api/usuarios");
  if (!r.ok) throw new Error("Error al cargar usuarios");
  return r.json();
}

function Usuarios() {
  const { data, isLoading, isError, error } = useQuery({
    queryKey: ["usuarios"],   // identidad en caché
    queryFn: getUsuarios,     // cómo obtener el dato
  });

  if (isLoading) return <p>Cargando…</p>;
  if (isError) return <p>Error: {error.message}</p>;
  return <ul>{data!.map((u) => <li key={u.id}>{u.nombre}</li>)}</ul>;
}
```

Gratis y de serie: **caché** (si otro componente pide `["usuarios"]`, reusa el dato), **deduplicación** de peticiones simultáneas, **refetch** al reenfocar la ventana, **reintentos** ante fallo y datos "frescos vs obsoletos" (`staleTime`).

### useMutation e invalidación

Para crear/editar/borrar usas `useMutation`. Tras una mutación exitosa, **invalidas** las queries afectadas para que se refresquen con los datos nuevos.

```tsx
import { useMutation, useQueryClient } from "@tanstack/react-query";

function NuevoUsuario() {
  const queryClient = useQueryClient();

  const mutation = useMutation({
    mutationFn: (nombre: string) =>
      fetch("/api/usuarios", { method: "POST", body: JSON.stringify({ nombre }) }),
    onSuccess: () => {
      // marca ["usuarios"] como obsoleto → TanStack la vuelve a pedir
      queryClient.invalidateQueries({ queryKey: ["usuarios"] });
    },
  });

  return <button onClick={() => mutation.mutate("Nuevo")}>Crear</button>;
}
```

### Estado del servidor vs estado del cliente

La idea mental clave:

| | Estado del cliente | Estado del servidor |
|---|---|---|
| Ejemplos | tema, modal abierto, formulario | usuarios, productos, pedidos |
| Dueño | tu app | el backend |
| Naturaleza | síncrono | asíncrono, compartido, cacheable |
| Herramienta | `useState` / Zustand | **TanStack Query** |

No metas datos del servidor en `useState`/Zustand a mano: trátalos como **caché del servidor** con TanStack Query y te ahorras una clase entera de bugs.

> [!tip] Frameworks con fetching integrado
> Si usas **Next.js** o **React Router** en modo framework, parte de esto se resuelve en el servidor con Server Components o `loaders` (ver [[Desarrollo Profesional/React/11 - React Router|11]]). TanStack Query sigue siendo el rey para fetching del lado del cliente en SPAs.

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - El **estado del servidor** es asíncrono, compartido y cacheable: distinto del estado local.
> - El fetching manual con `useEffect` carece de caché, reintentos, dedupe y protección ante carreras.
> - **`useQuery`** te da `data`/`isLoading`/`isError` y caché por **query key**, con refetch y reintentos de serie.
> - **`useMutation`** + **`invalidateQueries`** es el patrón para escribir y mantener la caché fresca.
> - Regla: **no guardes datos del servidor en `useState`**; trátalos como caché con TanStack Query.

### Para llevar a la práctica
- [ ] Sustituye un `useEffect`+`fetch` por `useQuery` y observa la caché entre componentes
- [ ] Implementa un `useMutation` para crear un recurso e invalida su lista al terminar
- [ ] Juega con `staleTime` y el refetch al reenfocar la pestaña

### Recursos
- 🌐 tanstack.com/query/latest — documentación oficial de TanStack Query
- 🌐 tkdodo.eu/blog — serie "Practical React Query", la mejor guía de patrones
- 🌐 react.dev/learn/you-might-not-need-an-effect#fetching-data — por qué no fetch con efectos

---
`#react` `#data-fetching` `#tanstack-query` `#async`
