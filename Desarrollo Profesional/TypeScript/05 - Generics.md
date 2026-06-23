# 📘 05 — Generics

[[Desarrollo Profesional/TypeScript/TypeScript|⬅️ Volver al índice]] | [[Desarrollo Profesional/TypeScript/04 - Clases|← 04]] | [[Desarrollo Profesional/TypeScript/06 - Narrowing y Type Guards|06 →]]

> [!abstract] Introducción
> Los generics son el mecanismo que permite escribir código que funciona con múltiples tipos sin perder la información de tipo. Son la diferencia entre una función que acepta `any` (y pierde todas las garantías) y una función que acepta "cualquier tipo que decidas, y el compilador recuerda cuál es". Los generics son lo que hace a TypeScript realmente potente.

## ¿De qué vamos a cubrir?

Los generics desde lo básico hasta los patrones avanzados: constraints, conditional types, y los utility types built-in que derivan tipos de otros tipos.

### Conceptos que vamos a cubrir
- Generics básicos en funciones, interfaces y clases
- Constraints: `extends` para limitar el tipo genérico
- Default type parameters
- Conditional types básicos
- Utility types: los más importantes y cómo funcionan internamente

---

## El Concepto

### Por Qué Generics

El problema que resuelven:

```typescript
// Sin generics: pierdes la información de tipo
function primero_any(array: any[]): any {
  return array[0];
}

const n = primero_any([1, 2, 3]);
n.toUpperCase(); // TypeScript no detecta el error — n es 'any'

// Con generics: el compilador recuerda el tipo
function primero<T>(array: T[]): T | undefined {
  return array[0];
}

const num = primero([1, 2, 3]);     // TypeScript infiere T = number
const str = primero(["a", "b"]);   // TypeScript infiere T = string

num.toUpperCase(); // Error: Property 'toUpperCase' does not exist on type 'number'
str.toUpperCase(); // ✅
```

El parámetro de tipo `T` es un placeholder. El compilador lo sustituye por el tipo real cuando infiere o cuando se especifica explícitamente.

### Funciones Genéricas

```typescript
// T se infiere del argumento
function envolver<T>(valor: T): { valor: T; timestamp: number } {
  return { valor, timestamp: Date.now() };
}

const envuelto = envolver(42);           // { valor: number; timestamp: number }
const envu2    = envolver("hola");       // { valor: string; timestamp: number }

// Múltiples parámetros de tipo
function par<A, B>(primero: A, segundo: B): [A, B] {
  return [primero, segundo];
}

const p = par(1, "hola");   // [number, string]
const q = par(true, [1,2]); // [boolean, number[]]

// Tipo de retorno que depende de los tipos de entrada
function intercambiar<A, B>(par: [A, B]): [B, A] {
  return [par[1], par[0]];
}

const r = intercambiar([1, "hola"]); // [string, number]

// Cuando TypeScript no puede inferir — especifica explícitamente
const resultado = envolver<boolean>(true);
```

### Constraints — Limitar el Tipo Genérico

Sin constraints, `T` puede ser cualquier tipo. Esto a veces es demasiado permisivo:

```typescript
// Error: no puedes acceder a .length en un T arbitrario
function longitudDe<T>(valor: T): number {
  return valor.length; // Error: Property 'length' does not exist on type 'T'
}

// Con constraint: T debe tener length
function longitudDe<T extends { length: number }>(valor: T): number {
  return valor.length; // ✅
}

longitudDe("hola");      // 4
longitudDe([1, 2, 3]);  // 3
longitudDe({ length: 5, datos: [] }); // 5
// longitudDe(42);       // Error: number no tiene length

// keyof constraint — la herramienta más poderosa para objetos
function obtener<T, K extends keyof T>(obj: T, clave: K): T[K] {
  return obj[clave];
}

const usuario = { nombre: "Andrés", edad: 30, activo: true };
const nombre = obtener(usuario, "nombre"); // string
const edad   = obtener(usuario, "edad");   // number
// obtener(usuario, "email"); // Error: "email" no es keyof typeof usuario

// Múltiples constraints
function merge<T extends object, U extends object>(a: T, b: U): T & U {
  return { ...a, ...b };
}
```

### Interfaces y Clases Genéricas

```typescript
// Interface genérica
interface Repositorio<T> {
  findById(id: number): Promise<T | null>;
  findAll(): Promise<T[]>;
  save(entity: T): Promise<T>;
  delete(id: number): Promise<void>;
}

interface PaginatedResult<T> {
  items: T[];
  total: number;
  pagina: number;
  porPagina: number;
}

// Clase genérica
class Cache<T> {
  private datos: Map<string, { valor: T; expira: number }> = new Map();

  set(clave: string, valor: T, ttlMs: number = 60_000): void {
    this.datos.set(clave, {
      valor,
      expira: Date.now() + ttlMs,
    });
  }

  get(clave: string): T | undefined {
    const entrada = this.datos.get(clave);
    if (!entrada) return undefined;
    if (Date.now() > entrada.expira) {
      this.datos.delete(clave);
      return undefined;
    }
    return entrada.valor;
  }
}

const cacheUsuarios = new Cache<{ id: number; nombre: string }>();
cacheUsuarios.set("user:1", { id: 1, nombre: "Andrés" });
```

### Default Type Parameters

```typescript
// T tiene un tipo por defecto si no se especifica
interface ApiResponse<T = unknown> {
  data: T;
  status: number;
  message: string;
}

const respuesta: ApiResponse = { data: {}, status: 200, message: "OK" };
// T = unknown por defecto

const respuestaTyped: ApiResponse<string[]> = {
  data: ["a", "b", "c"],
  status: 200,
  message: "OK"
};
```

### Conditional Types — Lo Básico

Los conditional types son genéricos que cambian según una condición:

```typescript
// Sintaxis: T extends U ? X : Y
type EsString<T> = T extends string ? true : false;

type A = EsString<string>;   // true
type B = EsString<number>;   // false
type C = EsString<"hola">;  // true — "hola" extends string

// Caso de uso real: extraer el tipo de retorno de una Promise
type Awaited<T> = T extends Promise<infer R> ? R : T;

type Resultado1 = Awaited<Promise<string>>;  // string
type Resultado2 = Awaited<number>;           // number (no es Promise)

// NonNullable — elimina null y undefined
type NonNullable<T> = T extends null | undefined ? never : T;
type A2 = NonNullable<string | null | undefined>; // string
```

### Utility Types — Cómo Funcionan Internamente

Los utility types de TypeScript son generics avanzados. Entender cómo están implementados ayuda a crear los tuyos:

```typescript
// Partial — hace todas las propiedades opcionales
type MyPartial<T> = {
  [K in keyof T]?: T[K];
};

// Required — inverso de Partial
type MyRequired<T> = {
  [K in keyof T]-?: T[K];  // el - elimina el modificador ?
};

// Readonly
type MyReadonly<T> = {
  readonly [K in keyof T]: T[K];
};

// Pick — selecciona propiedades
type MyPick<T, K extends keyof T> = {
  [P in K]: T[P];
};

// Omit — excluye propiedades
type MyOmit<T, K extends keyof T> = MyPick<T, Exclude<keyof T, K>>;

// Record — mapa tipado
type MyRecord<K extends string | number | symbol, V> = {
  [P in K]: V;
};

// Exclude — filtra tipos de una union
type MyExclude<T, U> = T extends U ? never : T;
type Solo = Exclude<string | number | boolean, boolean>; // string | number

// Extract — lo opuesto de Exclude
type MyExtract<T, U> = T extends U ? T : never;
type SoloBool = Extract<string | number | boolean, boolean>; // boolean
```

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - Los generics permiten escribir código reutilizable sin perder la información de tipo. La diferencia con `any`: el compilador recuerda qué tipo usaste.
> - `extends` en un generic es un constraint — limita qué tipos son válidos. `T extends { length: number }` significa "T debe tener al menos la propiedad length de tipo number".
> - `keyof T` retorna un union de las claves de T. `T[K]` retorna el tipo de la propiedad K de T. Combinados son la herramienta más potente para trabajar con objetos de forma type-safe.
> - Los conditional types (`T extends U ? X : Y`) permiten tipos que cambian según otras condiciones.
> - Los utility types built-in (Partial, Pick, Omit, Record...) están implementados con mapped types y conditional types — los puedes recrear y crear los tuyos.

### Para llevar a la práctica
- [ ] Implementa `DeepPartial<T>` que hace Partial recursivamente en objetos anidados
- [ ] Crea `NonNullableKeys<T>` que retorna un union de las claves de T cuyo valor no es null/undefined
- [ ] Escribe una función `groupBy<T, K extends keyof T>(items: T[], key: K): Record<string, T[]>`

### Recursos
- 🌐 typescriptlang.org/docs/handbook/2/generics.html
- 🌐 totaltypescript.com/tutorials/typescript-generics — el mejor recurso de generics
- 📖 *Programming TypeScript* — Capítulo 4 (funciones y generics)
- 🌐 typescriptlang.org/docs/handbook/utility-types.html

---
`#typescript` `#generics` `#utility-types` `#conditional-types`
