# 📘 06 — Narrowing y Type Guards

[[Desarrollo Profesional/TypeScript/TypeScript|⬅️ Volver al índice]] | [[Desarrollo Profesional/TypeScript/05 - Generics|← 05]] | [[Desarrollo Profesional/TypeScript/07 - Tipos Avanzados|07 →]]

> [!abstract] Introducción
> El narrowing es el proceso por el que TypeScript refina el tipo de una variable dentro de un bloque de código. Cuando tienes `string | number`, TypeScript necesita saber qué hacer en cada rama. El narrowing es el mecanismo que hace eso posible — y cuando está bien diseñado, el compilador te protege de todos los casos que no manejaste.

## ¿De qué vamos a cubrir?

Las técnicas de narrowing de TypeScript, desde las básicas (`typeof`, `instanceof`) hasta las avanzadas (discriminated unions, assertion functions) y cómo diseñar APIs que aprovechen el narrowing.

### Conceptos que vamos a cubrir
- `typeof` y `instanceof` guards
- `in` operator
- Discriminated unions — el patrón más poderoso
- Custom type guards con `is`
- Assertion functions
- Exhaustividad con `never`

---

## El Concepto

### Por Qué Narrowing Existe

```typescript
function formatear(valor: string | number): string {
  // Aquí, valor es string | number — no puedes llamar métodos de uno ni de otro
  // valor.toUpperCase(); // Error: no existe en number
  // valor.toFixed(2);    // Error: no existe en string

  if (typeof valor === "string") {
    // Aquí, TypeScript SABE que valor es string
    return valor.toUpperCase(); // ✅
  }

  // Aquí, TypeScript SABE que valor es number (string ya fue descartado)
  return valor.toFixed(2); // ✅
}
```

TypeScript analiza el flujo de control y refina el tipo en cada rama. Esto se llama **control flow analysis**.

### `typeof` — Para Primitivos

```typescript
type Primitivo = string | number | boolean | null | undefined;

function describir(val: Primitivo): string {
  if (val === null)               return "null";
  if (val === undefined)          return "undefined";
  if (typeof val === "string")    return `string: "${val}"`;
  if (typeof val === "number")    return `number: ${val}`;
  if (typeof val === "boolean")   return `boolean: ${val}`;

  // TypeScript sabe que llegamos aquí si todos los casos anteriores fallaron
  // Con strict types, este punto es never — el compilador verifica exhaustividad
  const _agotado: never = val;
  return _agotado;
}
```

`typeof` devuelve: `"string"`, `"number"`, `"boolean"`, `"object"`, `"undefined"`, `"function"`, `"symbol"`, `"bigint"`. **Trampa:** `typeof null === "object"` — siempre compara con `=== null` explícitamente antes de usar `typeof`.

### `instanceof` — Para Clases

```typescript
class HttpError extends Error {
  constructor(public statusCode: number, message: string) {
    super(message);
    this.name = "HttpError";
  }
}

class ValidationError extends Error {
  constructor(public campo: string, message: string) {
    super(message);
    this.name = "ValidationError";
  }
}

function manejarError(error: unknown): string {
  if (error instanceof HttpError) {
    return `HTTP ${error.statusCode}: ${error.message}`; // TypeScript sabe que es HttpError
  }
  if (error instanceof ValidationError) {
    return `Validación en '${error.campo}': ${error.message}`;
  }
  if (error instanceof Error) {
    return `Error genérico: ${error.message}`;
  }
  return `Desconocido: ${String(error)}`;
}
```

### `in` Operator — Para Propiedades

```typescript
interface Gato  { tipo: "gato"; maullar: () => string; }
interface Perro { tipo: "perro"; ladrar: () => string; }
type Animal = Gato | Perro;

function hacerSonido(animal: Animal): string {
  if ("maullar" in animal) {
    // TypeScript sabe que es Gato
    return animal.maullar();
  }
  return animal.ladrar();
}
```

### Discriminated Unions — El Patrón Más Poderoso

Un discriminated union tiene un campo literal común ("discriminant") que TypeScript usa para determinar exactamente qué variante es:

```typescript
// El campo "tipo" es el discriminant — cada variante tiene un valor literal único
type ResultadoOperacion =
  | { tipo: "exito"; valor: number; tiempo: number }
  | { tipo: "error"; codigo: string; mensaje: string }
  | { tipo: "pendiente"; progreso: number };

function manejarResultado(resultado: ResultadoOperacion): string {
  switch (resultado.tipo) {
    case "exito":
      // TypeScript sabe que resultado tiene valor y tiempo
      return `✅ Resultado: ${resultado.valor} (${resultado.tiempo}ms)`;
    case "error":
      // TypeScript sabe que resultado tiene codigo y mensaje
      return `❌ ${resultado.codigo}: ${resultado.mensaje}`;
    case "pendiente":
      return `⏳ Progreso: ${resultado.progreso}%`;
    // No necesitas default — TypeScript verifica exhaustividad
    // Si añades una nueva variante y olvidas manejarla, el compilador avisa
  }
}

// Ejemplo más completo: sistema de eventos
type EventoApp =
  | { tipo: "usuario:login";   userId: number; timestamp: Date }
  | { tipo: "usuario:logout";  userId: number; duracionSesion: number }
  | { tipo: "pedido:creado";   pedidoId: string; total: number }
  | { tipo: "pedido:enviado";  pedidoId: string; tracking: string }
  | { tipo: "error:critico";   mensaje: string; stack: string };

function procesarEvento(evento: EventoApp): void {
  switch (evento.tipo) {
    case "usuario:login":
      console.log(`Login: userId ${evento.userId}`);
      break;
    case "pedido:creado":
      console.log(`Pedido ${evento.pedidoId}: ${evento.total}€`);
      break;
    // Si no manejas todos los casos, TypeScript NO avisa por defecto en switch
    // Para forzar exhaustividad:
    default:
      const agotado: never = evento; // Error si hay casos sin manejar
  }
}
```

### Custom Type Guards con `is`

Cuando necesitas verificar la forma de un objeto, puedes escribir type guards personalizados:

```typescript
interface Usuario {
  id: number;
  nombre: string;
  email: string;
}

// Retorno tipo "predicado de tipo": valor is Usuario
function esUsuario(valor: unknown): valor is Usuario {
  return (
    typeof valor === "object" &&
    valor !== null &&
    "id" in valor &&
    "nombre" in valor &&
    "email" in valor &&
    typeof (valor as any).id     === "number" &&
    typeof (valor as any).nombre === "string" &&
    typeof (valor as any).email  === "string"
  );
}

function procesarRespuesta(data: unknown): string {
  if (esUsuario(data)) {
    // TypeScript sabe que data es Usuario dentro de este bloque
    return `Usuario: ${data.nombre} (${data.email})`;
  }
  return "Datos inválidos";
}

// Type guard para arrays
function esArrayDe<T>(
  arr: unknown,
  guard: (item: unknown) => item is T
): arr is T[] {
  return Array.isArray(arr) && arr.every(guard);
}

const esString = (val: unknown): val is string => typeof val === "string";
const data: unknown = ["a", "b", "c"];
if (esArrayDe(data, esString)) {
  data.forEach(s => console.log(s.toUpperCase())); // ✅ TypeScript sabe que son strings
}
```

### Assertion Functions

Una assertion function lanza un error si la condición no se cumple — y refina el tipo para el código que sigue:

```typescript
// El tipo de retorno "asserts condition" le dice a TS que si la función retorna,
// la condición es verdadera
function assert(condition: unknown, mensaje: string): asserts condition {
  if (!condition) throw new Error(mensaje);
}

function assertEsString(val: unknown): asserts val is string {
  if (typeof val !== "string") {
    throw new TypeError(`Se esperaba string, recibí ${typeof val}`);
  }
}

function procesarInput(input: unknown): void {
  assertEsString(input);
  // A partir de aquí, TypeScript sabe que input es string
  console.log(input.toUpperCase()); // ✅ sin necesidad de if/else
}

// Non-null assertion — cuando TÚ sabes que no es null (úsalo con cuidado)
const elemento = document.getElementById("app");
// elemento.style.color = "red"; // Error: elemento puede ser null

// Con assertion function:
assertDefinido(elemento, "Elemento #app no encontrado");
elemento.style.color = "red"; // ✅ — si llegamos aquí, no es null

// Con operador !: (non-null assertion — evitar cuando hay alternativa)
elemento!.style.color = "red"; // ✅ pero silencia el error sin verificar
```

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - TypeScript analiza el flujo de control y refina tipos en cada rama. `typeof`, `instanceof`, comparaciones con `===` y el operador `in` son los guards básicos.
> - Los **discriminated unions** son el patrón más potente: un campo literal común permite a TypeScript determinar exactamente qué variante tienes en cada rama del switch.
> - Los **custom type guards** (`valor is Tipo`) encapsulan verificaciones complejas en funciones reutilizables que el compilador entiende.
> - Las **assertion functions** (`asserts condition`) refinan el tipo para el código que sigue — si la función no lanza, TypeScript garantiza la condición.
> - El check de exhaustividad con `never` en el `default` es fundamental para mantener el código al día cuando añades nuevas variantes a un union.

### Para llevar a la práctica
- [ ] Modela un sistema de notificaciones con discriminated union: `Email | SMS | Push` y verifica exhaustividad
- [ ] Escribe un type guard `esRespuestaValida(data: unknown): data is ApiResponse` que verifique la forma del objeto
- [ ] Convierte un bloque `if (x !== null && x !== undefined)` en una assertion function reutilizable

### Recursos
- 🌐 typescriptlang.org/docs/handbook/2/narrowing.html
- 🌐 totaltypescript.com/discriminated-unions-are-a-devs-best-friend
- 📖 *Programming TypeScript* — Capítulo 6 (advanced types)

---
`#typescript` `#narrowing` `#type-guards` `#discriminated-unions`
