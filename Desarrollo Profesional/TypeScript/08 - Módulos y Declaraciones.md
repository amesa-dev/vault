# 📘 08 — Módulos y Declaraciones

[[Desarrollo Profesional/TypeScript/TypeScript|⬅️ Volver al índice]] | [[Desarrollo Profesional/TypeScript/07 - Tipos Avanzados|← 07]] | [[Desarrollo Profesional/TypeScript/09 - Patrones Avanzados|09 →]]

> [!abstract] Introducción
> El sistema de módulos de TypeScript está construido sobre el sistema de módulos de JavaScript — con la complejidad añadida de tener que manejar archivos de declaración de tipos (`.d.ts`) para código JavaScript sin tipos. Entender bien ESM vs CommonJS, cómo funcionan los archivos `.d.ts` y cómo extender los tipos de librerías externas es esencial para trabajar en proyectos reales.

## ¿De qué vamos a cubrir?

El sistema de módulos completo de TypeScript: ESM vs CJS, archivos de declaración, los paquetes `@types`, y los mecanismos para extender tipos de terceros.

### Conceptos que vamos a cubrir
- ESM vs CommonJS en TypeScript
- Archivos `.d.ts`: declaraciones de tipo sin implementación
- `@types/*`: DefinitelyTyped
- Module augmentation: extender tipos de librerías
- Ambient declarations y `declare`
- `tsconfig` paths para imports absolutos

---

## El Concepto

### ESM vs CommonJS en TypeScript

TypeScript soporta ambos sistemas de módulos. La configuración de `tsconfig` determina cuál se usa:

```typescript
// ESM (ECMAScript Modules) — el estándar moderno
import { funcion } from "./modulo";
import tipo, { OtroTipo } from "./tipos";
export const valor = 42;
export default function main() {}
export type { MiTipo };  // export solo de tipo — no genera código JS

// CommonJS — el sistema de Node.js original
const { funcion } = require("./modulo");
module.exports = { funcion };
```

```json
// tsconfig.json para Node.js con ESM
{
  "compilerOptions": {
    "module": "NodeNext",
    "moduleResolution": "NodeNext"
  }
}

// package.json — declarar que el proyecto usa ESM
{
  "type": "module"
}
```

**La trampa de los imports en ESM con TypeScript:**

```typescript
// En ESM con Node.js, los imports deben incluir la extensión .js
// (aunque el archivo fuente sea .ts)
import { funcion } from "./modulo.js";  // ✅ — TypeScript resuelve ./modulo.ts
// import { funcion } from "./modulo";  // ❌ en ESM estricto

// Con "moduleResolution": "NodeNext", TypeScript exige la extensión en ESM
```

### Archivos `.d.ts` — Declaraciones de Tipo

Los archivos `.d.ts` describen la forma de un módulo sin implementación. Son lo que hace que TypeScript sepa los tipos de librerías JavaScript:

```typescript
// mi-libreria.d.ts

// Declarar una función
export declare function formatear(valor: number, opciones?: FormateoOpciones): string;

// Declarar una interfaz
export interface FormateoOpciones {
  decimales?: number;
  separadorMiles?: string;
  simbolo?: string;
}

// Declarar una clase
export declare class Calculadora {
  constructor(precision: number);
  sumar(a: number, b: number): number;
  dividir(a: number, b: number): number | never;
}

// Declarar constantes
export declare const VERSION: string;
export declare const CONSTANTES: {
  readonly MAX_DECIMALES: number;
  readonly MONEDAS: string[];
};
```

TypeScript busca archivos `.d.ts` en:
1. El directorio del módulo (si `types` o `typings` están definidos en package.json)
2. En `@types/<nombre>` (DefinitelyTyped)
3. Según las rutas de `typeRoots` en tsconfig

### `@types/*` — DefinitelyTyped

Muchas librerías JavaScript populares no incluyen tipos — se publican por separado en `@types`:

```bash
npm install -D @types/node    # tipos de Node.js
npm install -D @types/express # tipos de Express
npm install -D @types/lodash  # tipos de lodash
```

Desde TypeScript 5, hay una forma más ergonómica con los "inlined types" — algunas librerías incluyen sus tipos directamente.

**`typeRoots` y `types` en tsconfig:**

```json
{
  "compilerOptions": {
    // Solo incluir estos paquetes @types (por defecto incluye todos)
    "types": ["node", "jest"],

    // Directorios donde buscar declaraciones de tipo
    "typeRoots": ["./node_modules/@types", "./types"]
  }
}
```

### Module Augmentation — Extender Tipos de Terceros

Para añadir propiedades a tipos existentes de librerías:

```typescript
// Extender Express Request para añadir campo 'usuario'
// En un archivo types/express.d.ts

import "express";
import { UsuarioAutenticado } from "../src/domain/usuario";

declare module "express-serve-static-core" {
  interface Request {
    usuario?: UsuarioAutenticado;
    sessionId?: string;
  }
}

// Ahora en cualquier parte de tu app:
app.get("/perfil", (req, res) => {
  if (req.usuario) {  // TypeScript sabe que usuario es UsuarioAutenticado | undefined
    res.json(req.usuario);
  }
});
```

```typescript
// Extender Window con propiedades globales custom
declare global {
  interface Window {
    analytics: Analytics;
    featureFlags: Record<string, boolean>;
  }
}

// Ahora puedes usar window.analytics sin TypeScript quejarse
window.analytics.track("pageview");
```

### `declare` — Ambient Declarations

`declare` declara la existencia de algo sin proporcionar implementación:

```typescript
// Declarar una variable global (definida en otro script o en el HTML)
declare const __DEV__: boolean;
declare const __VERSION__: string;
declare function gtag(command: string, ...args: unknown[]): void;

// Declarar un módulo que no tiene tipos
declare module "libreria-sin-tipos" {
  export function funcion(arg: string): number;
  export const constante: string;
}

// Declarar módulos por extensión (para importar archivos no-JS)
declare module "*.svg" {
  const content: string;
  export default content;
}

declare module "*.png" {
  const content: string;
  export default content;
}

declare module "*.css" {
  const styles: Record<string, string>;
  export default styles;
}

// Con estas declaraciones puedes importar:
import logo from "./logo.svg";  // TypeScript sabe que es string
import estilos from "./App.css";
```

### `tsconfig` Paths — Imports Absolutos

Los path aliases evitan el infierno de los imports relativos:

```json
// tsconfig.json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"],
      "@domain/*": ["src/domain/*"],
      "@infra/*": ["src/infrastructure/*"],
      "@utils/*": ["src/utils/*"]
    }
  }
}
```

```typescript
// Sin path aliases — frágil ante refactors
import { Usuario } from "../../../domain/usuario";
import { logger } from "../../../../utils/logger";

// Con path aliases — robusto y legible
import { Usuario } from "@domain/usuario";
import { logger } from "@utils/logger";
```

**Importante:** `tsconfig` paths solo afectan a TypeScript. Para que funcionen en ejecución, necesitas configurar el bundler también:

```bash
# Con ts-node
npm install -D tsconfig-paths
node -r tsconfig-paths/register dist/index.js

# Con Jest
npm install -D @swc/jest
# Configurar moduleNameMapper en jest.config.ts
```

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - Con `module: NodeNext`, los imports ESM deben incluir extensión `.js` (aunque el archivo fuente sea `.ts`).
> - Los archivos `.d.ts` describen tipos sin implementación — son el puente entre JavaScript sin tipos y TypeScript.
> - `@types/*` son los paquetes de tipos para librerías populares. Con TypeScript 5+ muchas librerías incluyen sus propios tipos.
> - El **module augmentation** (`declare module "..."`) permite extender tipos de librerías de terceros sin modificarlas.
> - Los **path aliases** en `tsconfig` simplifican imports pero requieren configuración adicional en el runtime/bundler para funcionar.

### Para llevar a la práctica
- [ ] Añade tipos a `req.usuario` en Express usando module augmentation en un archivo `types/express.d.ts`
- [ ] Configura path aliases `@/` → `src/` y asegúrate de que funcionan tanto en TypeScript como en Jest
- [ ] Escribe un archivo `.d.ts` para una librería JavaScript que uses sin tipos

### Recursos
- 🌐 typescriptlang.org/docs/handbook/modules.html
- 🌐 definitelytyped.org — repositorio de @types
- 🌐 typescriptlang.org/docs/handbook/declaration-files/introduction.html
- 📖 *Programming TypeScript* — Capítulo 10 (namespaces, modules, script files)

---
`#typescript` `#modulos` `#declaraciones` `#esm` `#tipos`
