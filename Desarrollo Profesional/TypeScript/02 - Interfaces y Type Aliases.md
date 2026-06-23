# 📘 02 — Interfaces y Type Aliases

[[Desarrollo Profesional/TypeScript/TypeScript|⬅️ Volver al índice]] | [[Desarrollo Profesional/TypeScript/01 - Tipos Básicos|← 01]] | [[Desarrollo Profesional/TypeScript/03 - Funciones|03 →]]

> [!abstract] Introducción
> TypeScript tiene dos formas de definir la forma de un objeto: `interface` y `type`. La pregunta "¿cuándo uso cada uno?" aparece en casi todas las bases de código TypeScript. La respuesta honesta es: para objetos, casi siempre puedes usar cualquiera. Pero hay diferencias importantes en casos específicos que determinan cuál elegir.

## ¿De qué vamos a hablar?

`interface` y `type` son las herramientas fundamentales para modelar datos en TypeScript. Esta página explica sus diferencias reales (no solo las que dicen las docs) y cuándo cada uno es la elección correcta.

### Conceptos que vamos a cubrir
- `interface`: cuándo y cómo
- `type`: lo que puede y `interface` no puede
- La diferencia que importa: declaration merging
- Uniones e intersecciones
- Tipos de utilidad sobre objetos: Pick, Omit, Partial, Required, Record

---

## El Concepto

### `interface` — Contratos para Objetos

Una `interface` define la forma de un objeto. Puede extenderse, implementarse en clases y fusionarse con otras declaraciones del mismo nombre.

```typescript
interface Usuario {
  readonly id: number;      // solo lectura — no se puede reasignar
  nombre: string;
  email: string;
  edad?: number;            // opcional — puede ser undefined
}

// Uso básico
const u: Usuario = {
  id: 1,
  nombre: "Andrés",
  email: "a@b.com",
};

u.id = 2;  // Error: Cannot assign to 'id' because it is a read-only property

// Extensión
interface Empleado extends Usuario {
  departamento: string;
  salario: number;
}

// Múltiple herencia
interface Admin extends Empleado, Auditable {
  permisos: string[];
}

// Métodos en interfaces
interface Repositorio<T> {
  findById(id: number): Promise<T | null>;
  save(entity: T): Promise<T>;
  delete(id: number): Promise<void>;
}
```

### `type` — Aliases con Superpoderes

`type` puede hacer todo lo que hace `interface` con objetos, pero además puede:

```typescript
// 1. Alias de primitivos
type UserId = number;
type Email = string;

// 2. Uniones — el caso de uso más importante
type Resultado<T> = 
  | { ok: true; valor: T }
  | { ok: false; error: string };

type EventoTeclado = "keydown" | "keyup" | "keypress";

// 3. Intersecciones (equivalente a extends pero más composable)
type ConTimestamps = {
  creadoEn: Date;
  actualizadoEn: Date;
};

type UsuarioConTimestamps = Usuario & ConTimestamps;

// 4. Tuplas con nombres
type Coordenada = [x: number, y: number];
type RangoFecha = [inicio: Date, fin: Date];

// 5. Tipos función
type Transformador<T, R> = (entrada: T) => R;
type Predicado<T> = (item: T) => boolean;

// 6. Tipos condicionados, mapped types (ver página 07)
type ClavesDeString<T> = {
  [K in keyof T as T[K] extends string ? K : never]: T[K];
};
```

### La Diferencia que Importa: Declaration Merging

Esta es la diferencia fundamental entre `interface` y `type`:

```typescript
// INTERFACE: declaration merging — se puede declarar múltiples veces
// TypeScript las fusiona automáticamente
interface Ventana {
  ancho: number;
}
interface Ventana {
  alto: number;
}
// Resultado: interface Ventana { ancho: number; alto: number; }
const v: Ventana = { ancho: 800, alto: 600 }; // ✅

// TYPE: no permite redeclaración
type Pantalla = { ancho: number; };
type Pantalla = { alto: number; };  // Error: Duplicate identifier 'Pantalla'
```

**¿Por qué importa?** El declaration merging es el mecanismo que usan las librerías para que puedas extender sus tipos. Si añades métodos a `Array`, `Window`, o tipos de `Express`, necesitas interfaces.

```typescript
// Extender tipos de Express (ejemplo real)
declare namespace Express {
  interface Request {
    usuario?: Usuario;  // añadir campo custom al Request de Express
  }
}

// Extender tipos globales
interface Array<T> {
  primero(): T | undefined;
}
```

Si estás construyendo una librería que otros van a extender: usa `interface`. Si solo estás modelando datos de tu aplicación: usa lo que prefieras.

### Uniones e Intersecciones

#### Uniones — OR de tipos

```typescript
type Input = string | number | boolean;

// La unión fuerza a manejar todos los casos
function formatear(val: string | number): string {
  if (typeof val === "string") {
    return val.toUpperCase();
  }
  return val.toFixed(2);
}

// Discriminated unions — el patrón más poderoso con type
type Evento =
  | { tipo: "clic"; x: number; y: number }
  | { tipo: "tecla"; codigo: string; ctrl: boolean }
  | { tipo: "scroll"; delta: number };

function manejarEvento(e: Evento): void {
  switch (e.tipo) {
    case "clic":   console.log(e.x, e.y); break;    // TypeScript sabe que e tiene x e y
    case "tecla":  console.log(e.codigo); break;    // TypeScript sabe que e tiene codigo
    case "scroll": console.log(e.delta); break;
  }
}
```

#### Intersecciones — AND de tipos

```typescript
type ConId = { id: number };
type ConNombre = { nombre: string };
type Entidad = ConId & ConNombre;  // { id: number; nombre: string }

// Útil para componer tipos
type ConCreacion = { creadoEn: Date };
type ConAuditoria = { creadoPor: string; actualizadoPor: string };
type EntidadAuditada = Entidad & ConCreacion & ConAuditoria;
```

### Tipos de Utilidad para Objetos

TypeScript incluye tipos de utilidad que transforman otros tipos:

```typescript
interface Producto {
  id: number;
  nombre: string;
  precio: number;
  descripcion: string;
  categoria: string;
}

// Partial<T> — hace todas las propiedades opcionales
type ActualizarProducto = Partial<Producto>;
// { id?: number; nombre?: string; precio?: number; ... }

// Required<T> — hace todas obligatorias (inverso de Partial)
type ProductoCompleto = Required<Producto>;

// Pick<T, K> — selecciona un subconjunto de propiedades
type ResumenProducto = Pick<Producto, "id" | "nombre" | "precio">;
// { id: number; nombre: string; precio: number }

// Omit<T, K> — todas excepto las indicadas
type ProductoSinId = Omit<Producto, "id">;
// { nombre: string; precio: number; descripcion: string; categoria: string }

// Record<K, V> — mapa de clave a valor
type CatalogoPrecios = Record<string, number>;
// { [key: string]: number }

type EstadoPorId = Record<number, "activo" | "inactivo">;

// Readonly<T> — todas las propiedades son de solo lectura
type ProductoFijo = Readonly<Producto>;

// ReturnType<T> — extrae el tipo de retorno de una función
function crearProducto(): Producto { /* ... */ return {} as Producto; }
type TipoRetorno = ReturnType<typeof crearProducto>;  // Producto

// Parameters<T> — extrae los parámetros de una función como tuple
function buscar(query: string, limite: number): Producto[] { return []; }
type Params = Parameters<typeof buscar>;  // [query: string, limite: number]
```

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - `interface` y `type` son intercambiables para modelar objetos en la mayoría de casos. La convención más extendida hoy es usar `type` por defecto.
> - **La diferencia que importa:** `interface` soporta declaration merging (fusión de declaraciones). Úsala cuando construyes librerías o extiendes tipos de terceros. `type` no se puede redeclarar.
> - `type` puede hacer uniones, intersecciones, alias de primitivos, y tipos complejos que `interface` no puede.
> - Los discriminated unions (`tipo: "clic" | "tecla"`) son el patrón más poderoso para modelar estados y eventos — permiten exhaustividad verificada por el compilador.
> - Los utility types (Partial, Pick, Omit, Record...) evitan repetición y permiten derivar tipos de otros tipos.

### Para llevar a la práctica
- [ ] Modela un sistema de pagos con discriminated unions: `Pago = PagoTarjeta | PagoBizum | PagoTransferencia`
- [ ] Crea un tipo `CreateDTO<T>` que sea `Omit<T, "id" | "creadoEn" | "actualizadoEn">`
- [ ] Extiende el tipo `Request` de Express para añadir un campo `usuario` tipado

### Recursos
- 🌐 typescriptlang.org/docs/handbook/utility-types.html — todos los utility types
- 🌐 totaltypescript.com/type-vs-interface — análisis definitivo de la diferencia
- 📖 *Programming TypeScript* — Capítulo 5 (clases e interfaces)
- 🌐 *TypeScript Deep Dive* — capítulo "Interfaces"

---
`#typescript` `#interfaces` `#types` `#union` `#intersection`
