# 📘 03 — Funciones

[[Desarrollo Profesional/TypeScript/TypeScript|⬅️ Volver al índice]] | [[Desarrollo Profesional/TypeScript/02 - Interfaces y Type Aliases|← 02]] | [[Desarrollo Profesional/TypeScript/04 - Clases|04 →]]

> [!abstract] Introducción
> TypeScript añade una capa de tipos al sistema de funciones de JavaScript que permite describir no solo qué hace una función, sino exactamente qué tipos acepta y retorna. Esto habilita sobrecarga de funciones, funciones genéricas, y patrones avanzados de composición que el compilador puede verificar.

## ¿De qué vamos a cubrir?

El tipado completo de funciones en TypeScript: parámetros, retornos, opcionales, overloads, y el tipo `this`. Más los patrones funcionales que TypeScript hace posibles.

### Conceptos que vamos a cubrir
- Tipado de parámetros y retornos
- Parámetros opcionales, por defecto y rest
- Function overloads
- Funciones genéricas
- El tipo `this` y method signatures
- Tipos de funciones y composición

---

## El Concepto

### Tipado Básico de Funciones

```typescript
// Anotación explícita
function sumar(a: number, b: number): number {
  return a + b;
}

// Arrow function — TypeScript infiere el retorno si puede
const multiplicar = (a: number, b: number) => a * b;

// Cuando la inferencia no alcanza — anotas el retorno
const dividir = (a: number, b: number): number | never => {
  if (b === 0) throw new Error("División por cero");
  return a / b;
};

// Funciones que no retornan valor útil
function log(mensaje: string): void {
  console.log(mensaje);
}

// Funciones que NUNCA retornan (lanzan o loop infinito)
function lanzar(msg: string): never {
  throw new Error(msg);
}
```

### Parámetros Opcionales, Por Defecto y Rest

```typescript
// Parámetros opcionales — deben ir al final
function saludar(nombre: string, saludo?: string): string {
  return `${saludo ?? "Hola"}, ${nombre}!`;
}

saludar("Andrés");           // "Hola, Andrés!"
saludar("Andrés", "Buenos días"); // "Buenos días, Andrés!"

// Parámetros con valor por defecto — no necesitan el ?
function conectar(host: string, puerto: number = 5432): void {
  console.log(`Conectando a ${host}:${puerto}`);
}

// Rest parameters — todos los argumentos restantes como array
function sumarTodos(...numeros: number[]): number {
  return numeros.reduce((acc, n) => acc + n, 0);
}

sumarTodos(1, 2, 3, 4, 5); // 15

// Spread al llamar funciones
const nums = [1, 2, 3] as const;
sumarTodos(...nums); // ✅
```

### Function Overloads — Múltiples Firmas

TypeScript permite declarar múltiples firmas para una misma función cuando el tipo de retorno depende del tipo de entrada:

```typescript
// Firma de overload 1
function formatear(valor: string): string;
// Firma de overload 2
function formatear(valor: number): string;
// Firma de overload 3
function formatear(valor: string[]): string[];
// Implementación — debe ser compatible con todas las firmas
function formatear(valor: string | number | string[]): string | string[] {
  if (Array.isArray(valor)) {
    return valor.map(v => v.toUpperCase());
  }
  if (typeof valor === "number") {
    return valor.toFixed(2);
  }
  return valor.toUpperCase();
}

// El compilador elige la firma correcta según el argumento
const a = formatear("hola");    // TypeScript sabe que retorna string
const b = formatear(3.14);      // TypeScript sabe que retorna string
const c = formatear(["a","b"]); // TypeScript sabe que retorna string[]
```

**Cuándo usar overloads:** Cuando el tipo de retorno cambia según el tipo del argumento y quieres que el compilador lo sepa. Si el retorno siempre es el mismo, usa un union type en el parámetro.

### Funciones Genéricas

Las funciones genéricas trabajan con tipos que se determinan en el momento de llamarlas:

```typescript
// Función identidad genérica
function identidad<T>(valor: T): T {
  return valor;
}

identidad(42);        // TypeScript infiere T = number
identidad("hola");   // TypeScript infiere T = string
identidad<boolean>(true); // T = boolean explícito

// Caso de uso real: función de transformación tipada
function mapear<T, R>(array: T[], fn: (item: T) => R): R[] {
  return array.map(fn);
}

const longitudes = mapear(["hola", "mundo"], s => s.length); // number[]
const numeros    = mapear([1, 2, 3], n => n * 2);             // number[]

// Constraints — T debe tener cierta forma
function obtenerPropiedad<T, K extends keyof T>(obj: T, clave: K): T[K] {
  return obj[clave];
}

const usuario = { nombre: "Andrés", edad: 30 };
const nombre = obtenerPropiedad(usuario, "nombre");  // string — TypeScript lo sabe
const edad   = obtenerPropiedad(usuario, "edad");    // number
// obtenerPropiedad(usuario, "email");  // Error: "email" no es keyof typeof usuario

// Multiple type parameters
function merge<T extends object, U extends object>(obj1: T, obj2: U): T & U {
  return { ...obj1, ...obj2 };
}

const resultado = merge({ nombre: "Andrés" }, { edad: 30 });
// { nombre: string; edad: number }
```

### El Tipo `this` en Funciones

TypeScript puede tipar el contexto `this` de una función — algo que JavaScript no puede hacer:

```typescript
interface Contador {
  valor: number;
  incrementar(this: Contador, cantidad?: number): void;
}

const contador: Contador = {
  valor: 0,
  incrementar(this: Contador, cantidad = 1) {
    this.valor += cantidad;
  },
};

contador.incrementar();     // ✅
contador.incrementar(5);    // ✅

// Sin el tipo this, TypeScript no sabría que this.valor es number
const fn = contador.incrementar;
fn(); // Error: The 'this' context of type 'void' is not assignable to type 'Contador'
```

### Tipos de Funciones

En TypeScript las funciones son ciudadanos de primera clase — puedes tiparlas como cualquier otro valor:

```typescript
// Tipo función con type alias
type Transformador<T> = (valor: T) => T;
type Predicado<T> = (item: T) => boolean;
type Callback = (error: Error | null, result?: string) => void;

// Interface para funciones (menos común pero útil con overloads en interfaz)
interface Comparador<T> {
  (a: T, b: T): number;
  descripcion: string;  // interfaces de función pueden tener propiedades
}

// Higher-order functions bien tipadas
function componer<A, B, C>(
  f: (a: A) => B,
  g: (b: B) => C
): (a: A) => C {
  return (a) => g(f(a));
}

const doble    = (n: number) => n * 2;
const aString  = (n: number) => n.toString();
const dobleStr = componer(doble, aString);
dobleStr(5);  // "10"

// Memoización tipada
function memoize<Args extends unknown[], R>(
  fn: (...args: Args) => R
): (...args: Args) => R {
  const cache = new Map<string, R>();
  return (...args) => {
    const key = JSON.stringify(args);
    if (cache.has(key)) return cache.get(key)!;
    const result = fn(...args);
    cache.set(key, result);
    return result;
  };
}

const fibMemo = memoize((n: number): number => {
  if (n <= 1) return n;
  return fibMemo(n - 1) + fibMemo(n - 2);
});
```

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - Anota los tipos de parámetros siempre; el retorno solo cuando TypeScript no puede inferirlo correctamente.
> - Los parámetros opcionales (`?`) deben ir al final. Los parámetros con valor por defecto no necesitan `?`.
> - Los overloads permiten que el tipo de retorno dependa del tipo del argumento — el compilador elige la firma correcta.
> - Las funciones genéricas (`<T>`) permiten reutilizar lógica con múltiples tipos de forma type-safe. Los constraints (`extends`) limitan qué tipos son válidos.
> - Puedes tipar el `this` de una función con un primer parámetro especial `this: TipoContext`. TypeScript lo borra en la compilación.

### Para llevar a la práctica
- [ ] Escribe una función genérica `agrupar<T>(items: T[], key: keyof T): Record<string, T[]>`
- [ ] Implementa overloads para una función `buscar` que retorna `T` si pasa un id y `T[]` si pasa un string
- [ ] Crea un tipo `Middleware<T>` que represente `(req: T, next: () => void) => void`

### Recursos
- 🌐 typescriptlang.org/docs/handbook/2/functions.html
- 📖 *Programming TypeScript* — Capítulo 4 (funciones completo)
- 🌐 totaltypescript.com/tutorials/typescript-generics — ejercicios de generics

---
`#typescript` `#funciones` `#generics` `#overloads`
