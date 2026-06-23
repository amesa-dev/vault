# 📘 09 — Patrones Avanzados

[[Desarrollo Profesional/TypeScript/TypeScript|⬅️ Volver al índice]] | [[Desarrollo Profesional/TypeScript/08 - Módulos y Declaraciones|← 08]] | [[Desarrollo Profesional/TypeScript/10 - Configuración y Tooling|10 →]]

> [!abstract] Introducción
> El sistema de tipos de TypeScript es suficientemente expresivo para modelar patrones de diseño con garantías estáticas que otros lenguajes no pueden dar. Los branded types, los opaque types y los builder patterns tipados permiten llevar la corrección de código del runtime al tiempo de compilación — errores que antes solo se descubrían en producción ahora los detecta el compilador antes de ejecutar una línea.

## ¿De qué vamos a cubrir?

Patrones avanzados que usan el sistema de tipos de TypeScript para crear APIs más seguras y expresivas: branded types para evitar confusión de primitivos, builder pattern tipado, y varianza.

### Conceptos que vamos a cubrir
- Branded/Nominal types: distinguir primitivos semánticamente distintos
- El patrón Builder tipado
- Readonly patterns avanzados
- Variance: covarianza y contravarianza
- Assertion types y `satisfies`

---

## El Concepto

### Branded Types — El Problema de los Primitivos

El sistema de tipos estructural de TypeScript hace que `string` y `string` sean siempre compatibles, aunque semánticamente representen cosas distintas:

```typescript
// El problema: UserId y OrderId son ambos number — TypeScript no los distingue
function procesarPedido(userId: number, orderId: number): void {}

const userId = 1;
const orderId = 42;
procesarPedido(orderId, userId);  // ✅ TypeScript no detecta el error — pero es incorrecto
```

La solución son los **branded types**: tipos que parecen primitivos pero tienen una marca que los hace incompatibles entre sí:

```typescript
// Patrón de branding con tipo intersection
declare const brand: unique symbol;
type Brand<T, TBrand extends string> = T & { readonly [brand]: TBrand };

// Tipos brandados
type UserId  = Brand<number, "UserId">;
type OrderId = Brand<number, "OrderId">;
type Email   = Brand<string, "Email">;

// Funciones de creación (smart constructors)
function createUserId(id: number): UserId {
  if (id <= 0) throw new Error("UserId must be positive");
  return id as UserId;
}

function createOrderId(id: number): OrderId {
  return id as OrderId;
}

function createEmail(email: string): Email {
  if (!email.includes("@")) throw new Error("Invalid email");
  return email as Email;
}

// Ahora el compilador distingue los tipos
function procesarPedido(userId: UserId, orderId: OrderId): void {
  console.log(`Usuario ${userId}, Pedido ${orderId}`);
}

const uid = createUserId(1);
const oid = createOrderId(42);

procesarPedido(uid, oid);  // ✅
procesarPedido(oid, uid);  // Error: Argument of type 'OrderId' is not assignable to 'UserId'
procesarPedido(1, 42);     // Error: number no es UserId

// Validación en runtime + tipos en compile-time
type Validated<T extends string> = Brand<T, "Validated">;

function sanitizar(input: string): Validated<string> {
  const limpio = input.replace(/<[^>]*>/g, "").trim();
  return limpio as Validated<string>;
}

function renderHtml(content: Validated<string>): void {
  document.innerHTML = content; // sabemos que está sanitizado
}

renderHtml(sanitizar(input));   // ✅
renderHtml(input);              // Error — input no está validado
```

### Builder Pattern Tipado

El builder pattern en TypeScript puede verificar en tiempo de compilación que todos los campos requeridos se establecieron antes de construir el objeto:

```typescript
// Sistema de tipos que rastrea qué propiedades están configuradas
type BuilderState<T, Set extends keyof T = never> = {
  readonly set: Set;
  readonly data: Partial<T>;
};

// Un builder tipado que solo permite build() cuando todos los campos están
class QueryBuilder<
  TEntity,
  TRequired extends keyof TEntity,
  TSet extends keyof TEntity = never
> {
  private state: Partial<TEntity> = {};

  where<K extends keyof TEntity>(
    campo: K,
    valor: TEntity[K]
  ): QueryBuilder<TEntity, TRequired, TSet | K> {
    this.state[campo] = valor;
    return this as any;
  }

  // build() solo disponible cuando TSet incluye todos los TRequired
  build(
    this: QueryBuilder<TEntity, TRequired, TRequired>
  ): TEntity {
    return this.state as TEntity;
  }
}

// Versión más simple — builder que rastrea campos opcionales vs requeridos
interface PedidoParams {
  userId: number;
  productos: string[];
  descuento?: number;
  notas?: string;
}

class PedidoBuilder {
  private params: Partial<PedidoParams> = {};

  conUsuario(userId: number): this & { _hasUserId: true } {
    this.params.userId = userId;
    return this as any;
  }

  conProductos(productos: string[]): this & { _hasProductos: true } {
    this.params.productos = productos;
    return this as any;
  }

  conDescuento(descuento: number): this {
    this.params.descuento = descuento;
    return this;
  }

  // Solo disponible si tiene userId y productos
  build(
    this: PedidoBuilder & { _hasUserId: true } & { _hasProductos: true }
  ): PedidoParams {
    return this.params as PedidoParams;
  }
}

const pedido = new PedidoBuilder()
  .conUsuario(1)
  .conProductos(["prod-1", "prod-2"])
  .conDescuento(10)
  .build(); // ✅

const incompleto = new PedidoBuilder()
  .conUsuario(1)
  .build(); // Error: _hasProductos falta
```

### `satisfies` — Validar sin Perder Información

El operador `satisfies` (TypeScript 4.9+) verifica que un valor cumple un tipo sin ensancharlo:

```typescript
type Palette = { [key: string]: string | [number, number, number] };

// Con 'as' Palette — pierde información (TypeScript olvida los tipos específicos)
const colores = {
  rojo:  [255, 0, 0],
  verde: "#00FF00",
  azul:  [0, 0, 255],
} as Palette;

colores.rojo.toUpperCase(); // TypeScript permite esto — tipo es string | number[]

// Con satisfies — verifica sin ensanchar
const colores2 = {
  rojo:  [255, 0, 0],
  verde: "#00FF00",
  azul:  [0, 0, 255],
} satisfies Palette;

colores2.rojo.toUpperCase();  // Error: rojo es number[] — TypeScript recuerda
colores2.verde.toUpperCase(); // ✅ — verde es string
```

### Readonly Profundo y Mutabilidad Controlada

```typescript
// Deep readonly — TypeScript no incluye esto nativo, hay que construirlo
type DeepReadonly<T> = {
  readonly [K in keyof T]: T[K] extends object ? DeepReadonly<T[K]> : T[K];
};

interface Config {
  db: { host: string; puerto: number };
  api: { timeout: number };
}

const config: DeepReadonly<Config> = {
  db: { host: "localhost", puerto: 5432 },
  api: { timeout: 5000 },
};

config.db.host = "remoto";   // Error: readonly
config.db.puerto = 5433;     // Error: readonly

// Patrón: objeto mutable que se "congela" al retornarse como readonly
class ConfigBuilder {
  private config: Config = {
    db: { host: "localhost", puerto: 5432 },
    api: { timeout: 5000 },
  };

  setDb(host: string, puerto: number): this {
    this.config.db = { host, puerto };
    return this;
  }

  build(): Readonly<Config> {
    return Object.freeze(this.config) as Readonly<Config>;
  }
}
```

### Variance — Covarianza y Contravarianza

La varianza describe cómo se relacionan los tipos cuando están en posición de subtipo:

```typescript
// Covariante: si A extends B, entonces F<A> extends F<B>
// Arrays de TypeScript son covariantes (por compatibilidad con JS)
type Perro = { nombre: string; ladrar: () => void };
type Animal = { nombre: string };

let perros: Perro[] = [{ nombre: "Rex", ladrar: () => console.log("Guau") }];
let animales: Animal[] = perros; // ✅ — Perro[] es asignable a Animal[] (covariante)

// Esto puede causar problemas:
animales.push({ nombre: "Gato" }); // ✅ para TypeScript — Error en runtime conceptualmente

// Contravariante: si A extends B, entonces F<B> extends F<A)
// Las funciones son contravariantes en sus parámetros
type ManejarAnimal = (animal: Animal) => void;
type ManejarPerro  = (perro: Perro) => void;

function usar(handler: ManejarPerro): void {
  handler({ nombre: "Rex", ladrar: () => {} });
}

const handlerAnimal: ManejarAnimal = (animal) => console.log(animal.nombre);
usar(handlerAnimal); // ✅ — ManejarAnimal es subtipo de ManejarPerro (contravariante en parámetros)
// Una función que acepta Animal puede usarse donde se espera una que acepta Perro
// porque Perro tiene todo lo que tiene Animal

// TypeScript 4.7+ — varianza explícita
interface Provider<out T> {  // out = covariante
  get(): T;
}

interface Consumer<in T> {  // in = contravariante
  set(value: T): void;
}
```

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - Los **branded types** resuelven el problema del tipado estructural con primitivos: hacen incompatibles `UserId` y `OrderId` aunque ambos sean `number`.
> - El **builder pattern tipado** puede verificar en tiempo de compilación que todos los campos requeridos se establecieron antes de llamar `build()`.
> - `satisfies` valida que un valor cumple un tipo sin ensancharlo — TypeScript recuerda los tipos específicos de cada campo, no solo el tipo del contenedor.
> - La **varianza** explica por qué puedes pasar una función `(Animal) => void` donde se espera `(Perro) => void`: las funciones son contravariantes en sus parámetros.

### Para llevar a la práctica
- [ ] Crea tipos brandados para `ProductId`, `CategoryId`, `Price` y usa smart constructors con validación
- [ ] Implementa un builder para crear queries SQL tipadas: `.select()`, `.from()`, `.where()`, `.build()` donde `build()` requiere al menos `.from()` y `.select()`
- [ ] Reemplaza un `as Tipo` en tu código con `satisfies Tipo` y observa qué información ganas

### Recursos
- 📖 *Programming TypeScript* — Capítulo 6 (advanced types)
- 🌐 totaltypescript.com/branded-types — guía completa de branded types
- 🌐 typescriptlang.org/docs/handbook/2/variance-annotations.html
- 🌐 github.com/sindresorhus/type-fest — librería de utility types avanzados

---
`#typescript` `#patrones` `#branded-types` `#builder` `#varianza`
