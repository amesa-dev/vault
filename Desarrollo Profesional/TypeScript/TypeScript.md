# 📘 TypeScript

[[Desarrollo Profesional/Inicio Profesional|⬅️ Volver a Desarrollo Profesional]]

> [!abstract] TypeScript
> TypeScript es un superconjunto tipado de JavaScript que se compila a JavaScript limpio. Añade tipos estáticos para facilitar el desarrollo a gran escala.

---

## 🔑 Conceptos Esenciales

### 1. Tipos Básicos
```typescript
let isDone: boolean = false;
let lines: number = 42;
let name: string = "Andrés";
let numbers: number[] = [1, 2, 3];
let genericList: Array<number> = [1, 2, 3]; // Sintaxis alternativa
```

### 2. Interfaces vs Types
A menudo surge la duda sobre cuándo usar cada uno. Aquí hay un resumen:

| Característica | Interface | Type Alias |
| --- | --- | --- |
| **Sintaxis** | `interface User { ... }` | `type User = { ... }` |
| **Extensible (Inheritance)** | Sí (usando `extends`) | Sí (usando intersecciones `&`) |
| **Declaración repetida** | Sí (hace *Declaration Merging*) | No (da error de duplicación) |

#### Ejemplo de Interface:
```typescript
interface Persona {
  nombre: string;
  edad?: number; // Propiedad opcional
  readonly id: number; // Propiedad de solo lectura
}
```

#### Ejemplo de Type Alias (Intersección y Unión):
```typescript
type ID = string | number; // Unión
type Empleado = Persona & {
  puesto: string;
};
```

---

## ⚡ Genéricos (Generics)
Permiten crear componentes reutilizables que funcionan con varios tipos en lugar de uno solo.

```typescript
function identidad<T>(arg: T): T {
  return arg;
}

let output = identidad<string>("miCadena");
```

---

## 🛠️ Tipos Utilitarios (Utility Types)
TypeScript proporciona varios tipos utilitarios globales para facilitar transformaciones comunes de tipos:

- `Partial<T>`: Convierte todas las propiedades de `T` en opcionales.
- `Required<T>`: Convierte todas las propiedades de `T` en obligatorias.
- `Readonly<T>`: Convierte todas las propiedades de `T` en solo lectura.
- `Record<K, T>`: Crea un mapa con claves `K` y valores `T`.

---
`#typescript` `#javascript` `#programacion` `#apuntes`
