# 📘 00 — Introducción y Setup

[[Desarrollo Profesional/TypeScript/TypeScript|⬅️ Volver al índice de TypeScript]]

> [!abstract] Introducción
> TypeScript es JavaScript con un sistema de tipos estático. Pero la frase se queda corta: el sistema de tipos de TypeScript es uno de los más expresivos que existen en cualquier lenguaje, capaz de modelar estructuras de datos complejas que JavaScript ni siquiera puede describir. Esta primera página establece qué es TypeScript, cómo funciona internamente y cómo montar un entorno de trabajo profesional.

## ¿De qué vamos a hablar?

Antes de escribir código TypeScript útil, hay que entender la relación entre TypeScript y JavaScript, cómo funciona el compilador y qué opciones del `tsconfig.json` importan de verdad.

### Conceptos que vamos a cubrir
- TypeScript como lenguaje de compilación: qué hace el compilador
- El sistema de tipos es estructural, no nominal
- tsconfig.json: las opciones que realmente importan
- ts-node y tsx: ejecución directa sin build step
- Herramientas del ecosistema moderno

---

## El Concepto

### TypeScript = JavaScript + Sistema de Tipos en Tiempo de Compilación

TypeScript no es un runtime. No existe un "motor TypeScript" ejecutando tu código en producción. Lo que existe es un **compilador** (`tsc`) que:

1. Lee código TypeScript (`.ts`)
2. Verifica los tipos — y falla si hay errores
3. Emite JavaScript (`.js`) con los tipos eliminados

```
código.ts → tsc (verificación de tipos) → código.js → Node.js / navegador
```

En producción ejecutas JavaScript. TypeScript solo existe en desarrollo. Esto tiene una implicación crítica: **los tipos de TypeScript no existen en runtime**. No puedes usar `instanceof MiTipo` con un type alias en tiempo de ejecución — el tipo ha sido borrado.

### Tipado Estructural vs Nominal

La mayoría de lenguajes con tipado estático (Java, C#, Swift) usan tipado **nominal**: dos tipos son iguales si tienen el mismo nombre.

TypeScript usa tipado **estructural**: dos tipos son iguales si tienen la misma forma (las mismas propiedades con los mismos tipos).

```typescript
interface Punto {
  x: number;
  y: number;
}

// Este objeto es compatible con Punto aunque no lo declare explícitamente
const p = { x: 1, y: 2, z: 3 }; // tiene x e y — es un Punto válido

function mover(punto: Punto) {
  console.log(punto.x, punto.y);
}

mover(p); // ✅ válido — tipado estructural
```

Esto es poderoso y a veces sorprendente. Vuélvete a leer este ejemplo cuando te encuentres con comportamientos inesperados de compatibilidad de tipos.

### Instalación y Setup Básico

```bash
# Instalar TypeScript globalmente (para usar tsc)
npm install -g typescript

# O como dependencia de desarrollo del proyecto (recomendado)
npm install -D typescript

# Verificar versión
tsc --version

# Inicializar tsconfig.json
tsc --init
```

### tsconfig.json — Las Opciones que Importan

El `tsconfig.json` controla cómo compila TypeScript. Hay decenas de opciones, pero estas son las que más impacto tienen:

```json
{
  "compilerOptions": {
    // Target: a qué versión de JavaScript compila
    "target": "ES2022",

    // Module system: qué sistema de módulos usa el output
    "module": "NodeNext",     // para Node.js moderno con ESM
    "moduleResolution": "NodeNext",

    // Directorio de salida
    "outDir": "./dist",
    "rootDir": "./src",

    // Rigor del sistema de tipos — activa esto siempre
    "strict": true,           // activa todas las comprobaciones estrictas

    // Control de null/undefined
    "strictNullChecks": true, // incluido en strict

    // Interoperabilidad con CommonJS
    "esModuleInterop": true,

    // Source maps para debugging
    "sourceMap": true,

    // Evita la emisión de archivos si hay errores de tipo
    "noEmitOnError": true,

    // Sin archivos .js implícitos como "any"
    "noImplicitAny": true,    // incluido en strict
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

> `"strict": true` activa 8 comprobaciones estrictas a la vez. **Siempre activarlo en proyectos nuevos.**

### ts-node y tsx: Sin Build Step

Para desarrollo y scripts, compilar con `tsc` cada vez es lento. `ts-node` y `tsx` ejecutan TypeScript directamente:

```bash
# ts-node: la opción clásica
npm install -D ts-node
npx ts-node src/main.ts

# tsx: alternativa moderna (más rápida, basada en esbuild)
npm install -D tsx
npx tsx src/main.ts

# Ejecutar con watch mode (re-ejecuta al guardar)
npx tsx watch src/main.ts
```

### Proyecto Típico con Node.js

```
mi-proyecto/
├── src/
│   ├── index.ts
│   ├── types/
│   │   └── index.ts        # tipos compartidos
│   └── utils/
│       └── helpers.ts
├── tests/
│   └── helpers.test.ts
├── dist/                   # output compilado (en .gitignore)
├── tsconfig.json
├── package.json
└── .gitignore
```

```json
// package.json
{
  "scripts": {
    "build": "tsc",
    "dev": "tsx watch src/index.ts",
    "start": "node dist/index.js",
    "type-check": "tsc --noEmit"
  }
}
```

`tsc --noEmit` verifica tipos sin generar archivos — útil en CI para comprobar que no hay errores de tipo sin producir output.

### ESLint con TypeScript

```bash
npm install -D eslint @typescript-eslint/parser @typescript-eslint/eslint-plugin

# O usar el nuevo flat config de ESLint 9
npm install -D eslint typescript-eslint
```

```js
// eslint.config.js (ESLint 9 flat config)
import tseslint from 'typescript-eslint';

export default tseslint.config(
  ...tseslint.configs.strictTypeChecked,
  {
    languageOptions: {
      parserOptions: {
        project: true,
        tsconfigRootDir: import.meta.dirname,
      },
    },
  }
);
```

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - TypeScript es un compilador, no un runtime. Los tipos se borran en el output. En producción siempre ejecutas JavaScript.
> - El sistema de tipos es estructural: lo que importa es la forma (propiedades y sus tipos), no el nombre del tipo.
> - `"strict": true` en tsconfig es obligatorio en proyectos nuevos. Sin él, pierdes la mitad del valor de TypeScript.
> - `tsx` es la forma más rápida de ejecutar TS directamente en desarrollo. `tsc --noEmit` para verificar tipos en CI.

### Para llevar a la práctica
- [ ] Crea un proyecto con `npm init -y` y `npm install -D typescript`
- [ ] Ejecuta `npx tsc --init` y activa `"strict": true` en el tsconfig
- [ ] Instala `tsx` y verifica que puedes ejecutar un `.ts` directamente
- [ ] Añade el script `"type-check": "tsc --noEmit"` al package.json

### Recursos
- 🌐 typescriptlang.org/tsconfig — referencia completa de opciones de tsconfig
- 🌐 *TypeScript Deep Dive* — Basarat Ali Syed (gratis online)
- 📖 *Programming TypeScript* — Boris Cherny (capítulo 1-2 para el setup)
- 🌐 totaltypescript.com — Matt Pocock, el mejor recurso avanzado

---
`#typescript` `#setup` `#compilador` `#tsconfig`
