# Configuration Files — Educational Reference

This document explains every configuration file in this monorepo: what it does, why it exists, and what each key setting means.

---

## Table of Contents

1. [Monorepo Overview](#monorepo-overview)
2. [Root-Level Files](#root-level-files)
   - [nx.json](#nxjson)
   - [package.json](#packagejson)
   - [tsconfig.base.json](#tsconfigbasejson)
   - [jest.config.ts](#jestconfigts)
   - [eslint.config.mjs](#eslintconfigmjs)
   - [.gitignore](#gitignore)
   - [docker-compose.yaml](#docker-composeyaml)
3. [apps/jobber-auth — App-Level Files](#appsjobber-auth--app-level-files)
   - [project.json](#projectjson)
   - [tsconfig.json](#tsconfigjson)
   - [tsconfig.app.json](#tsconfigappjson)
   - [tsconfig.spec.json](#tsconfigspecjson)
   - [jest.config.cts](#jestconfigcts)
   - [webpack.config.js](#webpackconfigjs)
   - [eslint.config.mjs (app)](#eslintconfigmjs-app)
   - [prisma.config.ts](#prismaconfigts)
   - [prisma/schema.prisma](#prismaschemaprisma)

---

## Monorepo Overview

This project is an **Nx monorepo** — a single git repository that holds multiple applications and libraries. Nx adds build orchestration, caching, and tooling on top of a standard Node.js workspace. Currently there is one application: `apps/jobber-auth`, a NestJS backend service with a PostgreSQL database via Prisma.

```
jobber/
├── apps/
│   └── jobber-auth/        ← NestJS application
├── nx.json                 ← Nx workspace config
├── package.json            ← Root dependencies & scripts
├── tsconfig.base.json      ← Shared TypeScript config
├── jest.config.ts          ← Root Jest config
├── eslint.config.mjs       ← Root ESLint config
├── docker-compose.yaml     ← Local dev database
└── .gitignore
```

---

## Root-Level Files

### nx.json

**Purpose:** Configures the Nx workspace — how it discovers projects, caches work, and what plugins are active.

```json
{
  "$schema": "./node_modules/nx/schemas/nx-schema.json",
  "namedInputs": { ... },
  "plugins": [ ... ]
}
```

**Key concepts:**

#### `namedInputs`
Named input sets tell Nx which files affect a given task's cache. If none of these files change, Nx can skip re-running the task and replay the cached result instead.

- **`default`** — includes everything under the project root plus `sharedGlobals`. Used as the base for most tasks.
- **`production`** — starts from `default` then *excludes* test files, ESLint configs, and jest configs. This means linting or test config changes don't invalidate a production build cache.
- **`sharedGlobals`** — points to `.github/workflows/ci.yml`. If the CI pipeline definition changes, all caches are busted, ensuring CI always re-runs tasks after a workflow update.

The `!` prefix means "exclude this pattern from the input set."

#### `plugins`
Nx plugins extend the workspace with auto-detected targets. Instead of manually configuring every `build`, `serve`, `test`, and `lint` target in every `project.json`, plugins discover and wire them up automatically.

- **`@nx/webpack/plugin`** — Scans for `webpack.config.js` files and registers `build`, `serve`, `preview`, and related targets. `buildTargetName: "build"` means the registered target will be named `build`.
- **`@nx/eslint/plugin`** — Scans for `eslint.config.mjs` files and registers a `lint` target.
- **`@nx/jest/plugin`** — Scans for `jest.config.[jt]s` files and registers a `test` target. The `exclude` for `apps/jobber-auth-e2e` means Nx won't auto-register tests for the e2e app (it has its own manual config).

---

### package.json

**Purpose:** The standard Node.js package manifest. At the root of a monorepo it defines shared dependencies for all apps and tools.

```json
{
  "name": "@jobber/source",
  "private": true,
  "dependencies": { ... },
  "devDependencies": { ... }
}
```

**Key settings:**

- **`name: "@jobber/source"`** — The workspace name. Using a scoped name (`@jobber/`) is a convention that signals this is an organisational monorepo root, not a publishable package.
- **`private: true`** — Prevents accidentally publishing this root package to npm.
- **`scripts: {}`** — Empty here because Nx handles all task running. You use `nx build jobber-auth` instead of `npm run build`.

**Dependencies breakdown:**

| Package | Why it's here |
|---|---|
| `@nestjs/common`, `@nestjs/core`, `@nestjs/platform-express` | NestJS framework — the core, decorators, and Express HTTP adapter |
| `@prisma/client` | The Prisma runtime client that your app code imports |
| `reflect-metadata` | Required by TypeScript decorators (used heavily by NestJS) |
| `rxjs` | Reactive extensions — NestJS uses Observables internally |
| `axios` | HTTP client, commonly used in NestJS microservice communication |
| `prisma` (devDep) | The Prisma CLI — `prisma migrate`, `prisma generate`, etc. |
| `@nx/*` packages | Nx plugins for webpack, jest, eslint, nest, node, etc. |
| `typescript-eslint` | ESLint rules that understand TypeScript syntax |
| `ts-jest` | Lets Jest run TypeScript test files directly via ts-jest transform |
| `@swc/core`, `@swc-node/register` | SWC is a Rust-based TS/JS compiler — faster alternative to `tsc` for dev transforms |

---

### tsconfig.base.json

**Purpose:** The single shared TypeScript configuration that every app and library in the monorepo extends. Defines the common compiler behaviour.

```json
{
  "compilerOptions": {
    "rootDir": ".",
    "sourceMap": true,
    "moduleResolution": "node",
    "emitDecoratorMetadata": true,
    "experimentalDecorators": true,
    "target": "es2015",
    "module": "esnext",
    ...
  }
}
```

**Key settings:**

| Option | Value | What it means |
|---|---|---|
| `rootDir` | `"."` | The root of the source tree is the repo root. Ensures output paths mirror source paths. |
| `sourceMap` | `true` | Generates `.map` files so debuggers and stack traces point to TypeScript source lines, not compiled JS. |
| `moduleResolution` | `"node"` | Resolves imports the same way Node.js does (checks `node_modules`, `index.ts`, etc.). |
| `emitDecoratorMetadata` | `true` | Required for NestJS dependency injection — emits type metadata for decorators like `@Injectable()`. |
| `experimentalDecorators` | `true` | Enables the `@Decorator()` syntax used throughout NestJS. |
| `importHelpers` | `true` | Uses `tslib` to share helper functions across compiled files instead of inlining them — reduces bundle size. |
| `target` | `"es2015"` | Compiles TypeScript down to ES2015 (ES6) JavaScript. |
| `module` | `"esnext"` | Outputs ES module `import/export` syntax — bundlers like webpack handle the final format. |
| `lib` | `["es2020", "dom"]` | Which built-in type definitions are available. `es2020` includes Promises, Map, etc. `dom` adds browser types (useful even in Node if you reference `setTimeout` etc.). |
| `skipLibCheck` | `true` | Skips type-checking `.d.ts` files in `node_modules`. Speeds up compilation significantly. |
| `paths` | `{}` | Empty here but this is where Nx adds path aliases like `@jobber/shared` → `libs/shared/src`. |

---

### jest.config.ts

**Purpose:** The root Jest configuration that collects all projects' test configs.

```typescript
import { getJestProjectsAsync } from '@nx/jest';

export default async (): Promise<Config> => ({
  projects: await getJestProjectsAsync(),
});
```

This file is minimal by design. `getJestProjectsAsync()` scans the Nx workspace graph and returns the paths to every app/library's individual `jest.config` file. When you run `jest` from the root, Jest loads all projects at once and can run tests across the entire monorepo.

Each individual project (like `apps/jobber-auth`) has its own `jest.config.cts` with project-specific settings.

---

### eslint.config.mjs

**Purpose:** Root ESLint configuration using the new "flat config" format (ESLint v9+). All app-level ESLint configs extend this.

```js
import nx from '@nx/eslint-plugin';

export default [
  ...nx.configs['flat/base'],
  ...nx.configs['flat/typescript'],
  ...nx.configs['flat/javascript'],
  { ignores: ['**/dist', '**/out-tsc'] },
  {
    files: ['**/*.ts', '**/*.tsx', '**/*.js', '**/*.jsx'],
    rules: {
      '@nx/enforce-module-boundaries': ['error', { ... }],
    },
  },
];
```

**Key concepts:**

- **Flat config** — ESLint v9 replaced `.eslintrc.json` with a flat `eslint.config.mjs` array. Each element in the array is a config object that can target specific files and define rules.
- **`nx.configs['flat/base']`** — Nx's base rules (file naming conventions, etc.).
- **`nx.configs['flat/typescript']`** — TypeScript-specific lint rules via `typescript-eslint`.
- **`@nx/enforce-module-boundaries`** — The most important Nx lint rule. It enforces that apps/libs only import from other apps/libs they are allowed to depend on, preventing circular dependencies and architectural violations. `depConstraints` with `sourceTag: '*'` means any project can depend on any other (no restrictions yet — you add tags to projects to tighten this later).

---

### .gitignore

**Purpose:** Tells git which files and directories to never track.

**Notable entries:**

| Entry | Why |
|---|---|
| `dist`, `tmp`, `out-tsc` | Compiled output — regenerated by the build, never committed |
| `node_modules` | npm packages — restored by `npm install` |
| `.nx/cache`, `.nx/workspace-data` | Nx's local computation cache — machine-specific, very large |
| `/apps/jobber-auth/.env` | Environment variables with secrets — never committed |
| `**/src/generated/prisma` | Prisma-generated client code — regenerated by `prisma generate` |

---

### docker-compose.yaml

**Purpose:** Defines local development infrastructure — currently just a PostgreSQL database.

```yaml
services:
  postgres:
    image: postgres
    ports:
      - '5432:5432'
    environment:
      POSTGRES_PASSWORD: example
```

**What it does:** Running `docker compose up` starts a PostgreSQL container accessible at `localhost:5432`. The password `example` is intentionally simple — this is only for local development, not production. The `AUTH_DATABASE_URL` in your `.env` file would point here: `postgresql://postgres:example@localhost:5432/jobber_auth`.

---

## apps/jobber-auth — App-Level Files

### project.json

**Purpose:** The Nx project configuration for the `jobber-auth` app. Defines all the tasks (targets) Nx can run for this project.

```json
{
  "name": "jobber-auth",
  "projectType": "application",
  "sourceRoot": "apps/jobber-auth/src",
  "targets": { ... }
}
```

**Targets explained:**

#### `build`
```json
{
  "executor": "nx:run-commands",
  "options": {
    "command": "webpack-cli build",
    "args": ["--node-env=production"],
    "cwd": "apps/jobber-auth"
  }
}
```
Runs webpack to compile the app. Uses `nx:run-commands` because it's a raw shell command rather than an Nx executor. The `development` configuration swaps `--node-env=production` for `--node-env=development`.

#### `serve`
```json
{
  "executor": "@nx/js:node",
  "dependsOn": ["build"],
  "continuous": true
}
```
Builds the app then runs it with file watching. `continuous: true` means it stays alive and restarts on changes. `dependsOn: ["build"]` ensures the build always runs first.

#### `prune-lockfile` and `copy-workspace-modules`
These two targets prepare the app for deployment by producing a self-contained `dist/` folder. `prune-lockfile` generates a minimal `package-lock.json` with only the production dependencies. `copy-workspace-modules` copies the actual `node_modules` for those dependencies.

#### `prune`
A no-op target (`nx:noop`) that depends on both `prune-lockfile` and `copy-workspace-modules`. It's a convenience target — running `nx run jobber-auth:prune` runs both preparation steps.

#### `generate-types`
```json
{
  "command": "prisma generate",
  "options": { "cwd": "apps/jobber-auth" }
}
```
Runs `prisma generate` to produce the TypeScript client from `schema.prisma`. Run this after changing the schema.

---

### tsconfig.json

**Purpose:** The TypeScript project root for `jobber-auth`. It doesn't compile anything directly — it exists to group the two compilation configs (`app` and `spec`) under one project.

```json
{
  "extends": "../../tsconfig.base.json",
  "files": [],
  "include": [],
  "references": [
    { "path": "./tsconfig.app.json" },
    { "path": "./tsconfig.spec.json" }
  ]
}
```

- **`files: []`, `include: []`** — Intentionally empty. This file is not meant to compile any files itself.
- **`references`** — TypeScript project references. This links the three tsconfigs together so editors and `tsc --build` understand the relationship between them.
- **`esModuleInterop: true`** — Allows `import fs from 'fs'` instead of `import * as fs from 'fs'` for CommonJS modules.

---

### tsconfig.app.json

**Purpose:** TypeScript configuration for compiling the actual application source code (not tests).

```json
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "module": "commonjs",
    "target": "es2021",
    "types": ["node"],
    ...
  },
  "include": ["src/**/*.ts"],
  "exclude": ["src/**/*.spec.ts", "src/**/*.test.ts"]
}
```

**Key differences from `tsconfig.base.json`:**

| Option | Value | Why |
|---|---|---|
| `module` | `"commonjs"` | Node.js uses CommonJS (`require`/`module.exports`). The base uses `esnext` for bundlers, but the compiled app must use CommonJS. |
| `target` | `"es2021"` | Targets a newer JS version than the base (`es2015`) since Node.js 16+ supports ES2021 features natively — no need to polyfill. |
| `types` | `["node"]` | Only includes Node.js type definitions (`process`, `Buffer`, `__dirname`, etc.). Keeps the app's type environment clean. |
| `include` | `src/**/*.ts` | Only compiles app source files. |
| `exclude` | spec/test files | Test files are compiled by `tsconfig.spec.json`, not this one. |

---

### tsconfig.spec.json

**Purpose:** TypeScript configuration for compiling test files only.

```json
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "module": "commonjs",
    "moduleResolution": "node10",
    "types": ["jest", "node"]
  },
  "include": ["jest.config.ts", "jest.config.cts", "src/**/*.spec.ts", "src/**/*.test.ts"]
}
```

**Key differences from `tsconfig.app.json`:**

| Option | Value | Why |
|---|---|---|
| `types` | `["jest", "node"]` | Adds Jest globals (`describe`, `it`, `expect`) to the type environment. Only test files need these. |
| `moduleResolution` | `"node10"` | A slightly older resolution algorithm, needed for compatibility with how `ts-jest` resolves modules. |
| `include` | spec/test files + jest configs | Only test-related files are compiled by this config. |

---

### jest.config.cts

**Purpose:** Jest configuration for the `jobber-auth` project specifically. Uses `.cts` extension (CommonJS TypeScript) because Jest's config loading requires CommonJS format.

```js
module.exports = {
  displayName: 'jobber-auth',
  preset: '../../jest.preset.js',
  testEnvironment: 'node',
  transform: {
    '^.+\\.[tj]s$': ['ts-jest', { tsconfig: '<rootDir>/tsconfig.spec.json' }],
  },
  moduleFileExtensions: ['ts', 'js', 'html'],
  coverageDirectory: '../../coverage/apps/jobber-auth',
};
```

| Option | Value | What it means |
|---|---|---|
| `displayName` | `"jobber-auth"` | Label shown in test output when running tests across the whole monorepo. |
| `preset` | `../../jest.preset.js` | Base Jest settings shared across all Nx projects (generated by Nx). |
| `testEnvironment` | `"node"` | Runs tests in a Node.js environment, not jsdom (which is for browser code). |
| `transform` | `ts-jest` | Tells Jest to use `ts-jest` to compile `.ts` and `.js` files before running them. Points at `tsconfig.spec.json` for type-checking rules. |
| `moduleFileExtensions` | `['ts', 'js', 'html']` | When resolving imports without extensions, try these in order. |
| `coverageDirectory` | `../../coverage/apps/jobber-auth` | Where to write coverage reports when running with `--coverage`. |

---

### webpack.config.js

**Purpose:** Webpack bundler configuration for building the NestJS app into a single deployable file.

```js
const { NxAppWebpackPlugin } = require('@nx/webpack/app-plugin');

module.exports = {
  output: {
    path: join(__dirname, '../../dist/apps/jobber-auth'),
    clean: true,
  },
  plugins: [
    new NxAppWebpackPlugin({
      target: 'node',
      compiler: 'tsc',
      main: './src/main.ts',
      tsConfig: './tsconfig.app.json',
      optimization: false,
      outputHashing: 'none',
      generatePackageJson: true,
      sourceMap: true,
    }),
  ],
};
```

| Option | Value | What it means |
|---|---|---|
| `output.path` | `dist/apps/jobber-auth` | Where the compiled bundle is written. |
| `output.clean` | `true` | Deletes the output directory before each build — no stale files. |
| `devtoolModuleFilenameTemplate` | absolute path | In development mode only: makes source map paths absolute so your debugger can find the original `.ts` files. |
| `target: 'node'` | — | Tells webpack this is a Node.js app, not a browser app. Excludes built-in Node modules from the bundle. |
| `compiler: 'tsc'` | — | Uses the TypeScript compiler (not SWC or Babel) to transform files. |
| `main: './src/main.ts'` | — | The entry point — webpack starts building the dependency graph from here. |
| `optimization: false` | — | Disables minification and tree-shaking. Unnecessary for a server app and makes debugging easier. |
| `outputHashing: 'none'` | — | No content hash appended to filenames (like `main.abc123.js`). Not needed for server apps. |
| `generatePackageJson: true` | — | Writes a `package.json` into `dist/` listing only the production dependencies actually used — useful for Docker deployments. |
| `sourceMap: true` | — | Generates source maps so stack traces in production point to TypeScript source lines. |

---

### eslint.config.mjs (app)

**Purpose:** ESLint configuration for the `jobber-auth` app. Extends the root config with no additions.

```js
import baseConfig from '../../eslint.config.mjs';
export default [...baseConfig];
```

It simply spreads the root config. This file exists so ESLint can scope its rules to this project's directory. App-specific rules would be added here as additional array elements.

---

### prisma.config.ts

**Purpose:** Prisma CLI configuration file (introduced in Prisma v6). Tells the Prisma CLI where to find the schema, where to put migrations, and how to connect to the database.

```typescript
import 'dotenv/config';
import { defineConfig, env } from 'prisma/config';

export default defineConfig({
  schema: 'prisma/schema.prisma',
  migrations: {
    path: 'prisma/migrations',
  },
  datasource: {
    url: env('AUTH_DATABASE_URL'),
  },
});
```

| Setting | Value | What it means |
|---|---|---|
| `import 'dotenv/config'` | — | Loads the `.env` file before anything else so `env()` can read environment variables. |
| `schema` | `prisma/schema.prisma` | Path to the schema file, relative to this config file. |
| `migrations.path` | `prisma/migrations` | Where `prisma migrate` reads and writes migration SQL files. |
| `datasource.url` | `env('AUTH_DATABASE_URL')` | Reads the database connection string from the `AUTH_DATABASE_URL` environment variable. Never hardcoded. |

This file is run by the Prisma CLI (`prisma migrate`, `prisma generate`, etc.), not by your application. Your application uses the generated Prisma Client at runtime.

---

### prisma/schema.prisma

**Purpose:** The Prisma schema — the single source of truth for your database structure and how the Prisma Client is generated.

```prisma
generator client {
  provider     = "prisma-client-js"
  output       = "../src/generated/prisma"
  moduleFormat = "cjs"
}

datasource db {
  provider = "postgresql"
}

model User {
  id       Int    @default(autoincrement()) @id
  email    String @unique
  password String
}
```

**Sections:**

#### `generator client`
Configures what `prisma generate` produces.

| Option | Value | What it means |
|---|---|---|
| `provider` | `prisma-client-js` | Generate a JavaScript/TypeScript client (the standard Prisma Client). |
| `output` | `../src/generated/prisma` | Write the generated client to this path (relative to the schema file), i.e. `apps/jobber-auth/src/generated/prisma`. This directory is gitignored — it's regenerated on demand. |
| `moduleFormat` | `"cjs"` | Generate CommonJS modules (`require`/`module.exports`). Matches the app's `module: "commonjs"` TypeScript setting. |

#### `datasource db`
Tells Prisma which database engine to use. `provider = "postgresql"` means Prisma generates PostgreSQL-compatible SQL for migrations and uses the PostgreSQL wire protocol. The actual connection URL comes from `prisma.config.ts` at CLI time, or from an environment variable at runtime.

#### `model User`
Defines a database table called `users` (Prisma pluralises by convention) with three columns:

| Field | Type | Attributes | SQL equivalent |
|---|---|---|---|
| `id` | `Int` | `@id @default(autoincrement())` | `id SERIAL PRIMARY KEY` |
| `email` | `String` | `@unique` | `email TEXT UNIQUE NOT NULL` |
| `password` | `String` | — | `password TEXT NOT NULL` |

When you run `prisma migrate dev`, Prisma diffs this schema against the current database state and generates the SQL to bring them in sync.
