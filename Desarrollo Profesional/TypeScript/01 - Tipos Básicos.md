# 📘 01 — Tipos Básicos

[[Desarrollo Profesional/TypeScript/TypeScript|⬅️ Volver al índice]] | [[Desarrollo Profesional/TypeScript/00 - Introducción y Setup|← 00]] | [[Desarrollo Profesional/TypeScript/02 - Interfaces y Type Aliases|02 →]]

> [!abstract] Introducción
> TypeScript extiende el sistema de tipos de JavaScript añadiendo primitivos, tipos especiales y herramientas para modelar la ausencia de valor. Entender estos tipos y sus diferencias (especialmente `any` vs `unknown` vs `never`) es fundamental para escribir código que realmente aprovecha el compilador.

## ¿De qué vamos a cubrir?

Los tipos que vienen con TypeScript sin importar nada externo: primitivos, arrays, tuples, enums y los tipos especiales que hacen al sistema de tipos de TS único.

### Conceptos que vamos a cubrir
- Primitivos: string, number, boolean, symbol, bigint
- Arrays y Tuples
- Enums: const enum vs enum regular
- Los tipos especiales: any, unknown, never, void, undefined, null
- El tipo `object` y sus trampas

---

## El Concepto

### Primitivos

Los mismos que JavaScript, con anotación de tipo:

```typescript
const nombre: string = "Andrés";
const edad: number = 30;
const activo: boolean = true;
const nada: undefined = undefined;
const vacio: null = null;
const id: symbol = Symbol("id");
const grande: bigint = 9007199254740991n;
```

En la práctica, TypeScript infiere el tipo si inicializas la variable. No necesitas anotar explícitamente:

```typescript
const nombre = "Andrés";  // TypeScript infiere string
const edad = 30;          // TypeScript infiere number (literal 30, más preciso)
```

> Regla de oro: **no anotes tipos que TypeScript puede inferir**. Solo añade anotaciones cuando añaden información que el compilador no puede deducir.

### Arrays y Tuples

```typescript
// Arrays — dos sintaxis equivalentes
const nums: number[] = [1, 2, 3];
const strs: Array<string> = ["a", "b", "c"];

// Arrays de solo lectura — no se pueden mutar
const fijos: readonly number[] = [1, 2, 3];
fijos.push(4); // Error: Property 'push' does not exist on type 'readonly number[]'

// Tuples — arrays de longitud y tipos fijos en cada posición
const punto: [number, number] = [10, 20];
const entrada: [string, number] = ["edad", 30];

// Tuples con labels (más legible)
const coordenada: [x: number, y: number] = [10, 20];

// Tuples opcionales y rest
type HttpResponse = [status: number, body: string, headers?: Record<string, string>];

// Tuples de longitud variable (útil para argumentos)
type Cabecera = [first: string, ...rest: number[]];
```

**Trampas de las tuples:**

```typescript
const punto: [number, number] = [10, 20];

// TypeScript permite push() en tuples por defecto
punto.push(30);      // Compilador no lo detecta — bug silencioso
console.log(punto);  // [10, 20, 30]

// Solución: usar readonly
const puntoFijo: readonly [number, number] = [10, 20];
puntoFijo.push(30);  // Error en compilación
```

### Enums

Los enums son uno de los pocos constructos de TypeScript que generan código JavaScript real:

```typescript
// Enum regular — genera código JS con un objeto
enum Direccion {
  Norte = "NORTE",
  Sur   = "SUR",
  Este  = "ESTE",
  Oeste = "OESTE",
}

function mover(dir: Direccion): void {
  console.log(`Moviéndose hacia ${dir}`);
}

mover(Direccion.Norte); // "Moviéndose hacia NORTE"
```

```typescript
// const enum — borrado en compilación, inlined como literales
// Más eficiente en runtime pero no funciona con module bundlers que no usan tsc
const enum HttpStatus {
  OK = 200,
  NotFound = 404,
  ServerError = 500,
}

const status: HttpStatus = HttpStatus.OK;
// Compila a: const status = 200;
```

**Alternativa a enums:** Union de string literals — más idiomático en TS moderno:

```typescript
// Más predecible que enums, más fácil de usar con JSON
type Direccion = "NORTE" | "SUR" | "ESTE" | "OESTE";

function mover(dir: Direccion): void {
  console.log(`Moviéndose hacia ${dir}`);
}

mover("NORTE"); // ✅
mover("ABAJO"); // Error: Argument of type '"ABAJO"' is not assignable to type 'Direccion'
```

### Los Tipos Especiales: any, unknown, never, void

Estos cuatro tipos son donde TypeScript difiere más de cualquier otro lenguaje:

#### `any` — La Trampa

`any` desactiva el sistema de tipos completamente. Es el tipo de escape hatch que destruye todas las garantías:

```typescript
let x: any = 5;
x.metodo();      // No error — TypeScript confía en ti ciegamente
x = "hola";     // No error
x.propiedad;    // No error — aunque no exista

// any se propaga: contamina los tipos que lo tocan
function unsafe(val: any): string {
  return val.trim(); // TypeScript no detecta que val podría no tener trim()
}
```

> `any` es el tipo más peligroso de TypeScript. Cada `any` es una renuncia a las garantías del compilador. Usa `noImplicitAny: true` en tsconfig para que TypeScript te avise cuando infiere `any`.

#### `unknown` — El `any` Seguro

`unknown` es el tipo que no sabes qué es, pero que te obliga a verificar antes de usar:

```typescript
function procesar(valor: unknown): string {
  // No puedes usar valor directamente
  // valor.trim();  // Error: Object is of type 'unknown'

  // Debes verificar el tipo primero
  if (typeof valor === "string") {
    return valor.trim(); // ✅ aquí TypeScript sabe que es string
  }

  if (typeof valor === "number") {
    return valor.toString(); // ✅
  }

  throw new Error(`Tipo no soportado: ${typeof valor}`);
}
```

**Regla:** Usa `unknown` en lugar de `any` cuando no sabes el tipo pero lo vas a verificar. Usa `any` solo cuando integras código sin tipos o cuando el sistema de tipos de TS no puede modelar lo que necesitas.

#### `never` — El Tipo Vacío

`never` representa valores que nunca ocurren. Aparece en dos contextos:

```typescript
// 1. Funciones que nunca retornan (lanzan excepciones o loops infinitos)
function lanzarError(mensaje: string): never {
  throw new Error(mensaje);
}

function loopInfinito(): never {
  while (true) {}
}

// 2. Exhaustividad en switch — el compilador verifica que cubriste todos los casos
type Forma = "círculo" | "cuadrado" | "triángulo";

function area(forma: Forma): number {
  switch (forma) {
    case "círculo":   return Math.PI * 5 ** 2;
    case "cuadrado":  return 5 ** 2;
    case "triángulo": return (5 * 5) / 2;
    default:
      // Si añades un nuevo tipo a Forma y se te olvida manejarlo aquí,
      // TypeScript te lo dirá con un error en esta línea
      const agotado: never = forma;
      throw new Error(`Forma no manejada: ${agotado}`);
  }
}
```

El truco del `never` en el `default` es uno de los patrones más valiosos de TypeScript para mantener el código exhaustivo.

#### `void` — Ausencia de Valor de Retorno

```typescript
// void = la función retorna undefined (sin valor útil)
function saludar(nombre: string): void {
  console.log(`Hola ${nombre}`);
  // no necesita return, o puede tener return sin valor
}

// Diferencia con undefined:
// void: el caller no debe usar el valor de retorno
// undefined: la función explícitamente retorna undefined como valor

const resultado: undefined = (() => undefined)();  // asignable a undefined
const x: void = saludar("test");  // asignable a void
```

### `object`, `Object` y `{}`

Tres cosas distintas que causan confusión:

```typescript
// object (minúscula): cualquier valor no primitivo
let obj: object = {};
let arr: object = [];      // arrays son objects
let fn: object = () => {}; // funciones son objects
// obj = 42;               // Error: number es primitivo

// Object (mayúscula): casi cualquier valor (incluidos primitivos box)
// Evitar — rara vez es lo que quieres

// {} (empty object type): cualquier valor no null/undefined
let x: {} = 42;     // ✅ permite primitivos
let y: {} = "hola"; // ✅
// let z: {} = null; // Error con strictNullChecks
```

En la práctica: usa tipos más específicos que `object`. Si necesitas "cualquier objeto con propiedades", usa `Record<string, unknown>`.

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - No anotas tipos que TypeScript puede inferir. Deja que infiera cuando inicializas variables.
> - Arrays: `number[]` o `Array<number>`. Tuples: longitud y tipo por posición. Usa `readonly` para prevenir mutación.
> - Enums generan código JS. La alternativa moderna son union de string literals: más simple y más predecible.
> - `any` desactiva el compilador — evítalo. `unknown` fuerza verificación antes de usar. `never` modela imposibilidad y exhaustividad.
> - `void` es "no me importa el valor de retorno". `never` es "esto no puede ocurrir".

### Para llevar a la práctica
- [ ] Crea un tipo `Forma = "círculo" | "cuadrado"` y una función `area()` con el check de exhaustividad con `never`
- [ ] Escribe una función que recibe `unknown` y la hace funcionar para string y number sin usar `any`
- [ ] Compara qué código genera un `enum` vs una union de string literals mirando el output de `tsc`

### Recursos
- 🌐 typescriptlang.org/docs/handbook/2/everyday-types.html
- 🌐 totaltypescript.com/tutorials/beginners-typescript — ejercicios interactivos
- 📖 *Programming TypeScript* — Capítulo 3 (todo el sistema de tipos básico)
- 🌐 *TypeScript Deep Dive* — capítulo "Types"

---
`#typescript` `#tipos` `#fundamentos` `#any` `#unknown` `#never`
