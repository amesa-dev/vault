# 📘 07 — Tipos Avanzados

[[Desarrollo Profesional/TypeScript/TypeScript|⬅️ Volver al índice]] | [[Desarrollo Profesional/TypeScript/06 - Narrowing y Type Guards|← 06]] | [[Desarrollo Profesional/TypeScript/08 - Módulos y Declaraciones|08 →]]

> [!abstract] Introducción
> El sistema de tipos avanzados de TypeScript es Turing-completo — en teoría puedes computar cualquier cosa en el nivel de tipos. En la práctica, los mapped types, los conditional types y los template literal types te permiten expresar transformaciones de tipos que hacen el sistema de tipos semántico: no solo describe la forma de los datos, sino las reglas sobre cómo pueden transformarse.

## ¿De qué vamos a cubrir?

Los mecanismos de tipos avanzados de TypeScript: mapped types, conditional types con `infer`, template literal types y los patrones que los combinan.

### Conceptos que vamos a cubrir
- Mapped types: transformar todos los campos de un tipo
- Conditional types avanzados con `infer`
- Template literal types
- Recursive types
- Variadic tuple types

---

## El Concepto

### Mapped Types — Transformar Tipos

Un mapped type itera sobre las claves de un tipo y aplica una transformación:

```typescript
// Sintaxis base
type MiMappedType<T> = {
  [K in keyof T]: T[K];  // identidad — retorna el mismo tipo
};

// Añadir modificadores
type Opcional<T> = { [K in keyof T]?: T[K] };        // como Partial
type SoloLectura<T> = { readonly [K in keyof T]: T[K] };
type Obligatorio<T> = { [K in keyof T]-?: T[K] };    // elimina el ?
type Mutable<T> = { -readonly [K in keyof T]: T[K] }; // elimina readonly

// Transformar el tipo del valor
type Nullable<T> = { [K in keyof T]: T[K] | null };
type Promisify<T> = { [K in keyof T]: Promise<T[K]> };

// Remapear claves con 'as'
type WithGetters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};

interface Persona { nombre: string; edad: number; }
type PersonaGetters = WithGetters<Persona>;
// { getNombre: () => string; getEdad: () => number; }

// Filtrar propiedades
type SoloStrings<T> = {
  [K in keyof T as T[K] extends string ? K : never]: T[K];
};

interface Mixto { nombre: string; edad: number; activo: boolean; email: string; }
type CamposString = SoloStrings<Mixto>;
// { nombre: string; email: string; }
```

### Conditional Types Avanzados con `infer`

`infer` captura parte de un tipo para usarlo en la condición:

```typescript
// Extraer el tipo de retorno de una función
type ReturnType<T> = T extends (...args: any[]) => infer R ? R : never;

function crearUsuario(): { id: number; nombre: string } { return {} as any; }
type Usuario = ReturnType<typeof crearUsuario>;
// { id: number; nombre: string }

// Extraer el tipo envuelto en una Promise
type Awaited<T> =
  T extends null | undefined ? T :
  T extends object & { then(onfulfilled: infer F, ...args: infer _): any } ?
    F extends (value: infer V, ...args: infer _) => any ? Awaited<V> : never :
  T;

// Simplificado:
type UnwrapPromise<T> = T extends Promise<infer V> ? V : T;
type Str = UnwrapPromise<Promise<string>>; // string
type Num = UnwrapPromise<number>;          // number (no es Promise)

// Extraer el tipo de los elementos de un array
type ElementoArray<T> = T extends (infer E)[] ? E : never;
type Elemento = ElementoArray<string[]>; // string

// Extraer parámetros
type PrimerParametro<T extends (...args: any) => any> =
  T extends (first: infer P, ...rest: any[]) => any ? P : never;

function fn(x: number, y: string): boolean { return true; }
type Primero = PrimerParametro<typeof fn>; // number

// Conditional types distributivos sobre unions
type ToArray<T> = T extends any ? T[] : never;
type Resultado = ToArray<string | number>; // string[] | number[]
// (No string | number)[] — se distribuye sobre la union
```

### Template Literal Types

Los template literal types permiten construir tipos de string combinando otros tipos:

```typescript
// Combinación básica
type Saludo = `Hola, ${string}!`;
const s1: Saludo = "Hola, Andrés!"; // ✅
const s2: Saludo = "Hola, mundo!";  // ✅
// const s3: Saludo = "Adiós!";     // Error

// Con unions — genera todas las combinaciones
type Color     = "rojo" | "verde" | "azul";
type Intensidad = "claro" | "oscuro";
type ColorTono = `${Color}-${Intensidad}`;
// "rojo-claro" | "rojo-oscuro" | "verde-claro" | "verde-oscuro" | "azul-claro" | "azul-oscuro"

// Caso de uso real: eventos tipados
type EventoNombre = "click" | "hover" | "focus" | "blur";
type HandlerNombre = `on${Capitalize<EventoNombre>}`;
// "onClick" | "onHover" | "onFocus" | "onBlur"

// Rutas de API tipadas
type Metodo = "GET" | "POST" | "PUT" | "DELETE";
type Version = "v1" | "v2";
type Recurso = "usuarios" | "pedidos" | "productos";
type RutaApi = `/${Version}/${Recurso}`;
// "/v1/usuarios" | "/v1/pedidos" | "/v2/usuarios" | ...

// Template literals con keyof
type EventosCampo<T> = {
  [K in keyof T as `onChange${Capitalize<string & K>}`]: (valor: T[K]) => void;
};

interface Formulario { nombre: string; email: string; edad: number; }
type HandlersFormulario = EventosCampo<Formulario>;
// {
//   onChangeNombre: (valor: string) => void;
//   onChangeEmail:  (valor: string) => void;
//   onChangeEdad:   (valor: number) => void;
// }
```

### Recursive Types

TypeScript permite tipos que se referencian a sí mismos:

```typescript
// Árbol binario
type ArbolBinario<T> = {
  valor: T;
  izquierda?: ArbolBinario<T>;
  derecha?: ArbolBinario<T>;
};

const arbol: ArbolBinario<number> = {
  valor: 1,
  izquierda: { valor: 2, izquierda: { valor: 4 } },
  derecha: { valor: 3 },
};

// JSON — el tipo más clásico de recursividad
type JSONValor =
  | string
  | number
  | boolean
  | null
  | JSONValor[]
  | { [key: string]: JSONValor };

// DeepPartial — Partial recursivo
type DeepPartial<T> = T extends object
  ? { [P in keyof T]?: DeepPartial<T[P]> }
  : T;

interface Config {
  db: { host: string; puerto: number; credenciales: { user: string; pass: string } };
  api: { timeout: number; retries: number };
}

type ConfigParcial = DeepPartial<Config>;
// Todas las propiedades a cualquier nivel son opcionales
const config: ConfigParcial = { db: { host: "localhost" } }; // ✅
```

### Variadic Tuple Types

Los variadic tuples permiten componer tuples de forma genérica:

```typescript
// El tipo de una función que concatena tuples
function concat<T extends unknown[], U extends unknown[]>(
  a: [...T],
  b: [...U]
): [...T, ...U] {
  return [...a, ...b];
}

const resultado = concat([1, 2], ["a", "b"]); // [number, number, string, string]

// Prepend/Append a tuples
type Prepend<T, Tuple extends unknown[]> = [T, ...Tuple];
type Append<Tuple extends unknown[], T> = [...Tuple, T];

type ConPrimero = Prepend<string, [number, boolean]>; // [string, number, boolean]
type ConUltimo  = Append<[string, number], boolean>;  // [string, number, boolean]

// Tail de un tuple
type Tail<T extends unknown[]> = T extends [unknown, ...infer Rest] ? Rest : [];
type SinPrimero = Tail<[string, number, boolean]>; // [number, boolean]
```

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - Los **mapped types** iteran sobre las claves de un tipo y transforman cada propiedad. Combinados con `as` permiten filtrar y remapear claves.
> - `infer` en conditional types captura parte de un tipo para reutilizarlo: `T extends Promise<infer V> ? V : never` extrae el tipo dentro de la Promise.
> - Los **template literal types** permiten construir tipos de string combinando otros tipos, incluyendo unions. Una union en el template genera todas las combinaciones posibles.
> - Los **recursive types** permiten modelar estructuras de datos arbitrariamente profundas: árboles, JSON, configs anidadas.
> - Los **variadic tuple types** (`[...T, ...U]`) permiten componer tuples de forma genérica y type-safe.

### Para llevar a la práctica
- [ ] Implementa `DeepReadonly<T>` que aplica `readonly` recursivamente a objetos anidados
- [ ] Crea un tipo `EventMap` que genere todos los handlers a partir de un objeto de tipos de evento
- [ ] Escribe `Flatten<T>` que aplana un array anidado: `Flatten<[1, [2, 3], [4, [5]]]>` → `[1, 2, 3, 4, 5]`

### Recursos
- 🌐 typescriptlang.org/docs/handbook/2/types-from-types.html
- 🌐 totaltypescript.com/advanced-patterns — patterns avanzados
- 📖 *Programming TypeScript* — Capítulo 6 (advanced types)
- 🌐 github.com/millsp/ts-toolbelt — librería de utility types avanzados

---
`#typescript` `#tipos-avanzados` `#mapped-types` `#conditional-types` `#template-literal`
