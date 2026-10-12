---
name: boilerplate
description: Scaffold a new empty-but-wired pnpm workspace with a Next.js 16 App Router app (React 19, Mantine 9), type-graphql + graphql-yoga, Drizzle/SQLite, next-auth credentials, Apollo Client, oxlint/oxfmt at the root, port 3000. Use when the user asks to start/scaffold/boilerplate a new project of this stack, or to add an entity to one. Describes structure only - no domain entities.
---

# Project Boilerplate — Scaffold Specification

This document describes the **structure** of a project, not its business domain. An AI agent reading this file must be able to produce a working, empty-but-wired project that later receives domain logic (e.g. a CRM, a booking system, a helpdesk) without changing any of the plumbing.

Rule: **no domain entities are created by this document.** The only entity wired end-to-end is a placeholder `User` (id / name / secret / role) because authentication cannot exist without it. Everything else is added later by following §12.

---

## 0. Ground rules

- Package manager is **pnpm**. Never `npm`, `yarn`, `bun`.
- **Node ≥ 22**, declared in root `engines`. `node --env-file-if-exists` (scripts, §14) and the Next/drizzle toolchain both assume it.
- **Versions are always the latest available** at scaffold time. Do not copy pinned versions from anywhere; resolve each package with `pnpm add <pkg>@latest` / `pnpm add -D <pkg>@latest`, then commit whatever the lockfile resolves. The version list below is a _role_ list, not a version spec.
- Do not add packages beyond the role lists in §3 without explicit approval (name, latest version, weekly downloads, why, alternatives).
- Verification loop for every change: `pnpm typecheck`, `pnpm lint`, `pnpm format:check`, `pnpm build`. No test suite is configured.
- **oxfmt** is the formatter and **oxlint** is the linter (see §4 and §13). No ESLint, no Prettier, no Biome.
- Comments explain _why_, one line, never _what_.

---

## 0.5 Before you write a single file — ask

**Do not invent these names.** Ask the user first; it takes one round trip and everything downstream (dir names, package names, metadata, `AGENTS.md`) is derived from the answers.

1. **Root project name** — the repository / workspace name. Kebab-case, e.g. `acme-crm`. Goes in root `package.json#name`.
2. **Root description** — one line describing the whole project. Goes in root `package.json#description`.
3. **App package name** — the name of the Next.js. This is a _directory_ name and a _package_ name, so keep it short and kebab-case (e.g. `admin`, `dashboard`, `web`, `backoffice`). If the user has no preference, propose `<root>` itself (e.g. `acme-crm`) and say so in the question.

Ask all three in one message. Suggested form:

```
1. Root package name? (kebab-case, used for the repo + workspace)  e.g. acme-crm
2. Short description of the project?                                  e.g. CRM for the sales team
3. Name for the app? (folder + package name)               e.g. admin
```

Only then read §1–§17 and scaffold. Do not ask about anything the document already decides (stack, tooling, ports) — decide those, don't ask.

---

## 0.6 Command map — where things run

**Every command in this document is run from the workspace root.** The root `package.json` proxies the app's work with `pnpm -F <app>` and invokes the tooling directly against `app/<app>`. Do not `cd` into the app package to run any of these.

| Command                                  | Root | Resolves to                                |
| ---------------------------------------- | ---- | ------------------------------------------ |
| `pnpm install`                           | root | one lockfile for the whole workspace       |
| `pnpm dev`                               | root | `next dev -p 3000` in `app/<app>`          |
| `pnpm build` / `pnpm start`              | root | `next build` / `next start` in `app/<app>` |
| `pnpm typecheck`                         | root | `tsc --noEmit` in `app/<app>`              |
| `pnpm lint` / `pnpm lint:fix`            | root | `oxlint [app/<app>]` — root binary         |
| `pnpm format` / `pnpm format:check`      | root | `oxfmt [app/<app>]` — root binary          |
| `pnpm db:push` / `db:seed` / `db:studio` | root | drizzle-kit / seed script in `app/<app>`   |
| `pnpm codegen`                           | root | `graphql-codegen` in `app/<app>`           |
| `pnpm verify`                            | root | typecheck + lint + format:check + build    |

`pnpm add <pkg>` for app dependencies must run **inside `app/<app>/`** (or use `pnpm -F <app> add <pkg>`); tooling installs run at the root.

---

## 1. Stack

| Layer               | Choice                                                                                                    |
| ------------------- | --------------------------------------------------------------------------------------------------------- |
| Framework           | Next.js (latest 16.x) **App Router**, React 19                                                            |
| UI                  | Mantine (latest 9.x) — `@mantine/core`, `@mantine/hooks`, `@mantine/notifications`, `@tabler/icons-react` |
| API                 | `type-graphql` (v2 RC — decorators, class-based) + `graphql-yoga`                                         |
| Schema/ops codegen  | `@graphql-codegen/cli` → typed document nodes                                                             |
| ORM                 | Drizzle ORM over SQLite via `@libsql/client` (`drizzle-orm/libsql`)                                       |
| Auth                | `next-auth` v4, Credentials provider (`id` + shared secret)                                               |
| Client data         | Apollo Client 4 (`@apollo/client` + `@apollo/client-integration-nextjs`)                                  |
| Tables              | `@tanstack/react-table` (wrapped once, see §10)                                                           |
| Forms               | `react-hook-form` (+ `Controller` for Mantine non-native inputs)                                          |
| N+1 protection      | `dataloader`                                                                                              |
| Validation (server) | `class-validator` decorators on inputs                                                                    |
| Misc                | `uuid`, `reflect-metadata`, `rxjs`, `clsx`                                                                |

**pnpm workspace with two layers**, which is what makes it a monorepo even though there is one app:

| Layer            | Holds                                                                                       | Never holds      |
| ---------------- | ------------------------------------------------------------------------------------------- | ---------------- |
| **root**         | `oxlint`, `oxfmt`, `oxlint-tsgolint` + their configs, `pnpm-workspace.yaml`, shared scripts | app runtime deps |
| **`app/<app>/`** | the Next.js app and **all** of its dependencies                                             | the tooling      |

The tooling lives **only** at the root and is never duplicated into the app package. Consequences you must respect throughout this document:

- The app's `package.json` has **no** `lint`/`format` scripts — the root owns them, and they take the app directory as an argument.
- Every verification command runs **from the root**, never from inside `app/<app>/`.
- Install app dependencies from `app/<app>/` (`pnpm add` there), tooling from the root. One lockfile, one `node_modules` at the root with pnpm symlinks per package.

---

## 2. Directory tree (create exactly this)

`<root>` is the repo folder from §0.5 answer 1, `<app>` is **the same string** — that answer is also the app package name, so the app folder under `app/` is `app/<root>/`. Substitute one name for both placeholders everywhere.

```
<root>/
├─ app/
│  └─ <app>/                      ← the Next.js package; everything below is relative to THIS folder
│     ├─ app/                     ← App Router (note: app/<app>/app/, this is intentional)
│     │  ├─ layout.tsx            # root layout: html/head/body, MantineProvider, providers
│     │  ├─ global.css            # mantine css imports + theme vars
│     │  ├─ page.tsx              # home / module launcher (server component, session-gated)
│     │  ├─ login/page.tsx        # user picker + secret entry
│     │  ├─ api/
│     │  │  ├─ route.ts           # authenticated GraphQL endpoint (/api)
│     │  │  └─ public/route.ts    # unauthenticated GraphQL endpoint (/api/public)
│     │  └─ <module>/             # one folder per feature area, e.g. manage/
│     │     ├─ layout.tsx         # shell + role gate for that area
│     │     └─ <entity>/page.tsx  # CRUD screens
│     ├─ components/              # 'use client' shared UI
│     │  ├─ providers.tsx
│     │  ├─ confirm.tsx
│     │  ├─ data-table.tsx
│     │  ├─ user.tsx
│     │  └─ setting-control.tsx
│     ├─ database/
│     │  ├─ index.ts              # drizzle client + schema registry
│     │  └─ models/
│     │     ├─ base.model.ts
│     │     └─ <entity>.model.ts  # table (+ relations when needed)
│     ├─ graphql/
│     │  ├─ schema.ts             # builds the authenticated schema (memoized)
│     │  ├─ client.ts             # Apollo client factory (browser + server use)
│     │  ├─ scalars.ts            # DateTime scalar + one scalar per entity id + registerEnumType
│     │  ├─ decorators.ts         # shared @Pagination() arg + PaginationInput
│     │  ├─ loaders.ts            # DataLoaders, one per hot relation
│     │  ├─ request-context.ts
│     │  ├─ inputs/<entity>.input.ts
│     │  ├─ types/<entity>.type.ts
│     │  ├─ types/success-payload.ts
│     │  ├─ resolvers/<entity>.resolver.ts
│     │  └─ operations/
│     │     ├─ queries.graphql
│     │     └─ mutations.graphql
│     ├─ lib/
│     │  ├─ auth.ts               # next-auth options
│     │  ├─ common.state.ts       # makeVar global UI state
│     │  └─ helpers.ts
│     ├─ scripts/                 # plain .mjs run by bare node — never .ts, never tsx (§14)
│     ├─ types/
│     │  ├─ enums.ts              # plain TS enums, registered into GraphQL
│     │  ├─ identities.ts         # branded nominal id types
│     │  ├─ gql.ts                # GENERATED — do not hand-edit
│     │  └─ next-auth.d.ts        # session/JWT augmentation
│     ├─ public/
│     │  ├─ logo.png, default-logo.png
│     │  └─ <uploads>/            # user-uploaded files (gitignored)
│     ├─ codegen.ts
│     ├─ drizzle.config.ts
│     ├─ next.config.ts
│     ├─ next-env.d.ts            # generated by next
│     ├─ package.json             # app deps only — NO lint/format scripts
│     ├─ postcss.config.mjs
│     ├─ tsconfig.json
│     ├─ .env                     # gitignored, app-scoped
│     └─ dev.db                   # gitignored, created by db:push
├─ .oxlintrc.json                 # linter config (type-aware) — ROOT ONLY
├─ .oxfmtrc.json                  # formatter config — ROOT ONLY
├─ .gitignore                     # one file, covers every package
├─ AGENTS.md                      # repo-local agent instructions
├─ package.json                   # root: name + description + tooling + shared scripts
├─ pnpm-workspace.yaml            # workspace globs + pnpm settings
├─ pnpm-lock.yaml                 # one lockfile for the whole workspace
└─ README.md
```

Rules:

- The App Router folder lives at `app/<app>/app/`. Next.js requires `app/` relative to the package root; do not "fix" the double `app` by moving things.
- `app/<app>/tsconfig.json` sets `"@/*": ["./*"]` — the alias maps to the **package** root, not the workspace root. `@/components/...` resolves inside the app package.
- `.gitignore`, `.oxlintrc.json`, `.oxfmtrc.json`, `AGENTS.md` and `package.json` are workspace-level single copies. If a second app is ever added, it reuses all of them unchanged.
- Never create `src/`.
- From §5 onward, every file path in this document is **relative to `app/<app>/`** unless it is explicitly prefixed with `app/<app>/` or described as root-level. So `types/identities.ts` means `app/<app>/types/identities.ts`.

---

## 3. `package.json` — two files

### Root `package.json`

Name and description come straight from the §0.5 answers. It owns the tooling and **all** shared scripts, each one prefixed with the workspace filter so it targets the app package. The app's own `dev`/`build`/`db:*`/`codegen` scripts stay on the app package and are called with `-F` (filter).

```jsonc
{
  "name": "<root>", // ← §0.5 answer 1
  "version": "0.1.0",
  "private": true,
  "description": "<root description>", // ← §0.5 answer 2
  "type": "module",
  "devEngines": {
    "packageManager": { "name": "pnpm", "version": "<your pnpm>", "onFail": "download" },
  },
  "packageManager": "pnpm@<your pnpm>",
  "engines": { "node": ">=22" },
  "scripts": {
    // pass-through to the app package
    "dev": "pnpm -F <app> dev",
    "build": "pnpm -F <app> build",
    "start": "pnpm -F <app> start",
    "typecheck": "pnpm -F <app> typecheck",
    "codegen": "pnpm -F <app> codegen",

    // tooling runs HERE — the binary only exists in the root node_modules
    "lint": "oxlint",
    "lint:fix": "oxlint --fix",
    "format": "oxfmt",
    "format:check": "oxfmt --check",

    "verify": "pnpm typecheck && pnpm lint && pnpm format:check && pnpm build",
  },
  "devDependencies": {
    "oxfmt": "<latest>",
    "oxlint": "<latest>",
    "oxlint-tsgolint": "<latest>",
  },
}
```

Substitute `<app>` with the §0.5 answer 1 folder name everywhere — including inside the `lint`/`format` paths. It is the same string as `<root>`. Do not add `oxlint`/`oxfmt`/`oxlint-tsgolint` to the app package.

### `app/<app>/package.json`

No `lint`, no `format`, no `typecheck` that reaches for root binaries — but `tsc` and `drizzle-kit` _are_ local devDeps of the app, so `typecheck` and the `db:*` scripts work from here.

```jsonc
{
  "name": "<app>",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "dev": "next dev -p 3000",
    "build": "next build",
    "start": "next start",
    "typecheck": "tsc --noEmit",
    "codegen": "graphql-codegen",
    "db:push": "drizzle-kit push",
    "db:studio": "drizzle-kit studio",
    "db:seed": "node --env-file-if-exists=.env scripts/seed.mjs",
  },
}
```

There is deliberately **no** `db:generate` / `db:migrate` script on the app package. Schema changes are applied with `pnpm db:push`; a SQL migration workflow is not part of this stack (§16). Do not add them back "just in case".

Dependencies — install the **latest** of each into `app/<app>/`: `@apollo/client`, `@apollo/client-integration-nextjs`, `@graphql-yoga/plugin-persisted-operations`, `@libsql/client`, `@mantine/core`, `@mantine/hooks`, `@mantine/notifications`, `@tabler/icons-react`, `@tanstack/react-table`, `class-validator`, `clsx`, `dataloader`, `drizzle-orm`, `graphql`, `graphql-yoga`, `next`, `next-auth`, `postcss`, `postcss-preset-mantine`, `react`, `react-dom`, `react-hook-form`, `reflect-metadata`, `rxjs`, `type-graphql`, `uuid`.

Dev (app only): `@graphql-codegen/cli`, `@graphql-typed-document-node/core`, `@types/node`, `@types/react`, `@types/react-dom`, `drizzle-kit`, `typescript`, `typescript-eslint`.

Lock `@mantine/*` and `react`/`react-dom` to exact matching versions across the three.

### Root `pnpm-workspace.yaml`

Now a real workspace — it declares the packages **and** carries pnpm settings.

```yaml
packages:
  - 'app/*'

allowBuilds:
  esbuild: true
  sharp: true
  unrs-resolver: true

# add the minimumReleaseAgeExclude entries your pnpm version requires for the
# Next packages younger than the release-age window — regenerate from install
# warnings, do not copy a list from another repo
```

There is no `pnpm-lock.yaml` inside `app/<app>/` — the workspace has exactly one lockfile, at the root. Run `pnpm install` from the root.

---

## 4. Config files

Two root files (`.oxlintrc.json`, `.oxfmtrc.json`) and the rest inside `app/<app>/`.

### `app/<app>/tsconfig.json`

Next defaults plus the three decorator flags (type-graphql is mandatory here). `@/*` maps to the **package** root:

```jsonc
{
  "compilerOptions": {
    "target": "esnext",
    "lib": ["dom", "dom.iterable", "esnext"],
    "module": "esnext",
    "moduleResolution": "bundler",
    "jsx": "react-jsx",
    "allowJs": false,
    "skipLibCheck": true,
    "strict": true,
    "noEmit": true,
    "esModuleInterop": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "incremental": true,
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true,
    "plugins": [{ "name": "next" }],
    "paths": { "@/*": ["./*"] },
    "forceConsistentCasingInFileNames": true,
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts", ".next/dev/types/**/*.ts"],
  "exclude": ["node_modules", ".next"],
}
```

### `app/<app>/next.config.ts`

Mantine barrel-import optimization and TS CLI typechecking:

```ts
import type { NextConfig } from 'next';

const nextConfig: NextConfig = {
  reactCompiler: false,
  experimental: {
    optimizePackageImports: ['@mantine/core', '@mantine/hooks'],
    useTypeScriptCli: true,
  },
};

export default nextConfig;
```

### `app/<app>/postcss.config.mjs`

```js
const config = { plugins: { 'postcss-preset-mantine': {} } };
export default config;
```

### `app/<app>/drizzle.config.ts`

Schema is the models folder, URL comes from env. Paths are relative to the app package, so `pnpm db:push` must run with that package as cwd — which the root `pnpm -F <app> db:push` guarantees.

```ts
import { defineConfig } from 'drizzle-kit';

export default defineConfig({
  out: '.drizzle',
  schema: './database/models',
  dialect: 'sqlite',
  dbCredentials: { url: process.env.DATABASE_FILE! },
});
```

### Root `.oxlintrc.json`

The linter, and it lives **at the workspace root only**. Type-aware (`typeAware` + `typeCheck` both on) so rules like `no-floating-promises` actually work; this needs `oxlint-tsgolint` in the **root** devDependencies. `scripts/*` and config files are ignored because they are plain `.mjs`/`.json` with no TS project behind them — they are still _formatted_, see §13.

```jsonc
{
  "$schema": "./node_modules/oxlint/configuration_schema.json",
  "plugins": ["typescript", "nextjs", "react"],
  "categories": { "correctness": "warn" },
  "globals": { "console": "readonly" },
  "options": { "typeAware": true, "typeCheck": true },
  "env": { "builtin": true },
  "ignorePatterns": [
    ".next/**",
    "out/**",
    "build/**",
    "next-env.d.ts",
    "node_modules/**",
    "postcss.config.mjs",
    "drizzle.config.ts",
    "**/scripts/*",
  ],
  "rules": {
    "no-array-constructor": "error",
    "no-unused-expressions": "warn",
    "no-unused-vars": "warn",
    "typescript/await-thenable": "error",
    "typescript/ban-ts-comment": "error",
    "typescript/no-array-delete": "error",
    "typescript/no-base-to-string": "error",
    "typescript/no-duplicate-enum-values": "error",
    "typescript/no-empty-object-type": "error",
    "typescript/no-extra-non-null-assertion": "error",
    "typescript/no-floating-promises": "error",
    "typescript/no-implied-eval": "error",
    "typescript/no-namespace": "error",
    "typescript/no-non-null-asserted-optional-chain": "error",
    "typescript/no-require-imports": "error",
    "typescript/no-this-alias": "error",
    "typescript/no-unnecessary-type-assertion": "error",
    "typescript/no-unsafe-argument": "error",
    "typescript/no-unsafe-assignment": "error",
    "typescript/no-unsafe-call": "error",
    "typescript/no-unsafe-member-access": "error",
    "typescript/only-throw-error": "error",
    "typescript/prefer-as-const": "error",
    "typescript/prefer-promise-reject-errors": "error",
    "typescript/require-await": "error",
    "typescript/restrict-plus-operands": "error",
    "typescript/restrict-template-expressions": "error",
    "typescript/triple-slash-reference": "error",
    "typescript/unbound-method": "error",
  },
}
```

Note the deliberate omissions: `no-explicit-any`, `typescript/no-misused-spread` and a few other strict rules are off. Apollo/Drizzle code is full of legitimate `any` and spread patterns — do not add workarounds to satisfy rules the project has chosen not to enforce.

### Root `.oxfmtrc.json`

The formatter, at the workspace root only. `sortImports` is what keeps the alphabetical `@/`-grouped import blocks consistent; run `pnpm format` before you finish, not after.

```jsonc
{
  "$schema": "./node_modules/oxfmt/configuration_schema.json",
  "printWidth": 120,
  "semi": true,
  "singleQuote": true,
  "quoteProps": "as-needed",
  "jsxSingleQuote": false,
  "trailingComma": "es5",
  "bracketSpacing": true,
  "bracketSameLine": false,
  "arrowParens": "always",
  "endOfLine": "lf",
  "sortPackageJson": false,
  "sortImports": true,
  "ignorePatterns": [],
}
```

### `app/<app>/.env`

App-scoped, never committed — commit `app/<app>/.env.example` instead so the next person knows what to fill in.

```
DATABASE_FILE=file:dev.db
AUTH_SECRET=<openssl rand -hex 32>
AUTH_URL=http://localhost:3000
IMAGE_BASE_URL=http://localhost:3000
```

Next.js only loads `.env` from the package it is running in, so this file lives in `app/<app>/`, **not** at the workspace root. `pnpm -F <app> dev` sets the cwd there, which is why `.env` is found.

### Root `.gitignore`

One file for the whole workspace. Must include `node_modules`, `.next/`, `out/`, `build`, `dist`, `*.tsbuildinfo`, `next-env.d.ts`, `.env*`, `*.db`, `.drizzle`, `app/*/public/<uploads>/`, `.vercel`, `coverage`.

**Port 3000 is fixed.** `codegen.ts`, `graphql/client.ts` and any sibling frontend hard-code `http://localhost:3000/api`. Bake `-p 3000` into the app's `dev` script so it cannot drift.

---

## 5. Types layer

**`types/identities.ts`** — every entity id is a branded string so the compiler catches id mix-ups:

```ts
type Nominal<T, Name extends string> = T & { readonly brand: Name };

export type UserId = Nominal<string, 'UserId'>;
// one line per entity added later
```

**`types/enums.ts`** — plain TS string enums plus a route registry. No domain values here yet; add enum members and route paths as features land, and register each enum in `graphql/scalars.ts`.

```ts
export enum UserType {
  Admin = 'admin',
  Operator = 'operator',
}

export enum Routes {
  Home = '/',
  Login = '/login',
  Manage = '/manage',
  Users = '/manage/users',
}
```

**`types/next-auth.d.ts`** — augment `Session`, `User` and `JWT` with `id: UserId`, `fullName: string`, `type: UserType` so session access is fully typed everywhere.

**`types/gql.ts`** — generated. Never edit, never import from a subpath.

---

## 6. Database layer

**`database/models/base.model.ts`** — the shared column factories every table spreads in:

```ts
import { integer, text } from 'drizzle-orm/sqlite-core';
import { v4 as uuidv4 } from 'uuid';

export const id = text({ length: 36 })
  .primaryKey()
  .$defaultFn(() => uuidv4());

export const timestamps = {
  createdAt: integer({ mode: 'timestamp' })
    .notNull()
    .$defaultFn(() => new Date()),
  updatedAt: integer({ mode: 'timestamp' })
    .notNull()
    .$defaultFn(() => new Date())
    .$onUpdateFn(() => new Date()),
};

export const auditFields = {
  createdBy: text({ length: 128 }).notNull(),
  updatedBy: text({ length: 128 }).notNull(),
};
```

Conventions:

- Table name is `snake_case` plural; exported const is `camelCase` plural.
- Booleans are `integer({ mode: 'boolean' })`.
- Money/quantities are plain `integer()` — never float.
- Free-form JSON is `text({ mode: 'json' })`.
- Every column carrying an id gets `.$type<ThatId>()`.
- Export `export type <Entity>Model = typeof <table>.$inferSelect;` — resolvers type `@Root()` against this, so the DB row and the GraphQL type stay aligned.
- Spread `...timestamps` on every table and `...auditFields` on every table a human mutates.
- Relations live in the same file as the child table as `<table>Relations = relations(<table>, ({ one }) => ({ ... }))`.

**`database/models/user.model.ts`** (the only pre-wired entity):

```ts
export const users = sqliteTable('users', {
  id: id.$type<UserId>(),
  firstName: text({ length: 64 }).notNull(),
  lastName: text({ length: 64 }).notNull(),
  secret: text({ length: 6 }).notNull().default('000000'),
  type: text({ length: 32 }).$type<UserType>().notNull().default(UserType.Operator),
  ...auditFields,
  ...timestamps,
});

export type UserModel = typeof users.$inferSelect;
```

**`database/index.ts`** — the single client plus an **explicit schema registry**. A table missing from this map is invisible to `db.query`:

```ts
import { drizzle } from 'drizzle-orm/libsql';

import { users } from './models/user.model';

export const db = drizzle({
  connection: { url: process.env.DATABASE_FILE! },
  logger: true,
  schema: { users },
});
```

Rules: register every table. Register relations as `{ <key>: { <table>, relations: <table>Relations } }` when the table has relations.

---

## 7. GraphQL layer

**`graphql/scalars.ts`**

```ts
import { GraphQLScalarType } from 'graphql';
import { registerEnumType } from 'type-graphql';

import { UserType } from '@/types/enums';
import type { UserId } from '@/types/identities';

const uuidRegex = /^[0-9a-f]{8}-[0-9a-f]{4}-[1-5][0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$/i;

export const DateScalar = new GraphQLScalarType({ name: 'DateTime', serialize, parseValue, parseLiteral });

function createIdScalar<T extends string>(name: string): GraphQLScalarType<T, string> {
  return new GraphQLScalarType<T, string>({
    name,
    serialize: (value) => String(value),
    parseValue: (value) => {
      if (typeof value !== 'string' || !uuidRegex.test(value)) throw new Error(`Invalid ${name}`);
      return value as T;
    },
    parseLiteral() {
      throw new Error('Literals are not allowed');
    },
  });
}

export const UserIdScalar = createIdScalar<UserId>('UserId');
// one per entity

registerEnumType(UserType, { name: 'UserType' });
// one per enum
```

**`graphql/decorators.ts`** — shared input types / arg decorators only:

```ts
@InputType('PaginationInput')
export class PaginationInput {
  @Field(() => Int) @IsInt() @Min(0) offset!: number;
  @Field(() => Int) @IsInt() @Min(1) @Max(50) limit!: number;
}

export function Pagination() {
  return Arg('pagination', () => PaginationInput, { nullable: true });
}
```

**`graphql/request-context.ts`**

```ts
import type { Session } from 'next-auth';
export interface RequestContext {
  user: Session['user'];
}
```

**`graphql/schema.ts`** — memoized schema, all resolvers listed explicitly, role check centralized:

```ts
import 'reflect-metadata';
import { GraphQLError } from 'graphql';
import { buildSchemaSync, type ResolverData } from 'type-graphql';

import type { RequestContext } from './request-context';
import { UserResolver } from './resolvers/user.resolver';

let schema: ReturnType<typeof buildSchemaSync> | null = null;

export function getSchema() {
  if (!schema) {
    schema = buildSchemaSync({
      resolvers: [UserResolver],
      authChecker: ({ context }: ResolverData<RequestContext>, roles: string[]) => {
        if (!context.user) throw new GraphQLError('Unauthorized');
        if (roles.length > 0 && !roles.includes(context.user.type.toLowerCase())) {
          throw new GraphQLError('Unauthorized', {
            extensions: { code: 'UNATHORIZED', http: { status: 401 } },
          });
        }
        return true;
      },
      authMode: 'null',
      validate: false,
    });
  }
  return schema;
}
```

`roles` are compared lowercased, so `@Authorized(UserType.Admin)` works with `UserType.Admin === 'admin'`.

**`graphql/client.ts`** — `client()` is a **factory**, because it is used from the browser _and_ from server code (the auth provider logs in through GraphQL). Hard-code the endpoint, chain four links, and declare `keyFields` for every type that is a singleton or has a composite key:

```ts
export const client = () => {
  const httpLink = new HttpLink({ uri: 'http://localhost:3000/api' });
  const trimStringsLink = new ApolloLink(/* trim + upload-safe recursion over operation.variables */);
  const removeTypeNameLink = new RemoveTypenameFromVariablesLink();
  const errorLink = new ErrorLink(/* notifications.show({ color: 'red' }) */);

  return new ApolloClient({
    link: ApolloLink.from([errorLink, removeTypeNameLink, trimStringsLink, httpLink]),
    cache: new InMemoryCache({ typePolicies: {/* singletons + composite keys */} }),
  });
};
```

**`graphql/loaders.ts`** — module-scope `DataLoader`s, `{ cache: false }` (a fresh batch per request). Pattern: one query with `inArray`, then map back into key order. Add one per relation resolved on a list; loaders are what keep list queries from N+1.

**Resolvers** (`graphql/resolvers/<entity>.resolver.ts`)

```ts
@Resolver(() => User)
export class UserResolver {
  @Query(() => [User])
  async users(@Pagination() page: PaginationInput) {
    return await db.query.users.findMany({ ...page });
  }

  @Mutation(() => User)
  async createUser(@Arg('input') input: CreateUserInput) {
    const [result] = await db
      .insert(users)
      .values({ ...input, createdBy: 'system', updatedBy: 'system' })
      .returning();
    return result;
  }

  @Mutation(() => User)
  async updateUser(@Arg('input') input: UpdateUserInput) {
    const { id, ...data } = input;
    const [result] = await db.update(users).set(data).where(eq(users.id, id)).returning();
    return result;
  }

  @Mutation(() => User)
  async deleteUser(@Arg('id', () => UserIdScalar) id: UserId) {
    const [result] = await db.delete(users).where(eq(users.id, id)).returning();
    return result;
  }
}
```

Conventions:

- Prefer `db.query.<table>.findMany/findFirst` with relational filters; use `db.select/update/delete` for aggregates and `.returning()` for mutations.
- Any write touching more than one table goes in `db.transaction(async (tx) => ...)`.
- `@FieldResolver` receives the Drizzle row (`@Root() row: XModel`) — use it for computed or loader-backed fields (aggregates, totals, display names) rather than duplicating them in the SQL.
- `@Authorized(UserType.Admin, ...)` guards any field/mutation returning sensitive data (a secret field, another user's row).
- `Get-<Entity>` naming: `users`, `createUser`, `updateUser`, `deleteUser`, args `input` / `id`.

**Types** (`graphql/types/<entity>.type.ts`) — `@ObjectType` classes; every id field uses its scalar; nullable DB columns are `{ nullable: true }` with `?:` props; computed fields declared here are filled by `@FieldResolver`.

**Inputs** (`graphql/inputs/<entity>.input.ts`) — `Create<Entity>Input`, `Update<Entity>Input` (id required, everything else optional/nullable), plus nested inputs for child objects. Decorate with `class-validator` for real constraints (`@IsInt`, `@Min`, `@MaxLength`, `@IsNotEmpty`).

**`graphql/types/success-payload.ts`** — `{ success: boolean }` for mutations that have nothing meaningful to return.

---

## 8. API routes

**`app/api/route.ts`** — authenticated, full resolver set:

```ts
import 'reflect-metadata';
import { createYoga } from 'graphql-yoga';
import { getServerSession } from 'next-auth';
import type { NextRequest } from 'next/server';

import { getSchema } from '@/graphql/schema';
import { authOptions } from '@/lib/auth';

const yoga = createYoga<{ req: NextRequest }>({
  schema: getSchema(),
  graphqlEndpoint: '/api',
  context: async () => {
    const session = await getServerSession(authOptions);
    return { user: session?.user ?? null };
  },
});

export { yoga as GET, yoga as OPTIONS, yoga as POST };
```

**`app/api/public/route.ts`** — a _second_, independently built schema with a deliberately small resolver allow-list and `authChecker: () => true`. This is the surface a separate separate public frontend consumes. Build it lazily inside `schema: () => {...}` so it never runs for the authenticated route. Only ever list resolvers here whose data is genuinely public.

Any file using type-graphql decorators must `import 'reflect-metadata';` first — `app/api/route.ts`, `app/api/public/route.ts`, `graphql/schema.ts`.

---

## 9. Auth

`lib/auth.ts` — Credentials provider that authenticates _through the API_ (one code path, no duplicated credential logic):

```ts
export const authOptions: NextAuthOptions = {
  secret: process.env.AUTH_SECRET,
  session: { maxAge: 60 * 60 },
  cookies: {
    sessionToken: { name: '_<app>', options: { httpOnly: true, sameSite: 'lax', path: '/', secure: true } },
  },
  pages: { signIn: '/login' },
  providers: [
    CredentialsProvider({
      name: 'Secret',
      credentials: { id: { label: 'User ID', type: 'text' }, secret: { label: 'Secret', type: 'password' } },
      async authorize(body) {
        if (!body) return null;
        const { data } = await client().query({
          query: Login,
          variables: { input: { id: body.id as UserId, secret: body.secret } },
        });
        return data?.login ?? null;
      },
    }),
  ],
  callbacks: {
    jwt({ token, user }) {
      /* copy id, fullName, type onto the token */
    },
    session({ session, token }) {
      /* copy them back onto session.user */
    },
  },
};
```

`Login` is a query in `graphql/operations/queries.graphql` selecting only `id fullName type` — never the secret field. Secret comparison (and any hashing) lives in the resolver, not here.

Session cookie is `secure: true`: a local dev server must be reached over HTTPS or the cookie is dropped. Keep the auth secret in `.env`, never inline in source.

---

## 10. App shell & components

**`app/layout.tsx`** (server component)

```
<html lang="en" {...mantineHtmlProps}>
  <head><ColorSchemeScript defaultColorScheme="auto" /></head>
  <body>
    <MantineProvider defaultColorScheme="auto">
      <Notifications position="bottom-right" />
      <Confirm />                       // global confirm modal, mounted once
      <Providers>{children}</Providers>
    </MantineProvider>
  </body>
</html>
```

Also export `metadata: Metadata` (`title`, `description`) from this file.

**`app/global.css`** — import Mantine styles there, never in JSX:

```css
@import '@mantine/core/styles.css';
@import '@mantine/notifications/styles.css';
```

**`components/providers.tsx`** (`'use client'`) — nesting order matters, keep it:

```
ApolloNextAppProvider makeClient={client}
└─ SessionProvider
   └─ Suspense fallback={<LoadingOverlay visible zIndex={1000} />}
```

**`lib/common.state.ts`** — Apollo reactive vars for cross-component UI state (confirm dialog state, current module/mode selection):

```ts
export const commonState = makeVar({ isConfirmOpen: false, confirmMessage: '', onConfirm: async () => {} });
```

**`components/confirm.tsx`** — one global `<Modal>` driven by `commonState`. Destructive actions call `commonState({ isConfirmOpen: true, confirmMessage: '…', onConfirm: async () => {...} })` instead of each page building its own dialog.

**`components/data-table.tsx`** — the _only_ table component. Generic over `T`, wraps `@tanstack/react-table` and Mantine `Table`, and owns: `colgroup` width/align from an `ExtendedColumnDef<T> = ColumnDef<T> & { width?, align?, sortable?, searchable? }`, the global-filter header row, sort icons, row-click + selected-row highlight, and the footer. Exported as `memo(DataTableInner)`. Pages pass Mantine `TableProps` straight through. **Do not build a second table.**

**`components/user.tsx`** — session name + role + sign-out button (`signOut({ callbackUrl: Routes.Login })`), takes `GroupProps`.

**`components/setting-control.tsx`** — key/value settings editor driven by one `Update<Singular>` mutation (`{ key, value }`), dispatching on a discriminated `type: 'text' | 'switch' | 'number' | 'textarea' | 'file' | 'dropdown' | 'array' | 'prints' | 'weekdays'`. Uploads go through `FileButton` → `FileReader.readAsDataURL` and validate mime + size **before** reading. Strings are stored pipe-delimited (`a|b|c`) and arrays/objects as JSON.

---

## 11. Pages

**Role-gated area layout** (`'use client'`, e.g. `app/manage/layout.tsx`):

```tsx
const { data: session, status } = useSession({ required: true });
// on non-permitted role: red notification + router.replace(Routes.Home)
// render <></> while loading and while forbidden
```

**Landing page** (`app/page.tsx`, server component) — `getServerSession(authOptions)`, `redirect(Routes.Login)` when signed out, then a card per module with `disabled={!card.role.includes(user.type)}`.

**Login** (`app/login/page.tsx`) — `useSuspenseQuery` a users-for-login query (id/fullName/type only), grid of avatars → select one → on-screen keypad → `signIn('credentials', { redirect: false, id, secret })` → `router.replace(callbackUrl ?? '/')`.

**CRUD page pattern** (`'use client'`, one per entity, e.g. `app/manage/users/page.tsx`) — this is the template to copy:

1. `useSuspenseQuery(Get<Entity>)`.
2. One `useForm` with the entity's shape; `id: '' as <Entity>Id` in `defaultValues` means "create mode".
3. `useMutation(Create/Update/Delete<Entity>)`, each with an explicit `update(cache, { data })`:
   - create → `cache.modify({ fields: modifiers({ <entities>: [created] }) })`
   - update → `cache.modify({ id: cache.identify(updated), fields: modifiers(updated) })`
   - delete → `cache.evict({ id: cache.identify(deleted) })` + `modifiers({ <entities>: deleted }, 'filter')`
     (`modifiers` from `lib/helpers` handles the replace/append/filter/merge cases; always use it, never hand-write cache writes.)
4. `onCompleted` → green success `notifications.show`, close the modal.
5. `onDelete` → route through `commonState` confirm, never delete on first click.
6. Render: `Group` (Title + Create button) → `<DataTable columns={[...]}/>` → `<Modal>` with a `react-hook-form` `<form onSubmit={handleSubmit(...)}>`; `register` for native inputs, `Controller` for `Select`/`MaskInput`/switches; submit picks create vs update on whether `id` is set.

Import operations and enums from `@/types/gql` by plain name (`import { GetUsers, UserType } from '@/types/gql';`).

**`lib/helpers.ts`** — `modifiers(values, 'filter' | 'replace')` for cache writes, `trimVariables` (recursive, `File`/`Blob`-safe) for the Apollo link, `formatNumber`.

---

## 12. Operations & codegen

`graphql/operations/queries.graphql` and `mutations.graphql` hold every client operation. Naming: `Get<Entity>` / `Get<Entity>For<Purpose>` queries, `Create/Update/Delete<Entity>` mutations. Never select a sensitive field (secrets, password hashes) in an operation unless the resolver guards it with `@Authorized`.

**`app/<app>/codegen.ts`** — introspects the _running_ server, so order matters. `documents` and `generates` paths are relative to the app package, which is the cwd when you run `pnpm codegen` from the root:

```ts
import type { CodegenConfig } from '@graphql-codegen/cli';

const config: CodegenConfig = {
  schema: 'http://localhost:3000/api',
  documents: ['graphql/operations/*.graphql'],
  ignoreNoDocuments: true,
  verbose: true,
  config: {
    scalars: {
      UUID: 'string',
      UserId: './identities#UserId', // one line per id type
      DateTime: 'Date',
    },
    documentVariableSuffix: '', // documents export as `GetUsers`, NOT `GetUsersDocument`
  },
  generates: {
    './types/gql.ts': {
      plugins: ['typescript-operations', 'typed-document-node'],
      config: { enumType: 'native' },
    },
  },
};

export default config;
```

Procedure: **start `pnpm dev` → `pnpm codegen`** → `pnpm typecheck`. Re-run after every schema/resolver/`.graphql` change. If the server is not running, codegen fails or writes against a stale schema — never work around it.

---

## 13. Lint & format

Both tools are **root-only** — installed in the root `package.json`, configured by the root `.oxlintrc.json` / `.oxfmtrc.json`, and run from the root against the app package:

|            | Binary lives in | Config                | Command                           |
| ---------- | --------------- | --------------------- | --------------------------------- |
| **oxlint** | root devDeps    | root `.oxlintrc.json` | `pnpm lint` → `oxlint app/<app>`  |
| **oxfmt**  | root devDeps    | root `.oxfmtrc.json`  | `pnpm format` → `oxfmt app/<app>` |

- **`oxlint`** — the linter. Type-aware, so it catches real bugs (`no-floating-promises`, `no-unsafe-*`, `require-await`) that a syntax-only linter misses. Requires `oxlint-tsgolint` in the **root** devDependencies; without it the type-aware rules silently do nothing. Run `pnpm lint`, fix with `pnpm lint:fix`.
- **`oxfmt`** — the formatter. Non-negotiable: it sorts imports and normalizes quotes/width, which is why every file looks the same. `pnpm format` writes, `pnpm format:check` verifies without writing.

Two traps worth knowing:

- The app package has **no** `lint`/`format` script and no `oxlint` dependency. `pnpm -F <app> lint` fails with "command not found" — that is expected, not a bug. Use the root scripts.
- The linter must resolve the app's `tsconfig.json` for its type-aware rules, so always pass the app directory (`oxlint app/<app>`), never a bare `oxlint .` from the root — the root has no tsconfig and the type rules go quiet.

`scripts/*` and config files are lint-ignored — plain `.mjs`/`.json` with no TS project behind them.

---

## 14. Scripts

Plain `.mjs`, run with `node`. **`tsx` is not installed — do not write `.ts` scripts.**

- `scripts/seed.mjs` — `createClient({ url: process.env.DATABASE_FILE })` from `@libsql/client`, raw `client.execute({ sql, args })`, timestamps as `Math.floor(Date.now() / 1000)` (seconds — `integer({ mode: 'timestamp' })` column), delete-then-insert, and `ON CONFLICT(id) DO NOTHING` for the first user so re-seeding is safe.

---

## 15. Adding an entity — the complete checklist

Every step is required; a missing one fails silently (unregistered table, untyped id, stale `gql.ts`). Paths below are relative to `app/<app>/`; commands run from the workspace root.

1. `types/identities.ts` — add `export type <Entity>Id = Nominal<string, '<Entity>Id'>;`
2. `database/models/<entity>.model.ts` — table (spread `id.$type<EntityId>()`, `...timestamps`, `...auditFields`), relations if needed, `export type <Entity>Model = typeof <table>.$inferSelect;`
3. `database/index.ts` — register it in `schema` (relations as `{ <key>: { <table>, relations } }`).
4. `graphql/scalars.ts` — `export const <Entity>IdScalar = createIdScalar<<Entity>Id>('<Entity>Id');` and `registerEnumType(...)` for any new enum.
5. `codegen.ts` — add `'<Entity>Id': './identities#<Entity>Id'` to `scalars`.
6. `graphql/types/<entity>.type.ts` — `@ObjectType` mirroring the table.
7. `graphql/inputs/<entity>.input.ts` — `Create<Entity>Input`, `Update<Entity>Input`, nested inputs.
8. `graphql/resolvers/<entity>.resolver.ts` — `@Resolver(() => <Entity>)` with queries/mutations, `@Authorized` on sensitive fields, transactions for multi-table writes.
9. `graphql/schema.ts` — add the resolver to the `resolvers` array. Add it to `app/api/public/route.ts`'s list only if it is public.
10. `graphql/loaders.ts` — a DataLoader for each relation resolved on a list.
11. `types/enums.ts` — new enums + route paths.
12. `graphql/operations/{queries,mutations}.graphql` — the client operations.
13. `pnpm db:push`.
14. `pnpm dev` in one shell → `pnpm codegen` in another.
15. `app/<module>/<entity>/page.tsx` - copy the CRUD page pattern, using `<DataTable>`.
16. `pnpm format`, then `pnpm typecheck && pnpm lint && pnpm format:check && pnpm build` (or just `pnpm verify`).

---

## 16. AGENTS.md (commit this with the project)

The root `AGENTS.md` must restate, briefly: pnpm only; no new packages without approval (name, latest version, weekly downloads, why, alternatives); **the monorepo layout** — tooling (`oxlint`, `oxfmt`) lives only at the root, the Next app and all its deps live in `app/<app>/`, all lint/format/verify commands run from the root; stack overview; the §15 checklist; dev server on port 3000 and `pnpm codegen` introspects the running server; apply schema changes with `pnpm db:push`, never `generate`/`migrate`; `app/<app>/.env` needs `DATABASE_FILE`, `AUTH_SECRET`, `AUTH_URL`; verification is `pnpm verify`.

---

## 17. Scaffolding order

0. **Ask** the §0.5 questions. Do not start before you have the folder name and the description.
1. Create the root folder `<root>/` and `app/<root>/`. Write root `package.json` (name from answer 1, description from answer 2), root `pnpm-workspace.yaml` (`packages: ['app/*']`), root `.gitignore`, root `AGENTS.md`.
2. Write the app's `package.json`, then `pnpm install` **from the root**.
3. From the root, install the tooling: `pnpm add -D oxlint oxfmt oxlint-tsgolint`.
4. From `app/<app>/`, install every app dependency + app devDependency at latest.
5. Write root `.oxlintrc.json` and `.oxfmtrc.json` (copy §4 verbatim).
6. Write the app configs: `tsconfig.json`, `next.config.ts`, `postcss.config.mjs`, `drizzle.config.ts`, `.env`.
7. `app/<app>/types/` — identities, enums, next-auth.d.ts.
8. `app/<app>/database/` — base.model, user.model, index.
9. `app/<app>/graphql/` — scalars, decorators, request-context, schema, client, loaders, user type/input/resolver, success-payload.
10. `app/<app>/lib/` — auth, common.state, helpers.
11. `app/<app>/app/api/route.ts`, `app/api/public/route.ts`, `app/layout.tsx`, `app/global.css`.
12. `app/<app>/components/` — providers, confirm, data-table, user, setting-control.
13. `app/<app>/app/page.tsx`, `login/page.tsx`, `manage/layout.tsx`, `manage/users/page.tsx`.
14. `app/<app>/scripts/seed.mjs`, `README.md`.
15. `pnpm db:push` → `pnpm db:seed`.
16. `pnpm dev`, then `pnpm codegen`.
17. `pnpm verify`.
