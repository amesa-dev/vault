# 📘 04 — Clases

[[Desarrollo Profesional/TypeScript/TypeScript|⬅️ Volver al índice]] | [[Desarrollo Profesional/TypeScript/03 - Funciones|← 03]] | [[Desarrollo Profesional/TypeScript/05 - Generics|05 →]]

> [!abstract] Introducción
> TypeScript añade al sistema de clases de JavaScript modificadores de acceso, propiedades readonly, clases abstractas y la verificación estática de que una clase implementa una interfaz. El resultado es un sistema de POO más próximo a Java o C# que al JavaScript nativo — con la ventaja de que todo desaparece en el output JavaScript.

## ¿De qué vamos a cubrir?

El sistema de clases de TypeScript: modificadores de acceso, clases abstractas, el sistema de herencia y cómo las clases interactúan con el sistema de tipos estructural de TS.

### Conceptos que vamos a cubrir
- Propiedades y modificadores de acceso
- Constructor shorthand
- Readonly y static
- Herencia y override
- Clases abstractas
- Implements vs extends
- Clases y el sistema de tipos estructural

---

## El Concepto

### Propiedades y Modificadores de Acceso

```typescript
class Persona {
  // Propiedades de clase — deben declararse antes del constructor
  public nombre: string;     // accesible desde cualquier sitio (default)
  private _edad: number;     // solo accesible dentro de esta clase
  protected email: string;   // accesible en esta clase y subclases
  readonly id: number;       // solo lectura — puede asignarse en constructor
  #salario: number;          // private "hard" de JS — NO en el prototipo

  constructor(nombre: string, edad: number, email: string, id: number) {
    this.nombre = nombre;
    this._edad  = edad;
    this.email  = email;
    this.id     = id;
    this.#salario = 0;
  }

  // Getter/setter
  get edad(): number {
    return this._edad;
  }

  set edad(valor: number) {
    if (valor < 0 || valor > 150) throw new Error("Edad inválida");
    this._edad = valor;
  }
}

const p = new Persona("Andrés", 30, "a@b.com", 1);
p.nombre = "Luis";      // ✅ public
// p._edad = 31;        // Error: private
// p.id = 2;            // Error: readonly
p.edad = 31;            // ✅ usa el setter
```

### Constructor Shorthand — Menos Boilerplate

TypeScript permite declarar y asignar propiedades directamente en el constructor:

```typescript
// Sin shorthand (verboso)
class ProductoSinShorthand {
  public nombre: string;
  private precio: number;
  readonly id: number;

  constructor(nombre: string, precio: number, id: number) {
    this.nombre = nombre;
    this.precio = precio;
    this.id     = id;
  }
}

// Con constructor shorthand (idiomático en TS)
class Producto {
  constructor(
    public nombre: string,
    private precio: number,
    readonly id: number,
    private readonly categoria: string = "general"  // con valor por defecto
  ) {}

  obtenerPrecio(): number {
    return this.precio;
  }

  toString(): string {
    return `${this.nombre} (${this.categoria}): ${this.precio}€`;
  }
}

const p = new Producto("Teclado", 89.99, 1);
console.log(p.nombre);       // "Teclado" — acceso a propiedad pública
// console.log(p.precio);   // Error: private
```

### Propiedades Static

```typescript
class Configuracion {
  private static instancia: Configuracion | null = null;
  private readonly valores: Map<string, string> = new Map();

  // Constructor privado para Singleton
  private constructor() {}

  // Método estático de fábrica
  static obtenerInstancia(): Configuracion {
    if (!Configuracion.instancia) {
      Configuracion.instancia = new Configuracion();
    }
    return Configuracion.instancia;
  }

  set(clave: string, valor: string): void {
    this.valores.set(clave, valor);
  }

  get(clave: string): string | undefined {
    return this.valores.get(clave);
  }
}

const config = Configuracion.obtenerInstancia();
config.set("db_host", "localhost");
```

### Herencia y Override

```typescript
class Animal {
  constructor(
    public nombre: string,
    protected edad: number
  ) {}

  hablar(): string {
    return `${this.nombre} hace un sonido`;
  }

  describir(): string {
    return `${this.nombre} tiene ${this.edad} años`;
  }
}

class Perro extends Animal {
  constructor(nombre: string, edad: number, public raza: string) {
    super(nombre, edad);  // OBLIGATORIO llamar a super primero
  }

  // override — keyword explícita (TS 4.3+)
  // Si el método padre se renombra, TypeScript avisa aquí
  override hablar(): string {
    return `${this.nombre} ladra: ¡Guau!`;
  }

  // Nuevo método que accede a protected del padre
  describir(): string {
    return `${super.describir()}, raza: ${this.raza}`;
  }
}

const p = new Perro("Rex", 3, "Pastor Alemán");
p.hablar();   // "Rex ladra: ¡Guau!"
p.describir(); // "Rex tiene 3 años, raza: Pastor Alemán"

// instanceof funciona con herencia
console.log(p instanceof Perro);   // true
console.log(p instanceof Animal);  // true
```

### Clases Abstractas

Las clases abstractas no se pueden instanciar directamente — son plantillas para subclases:

```typescript
abstract class Repositorio<T> {
  // Métodos abstractos — DEBEN implementarse en subclases
  abstract findById(id: number): Promise<T | null>;
  abstract save(entity: T): Promise<T>;
  abstract delete(id: number): Promise<void>;

  // Métodos concretos — disponibles en todas las subclases
  async findOrThrow(id: number): Promise<T> {
    const entity = await this.findById(id);
    if (!entity) throw new Error(`Entidad con id ${id} no encontrada`);
    return entity;
  }
}

interface Usuario {
  id: number;
  nombre: string;
  email: string;
}

class UsuarioRepositorio extends Repositorio<Usuario> {
  private usuarios: Usuario[] = [];

  async findById(id: number): Promise<Usuario | null> {
    return this.usuarios.find(u => u.id === id) ?? null;
  }

  async save(usuario: Usuario): Promise<Usuario> {
    const idx = this.usuarios.findIndex(u => u.id === usuario.id);
    if (idx >= 0) {
      this.usuarios[idx] = usuario;
    } else {
      this.usuarios.push(usuario);
    }
    return usuario;
  }

  async delete(id: number): Promise<void> {
    this.usuarios = this.usuarios.filter(u => u.id !== id);
  }
}

// new Repositorio<Usuario>(); // Error: Cannot create an instance of an abstract class
const repo = new UsuarioRepositorio();
```

### `implements` vs `extends`

```typescript
// extends: herencia — la clase hija ES-UN padre
// implements: contrato — la clase cumple una interfaz SIN herencia

interface Serializable {
  serializar(): string;
  static deserializar(data: string): Serializable; // No se puede en TS — limitación
}

interface Validable {
  validar(): boolean;
  errores(): string[];
}

// Una clase puede implementar múltiples interfaces
class Pedido implements Serializable, Validable {
  constructor(
    public id: number,
    public items: string[],
    public total: number
  ) {}

  serializar(): string {
    return JSON.stringify({ id: this.id, items: this.items, total: this.total });
  }

  validar(): boolean {
    return this.items.length > 0 && this.total > 0;
  }

  errores(): string[] {
    const errores: string[] = [];
    if (this.items.length === 0) errores.push("El pedido no tiene items");
    if (this.total <= 0)         errores.push("El total debe ser positivo");
    return errores;
  }
}
```

### Clases y Tipado Estructural

TypeScript usa tipado estructural — una clase es compatible con otra si tiene la misma forma:

```typescript
class Punto2D {
  constructor(public x: number, public y: number) {}
}

class Vector {
  constructor(public x: number, public y: number) {}
  longitud(): number { return Math.sqrt(this.x ** 2 + this.y ** 2); }
}

function dibujar(punto: Punto2D): void {
  console.log(punto.x, punto.y);
}

const v = new Vector(3, 4);
dibujar(v);  // ✅ — Vector es estructuralmente compatible con Punto2D

// Cuidado con los miembros private
class ConPrivado {
  private secreto: string = "x";
  publico: number = 1;
}

class OtroConPrivado {
  private secreto: string = "y";  // mismo nombre, pero TypeScript los trata como DISTINTOS
  publico: number = 1;
}

const a: ConPrivado = new OtroConPrivado();
// Error: types have private fields of the same name 'secreto'
// Los private de distintas clases NO son estructuralmente compatibles
```

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - Los modificadores de acceso (`public`, `private`, `protected`) son verificados en compilación — desaparecen en el JavaScript generado.
> - El constructor shorthand (`constructor(public x: number)`) elimina el boilerplate de declarar y asignar propiedades.
> - `override` (TS 4.3+) hace explícita la intención de sobrescribir — si el método padre desaparece, el compilador avisa.
> - Las clases abstractas definen plantillas con métodos obligatorios. `implements` aplica un contrato de interfaz sin herencia.
> - TypeScript usa tipado estructural en clases, con la excepción de los miembros `private` — que sí crean incompatibilidad nominal.

### Para llevar a la práctica
- [ ] Implementa un patrón de Repositorio genérico con clase abstracta: `Repositorio<T>` con `findAll`, `findById`, `save`, `delete`
- [ ] Crea una jerarquía de `Figura` abstracta con `area()` y `perimetro()`, implementada por `Circulo`, `Rectangulo` y `Triangulo`
- [ ] Usa `implements` para que una clase `Cache` cumpla la interfaz `Map<string, unknown>`

### Recursos
- 🌐 typescriptlang.org/docs/handbook/2/classes.html
- 📖 *Programming TypeScript* — Capítulo 5 (clases e interfaces)
- 📄 PEP sobre acceso privado en JS: tc39.es/proposal-class-fields

---
`#typescript` `#clases` `#poo` `#herencia` `#abstracto`
