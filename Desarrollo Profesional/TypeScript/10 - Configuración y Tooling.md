# 📘 10 — Configuración y Tooling

[[Desarrollo Profesional/TypeScript/TypeScript|⬅️ Volver al índice]] | [[Desarrollo Profesional/TypeScript/09 - Patrones Avanzados|← 09]]

> [!abstract] Introducción
> La configuración de TypeScript tiene un impacto enorme en la calidad del código que produce y en la velocidad del compilador. Un `tsconfig.json` mal configurado puede silenciar errores importantes o hacer que el compilador tarde minutos en proyectos medianos. Esta página cubre las opciones que realmente importan, los project references para monorepos y las herramientas del ecosistema moderno.

## ¿De qué vamos a cubrir?

El `tsconfig.json` en profundidad, los project references para proyectos grandes, y las herramientas del ecosistema TypeScript moderno: linters, formatters, bundlers y transpiladores.

### Conceptos que vamos a cubrir
- tsconfig.json en profundidad: las opciones que importan
- Project references para monorepos y proyectos grandes
- Configuraciones base: `@tsconfig/*`
- Bundlers y transpiladores modernos
- ESLint con TypeScript

---

## El Concepto

### tsconfig.json — Opciones que Importan

```json
{
  "compilerOptions": {
    // ── TARGET Y MODULE ──────────────────────────────────────────
    // A qué versión de JavaScript compila
    "target": "ES2022",   // usa ES2022 features nativamente

    // Sistema de módulos del output
    "module": "NodeNext", // para Node.js con ESM
    // "module": "ESNext", // para bundlers (Vite, webpack, etc.)
    // "module": "CommonJS", // para Node.js con CJS

    // Cómo TypeScript resuelve las importaciones
    "moduleResolution": "NodeNext",  // debe coincidir con module

    // ── RIGOR DEL SISTEMA DE TIPOS ──────────────────────────────
    "strict": true,              // activa todas las flags de strict

    // Lo que incluye strict:
    // "strictNullChecks": true, // null y undefined son tipos distintos
    // "strictFunctionTypes": true,  // verifica contravarianza
    // "strictPropertyInitialization": true,
    // "noImplicitAny": true,    // no permite any implícito
    // "noImplicitThis": true,
    // "alwaysStrict": true,     // añade "use strict" al output

    // Extras recomendados (no incluidos en strict)
    "noUncheckedIndexedAccess": true,  // arr[0] es T | undefined, no T
    "noImplicitReturns": true,         // todas las rutas deben retornar
    "noFallthroughCasesInSwitch": true,// switch sin break/return da error
    "exactOptionalPropertyTypes": true, // a?: T significa T, no T | undefined

    // ── OUTPUT ────────────────────────────────────────────────────
    "outDir": "./dist",
    "rootDir": "./src",
    "declaration": true,        // genera archivos .d.ts
    "declarationMap": true,     // source maps para .d.ts
    "sourceMap": true,          // source maps para .js
    "noEmitOnError": true,      // no genera output si hay errores de tipo

    // ── INTEROPERABILIDAD ─────────────────────────────────────────
    "esModuleInterop": true,    // permite import React from 'react'
    "allowSyntheticDefaultImports": true,

    // ── PERFORMANCE ──────────────────────────────────────────────
    "skipLibCheck": true,       // no verifica .d.ts de node_modules
    "incremental": true,        // caché incremental
    "tsBuildInfoFile": ".tsbuildinfo",

    // ── PATHS (imports absolutos) ────────────────────────────────
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"],
      "@domain/*": ["src/domain/*"]
    }
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "**/*.test.ts"]
}
```

### `noUncheckedIndexedAccess` — El Flag Más Ignorado

Esta opción hace que el acceso por índice a arrays y dicts retorne `T | undefined`:

```typescript
// Sin noUncheckedIndexedAccess
const arr = [1, 2, 3];
const x = arr[0]; // x es number — aunque arr podría estar vacío

// Con noUncheckedIndexedAccess
const x = arr[0]; // x es number | undefined
if (x !== undefined) {
  console.log(x * 2); // ✅
}

// También para acceso a dicts
const dict: Record<string, number> = {};
const val = dict["clave"]; // val es number | undefined con el flag
```

Es uno de los flags más conservadores y uno de los que más bugs captura en producción.

### Project References — Para Proyectos Grandes

Los project references permiten dividir proyectos grandes en subproyectos que se compilan independientemente:

```
mi-monorepo/
├── packages/
│   ├── domain/
│   │   ├── src/
│   │   └── tsconfig.json
│   ├── application/
│   │   ├── src/
│   │   └── tsconfig.json
│   └── infrastructure/
│       ├── src/
│       └── tsconfig.json
└── tsconfig.json  ← raíz
```

```json
// tsconfig.json raíz
{
  "references": [
    { "path": "./packages/domain" },
    { "path": "./packages/application" },
    { "path": "./packages/infrastructure" }
  ],
  "files": []  // no compila nada directamente
}

// packages/domain/tsconfig.json
{
  "compilerOptions": {
    "composite": true,        // requerido para project references
    "declaration": true,      // requerido para project references
    "outDir": "./dist"
  }
}

// packages/application/tsconfig.json
{
  "compilerOptions": {
    "composite": true,
    "outDir": "./dist"
  },
  "references": [
    { "path": "../domain" }   // depende de domain
  ]
}
```

```bash
# Compilar todo respetando dependencias
tsc -b  # o tsc --build

# Solo recompila lo que cambió
tsc -b --watch
```

**Ventajas:** Solo recompila los proyectos afectados por cambios. En proyectos grandes, esto reduce el tiempo de compilación de minutos a segundos.

### Configuraciones Base — `@tsconfig/*`

El paquete `@tsconfig` proporciona configuraciones base recomendadas:

```bash
npm install -D @tsconfig/node20
npm install -D @tsconfig/strictest
```

```json
{
  "extends": "@tsconfig/node20/tsconfig.json",
  "compilerOptions": {
    "outDir": "./dist"
    // Solo añades lo específico de tu proyecto
  }
}
```

### Bundlers y Transpiladores Modernos

```bash
# esbuild — el más rápido, ideal para desarrollo
npm install -D esbuild
npx esbuild src/index.ts --bundle --platform=node --outfile=dist/index.js

# tsx — ejecuta TS directamente (usa esbuild internamente)
npx tsx src/index.ts
npx tsx watch src/index.ts  # watch mode

# swc — alternativa en Rust, usado por Next.js y Vite
npm install -D @swc/core @swc/cli

# Vite — para proyectos de frontend
# Usa esbuild para dev, Rollup para producción
# TypeScript out-of-the-box, sin configuración extra
```

**Importante:** La mayoría de los bundlers modernos (esbuild, swc, Vite) **no verifican tipos** — solo transpilan. Para la verificación real de tipos, necesitas `tsc --noEmit` separadamente:

```json
// package.json
{
  "scripts": {
    "dev": "tsx watch src/index.ts",          // rápido, sin check de tipos
    "build": "npm run type-check && tsc",     // primero verifica, luego compila
    "type-check": "tsc --noEmit",             // solo verificación
    "ci": "npm run type-check && npm test"    // para CI
  }
}
```

### ESLint con TypeScript

La configuración moderna de ESLint para TypeScript:

```bash
npm install -D eslint typescript-eslint
```

```js
// eslint.config.js (ESLint 9 flat config)
import tseslint from "typescript-eslint";

export default tseslint.config(
  // Configuración base recomendada
  ...tseslint.configs.strictTypeChecked,
  // O para un proyecto menos estricto:
  // ...tseslint.configs.recommendedTypeChecked,
  {
    languageOptions: {
      parserOptions: {
        project: true,
        tsconfigRootDir: import.meta.dirname,
      },
    },
    rules: {
      // Reglas de TypeScript que añaden valor real
      "@typescript-eslint/no-explicit-any": "error",
      "@typescript-eslint/no-unused-vars": ["error", { argsIgnorePattern: "^_" }],
      "@typescript-eslint/prefer-nullish-coalescing": "error",  // ?? vs ||
      "@typescript-eslint/prefer-optional-chain": "error",      // a?.b vs a && a.b
      "@typescript-eslint/no-floating-promises": "error",       // await perdido
      "@typescript-eslint/await-thenable": "error",             // await de non-Promise
    },
  }
);
```

### Verificación en CI

```yaml
# .github/workflows/ci.yml
jobs:
  type-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: "20" }
      - run: npm ci
      - run: npm run type-check     # tsc --noEmit
      - run: npm run lint           # eslint
      - run: npm test               # jest/vitest
```

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - `"strict": true` es el mínimo. Añade `noUncheckedIndexedAccess`, `noImplicitReturns`, `exactOptionalPropertyTypes` para el nivel más alto de rigor.
> - Los project references reducen drásticamente los tiempos de compilación en proyectos grandes al compilar solo lo que cambió.
> - Los bundlers modernos (esbuild, swc, Vite) **no verifican tipos** — solo transpilan. Necesitas `tsc --noEmit` separado para la verificación real.
> - En CI, el pipeline debe incluir siempre: `type-check`, `lint`, `test`. En ese orden.
> - `@tsconfig/node20` o `@tsconfig/strictest` como base ahorran mantener la configuración manualmente.

### Para llevar a la práctica
- [ ] Añade `noUncheckedIndexedAccess: true` a tu tsconfig y corrige los errores que aparecen
- [ ] Configura `@typescript-eslint/no-floating-promises` y busca awaits olvidados en el código
- [ ] Separa la verificación de tipos (`tsc --noEmit`) de la compilación en tus scripts de npm

### Recursos
- 🌐 typescriptlang.org/tsconfig — referencia completa de opciones
- 🌐 typescript-eslint.io — documentación de typescript-eslint
- 🌐 github.com/tsconfig/bases — @tsconfig/* configuraciones base
- 📄 *TypeScript Project References* — typescriptlang.org/docs/handbook/project-references.html

---
`#typescript` `#tsconfig` `#tooling` `#eslint` `#ci`
