# ⚛️ 04 — Renderizado Condicional y Listas

[[Desarrollo Profesional/React/React|⬅️ Volver al índice]] | [[Desarrollo Profesional/React/03 - Estado con useState|← 03]] | [[Desarrollo Profesional/React/05 - Eventos y Formularios|05 →]]

> [!abstract] Introducción
> Casi toda UI real necesita dos cosas: mostrar elementos **según una condición** (¿está logueado? ¿hay error?) y renderizar **colecciones** de datos (una lista de productos, mensajes, tareas). React no tiene directivas tipo `*ngIf` o `v-for`: usa JavaScript puro dentro de las llaves. Y al renderizar listas aparece un concepto crítico para el rendimiento y la corrección: las **keys**.

## ¿De qué vamos a hablar?

Las técnicas de renderizado condicional (ternario, `&&`, `return null`), cómo transformar arrays en JSX con `.map()`, y por qué las `key` son obligatorias y cómo elegirlas bien.

### Conceptos que vamos a cubrir
- Condicional con ternario y con `&&`
- El cortocircuito `&&` y su trampa con números
- Renderizar listas con `.map()`
- Las `key`: qué son, por qué importan y cuál elegir

---

## El Concepto

### Renderizado condicional

Como JSX es JavaScript, condicionas con las herramientas del lenguaje. Para elegir entre dos opciones, el **ternario**:

```tsx
function Saludo({ logueado }: { logueado: boolean }) {
  return <h1>{logueado ? "Bienvenida de nuevo" : "Inicia sesión"}</h1>;
}
```

Para mostrar algo **o nada**, el operador `&&`:

```tsx
function Aviso({ mensaje }: { mensaje: string | null }) {
  return (
    <div>
      {mensaje && <p className="error">{mensaje}</p>}
    </div>
  );
}
```

Si toda la salida depende de una condición, puedes hacer un `return` temprano:

```tsx
function Perfil({ usuario }: { usuario: Usuario | null }) {
  if (!usuario) return null; // no renderiza nada
  return <h2>{usuario.nombre}</h2>;
}
```

> [!tip] La trampa del `&&` con números
> `cantidad && <p>...</p>` renderiza un **0** literal en pantalla si `cantidad` es 0, porque `0` es falsy pero React lo pinta. Usa siempre una condición booleana explícita: `cantidad > 0 && <p>...</p>`.

### Renderizar listas con `.map()`

Para convertir un array de datos en un array de elementos JSX, usa `.map()`:

```tsx
type Producto = { id: number; nombre: string; precio: number };

function ListaProductos({ productos }: { productos: Producto[] }) {
  return (
    <ul>
      {productos.map((p) => (
        <li key={p.id}>
          {p.nombre} — {p.precio.toFixed(2)} €
        </li>
      ))}
    </ul>
  );
}
```

### Las keys

Cada elemento de una lista necesita una prop **`key`** única **entre sus hermanos**. React la usa para identificar cada elemento entre renders: saber cuáles se añadieron, eliminaron o reordenaron, y así actualizar el DOM mínimamente y preservar el estado de cada fila.

```tsx
// MAL — usar el índice como key cuando la lista se reordena o filtra
{tareas.map((t, i) => <Tarea key={i} dato={t} />)}
// Si insertas al principio, todas las keys se "desplazan" y React
// asocia mal el estado (un input puede quedar con el texto de otra fila).

// BIEN — una key estable y única del propio dato
{tareas.map((t) => <Tarea key={t.id} dato={t} />)}
```

Reglas de las keys:
- **Únicas entre hermanos** (no globalmente).
- **Estables**: el mismo dato debe tener siempre la misma key (nada de `Math.random()`).
- Usa el **id** del dato. El índice del array solo es aceptable si la lista es estática y nunca se reordena ni filtra.

```
Sin keys correctas, al insertar "C" al principio:
  Antes:  [A(key0), B(key1)]
  Después:[C(key0), A(key1), B(key2)]   ← React cree que A pasó a ser C, etc.
            ↑ reusa el DOM equivocado, el estado se mezcla

Con keys por id:
  Antes:  [A(id=a), B(id=b)]
  Después:[C(id=c), A(id=a), B(id=b)]   ← React ve que A y B siguen igual
            ↑ solo inserta C, el resto intacto
```

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - Condiciona con **ternario** (elegir entre dos), **`&&`** (mostrar o nada) o **`return null`** (no renderizar).
> - Cuidado con **`&&` y el `0`**: usa condiciones booleanas explícitas (`n > 0 && ...`).
> - Renderiza colecciones transformando arrays con **`.map()`** que devuelve JSX.
> - Cada elemento de lista necesita una **`key` única entre hermanos, estable y derivada del id** del dato.
> - Usar el **índice como key** rompe el estado al reordenar/filtrar: evítalo salvo en listas estáticas.

### Para llevar a la práctica
- [ ] Renderiza una lista de objetos con `.map()` y `key` por id
- [ ] Provoca el bug del índice como key: una lista con inputs que reordenas, y observa cómo se cruza el estado
- [ ] Muestra un mensaje de "lista vacía" con renderizado condicional cuando el array no tenga elementos

### Recursos
- 🌐 react.dev/learn/conditional-rendering — renderizado condicional
- 🌐 react.dev/learn/rendering-lists — listas y keys en detalle
- 🌐 react.dev/learn/rendering-lists#why-does-react-need-keys — por qué React necesita keys

---
`#react` `#listas` `#keys` `#renderizado`
