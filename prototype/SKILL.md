---
name: prototype
description: Build complete UI-only prototype screens from requirements, with a mandatory preflight gate. Manual invocation only.
metadata:
  opencode/autoinvoke: false
disable-model-invocation: true
---

# Prototype — UI-only screen generator

Build complete, realistic screens that render entirely from static data, with **zero data logic**.

## 0. Invocation contract

This skill is **manual-only**.

- Run it only when the user explicitly invokes it with `/prototype` or `@prototype`.
- Never load it because a prompt contains "screen", "page", "UI", "mockup", "prototype", "dashboard", or similar. Those words are not an invocation.
- Never suggest this workflow as the default. If the user asks for UI work without invoking it, do normal work; you may mention `/prototype` as an option, once.
- Once invoked, follow this file exactly. No shortcuts, no skipped phases.

---

## 1. Contract

Produce screens that:

- look complete and production-ready,
- render with hardcoded static data,
- contain no fetching, persistence, authentication, or business logic,
- live under a single `prototypes/` folder,
- are reachable as preview routes,
- are documented in `project.md`, `prototype.md`, and `design.md`.

Nothing is written until the preflight in Phase 2 is approved.

---

## 2. Hard rules

| #   | Rule                                                                                                                                                                                                                          |
| --- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| R1  | **UI only.** Banned: `fetch`, `axios`, `ky`, GraphQL/Apollo, tRPC, React Query/SWR, server actions, API/route handlers, DB/ORM, auth, cookies, `localStorage`/`sessionStorage`, websockets/SSE, env vars, secrets, real validation. |
| R2  | **Static mock data only.** Declare it as a local `const` at the top of the same screen file. No JSON files, no fixtures module, no cross-screen mock imports.                                                                   |
| R3  | **Pure UI state is allowed.** `useState`/`useReducer` for tabs, modals, drawers, dropdowns, toggles, selection, hover. Data state is not allowed.                                                                              |
| R4  | **One `prototypes/` folder.** The only file outside it you may edit is the router registration, and only after showing the diff and getting approval.                                                                          |
| R5  | **Never study existing app UI.** Do not read existing components, pages, styles, or design-system files to copy patterns. Design is generated fresh, from scratch.                                                              |
| R6  | **Ask before every package install.** Show name, latest version, why, and the alternative. Install only after an explicit yes.                                                                                                 |
| R7  | **Ask for the UI library if none is installed.** Never assume one.                                                                                                                                                            |
| R8  | **Ask for styling if none is detected.** Never assume.                                                                                                                                                                        |
| R9  | **Ask design direction** unless `design.md` already answers it.                                                                                                                                                               |
| R10 | **Confirm the screen list only when the request is ambiguous.** Otherwise take the requirements as written.                                                                                                                    |
| R11 | **Ask before overwriting** an existing screen folder. Never silently replace work.                                                                                                                                            |
| R12 | **Match the language of the request** for UI copy. Code identifiers, comments, and file names stay English.                                                                                                                    |
| R13 | **No stub screens.** Every screen renders complete, realistic UI, including visual empty/loading/error states where relevant.                                                                                                  |
| R14 | **Verify with typecheck/lint only.** Never start a dev server or run a production build unless asked.                                                                                                                          |
| R15 | **Never delete prototype files** without asking.                                                                                                                                                                              |

### Allowed reads

`package.json`, lockfile, `tsconfig*.json`, framework config, router files, `README.md`, `docs/*.md`, and `prototypes/*.md`.

### Forbidden reads for design

Any existing component, page, layout, CSS, theme, or design-system file. You are starting from scratch, not matching the app.

---

## 3. Phase 0 — Load existing state

Before anything else, check the common prototype locations and read what exists:

1. `<root>/prototypes/project.md`
2. `<root>/prototypes/prototype.md`
3. `<root>/prototypes/design.md`
4. `<root>/src/prototypes/...`
5. `<root>/app/prototypes/...`

If a file exists, treat it as the source of truth and ask only about gaps. Never re-ask a decision that is already recorded.

---

## 4. Phase 0.5 — Project understanding

### Sources, in order

1. `package.json` — `name`, `description`, `keywords`, dependencies.
2. `README.md` — purpose, audience, features, terminology.
3. `docs/*.md` referenced by the README.
4. `prototypes/project.md` if it already exists — reuse it.

Do **not** read application source code for this.

### Completeness checklist

You must be able to answer every row before continuing:

| Area        | Must know                                              |
| ----------- | ------------------------------------------------------ |
| What        | One-paragraph description of the product               |
| Why         | Problem it solves / value proposition                  |
| Who         | Target users, personas, roles                          |
| Domain      | Core entities and how they relate                      |
| Journeys    | The 3–5 key user flows                                 |
| Screens     | What exists today vs. what is being requested          |
| Terms       | Domain vocabulary and its meaning                      |
| Constraints | Platform, locale, brand, regulatory limits             |
| Out of scope| What this prototype explicitly is not                  |

### Grilling protocol

- Ask only about missing rows.
- Maximum 5 questions per message.
- Prefer concrete options with a recommended choice; allow free-form answers.
- Keep asking in follow-up batches until every row is filled.
- Then restate the understanding in **5–8 bullets** and ask for confirmation.
- If a row cannot be answered, record it as `TBD` and confirm the user is okay proceeding.

**Do not start Phase 1 until the understanding is confirmed.**

### Save

Write or update `prototypes/project.md` using the template in §8.

---

## 5. Phase 1 — Detect the environment

Read only config files and `package.json`. Determine:

- framework and router type,
- app root,
- `prototypes/` directory,
- screen file convention,
- index route convention,
- route registration strategy,
- TypeScript vs JavaScript,
- installed UI libraries,
- installed styling systems,
- package manager (from the lockfile).

### Framework → paths

| Signal                                          | `prototypes/` directory      | Screen file                                  | Index file                                    | Routing         |
| ----------------------------------------------- | ---------------------------- | -------------------------------------------- | --------------------------------------------- | --------------- |
| `next` + `app/` or `src/app/`                   | `<appDir>/prototypes`        | `<dir>/<kebab>/page.tsx`                     | `<dir>/page.tsx`                              | automatic       |
| `next` + `pages/` or `src/pages/`               | `<pagesDir>/prototypes`      | `<dir>/<kebab>.tsx`                          | `<dir>/index.tsx`                             | automatic       |
| `vite` / `react-scripts` + `react-router-dom`   | `<srcDir>/prototypes`        | `<dir>/<kebab>/<Pascal>.tsx`                 | `<dir>/PrototypeIndex.tsx`                    | router edit     |
| `nuxt`                                          | `<pagesDir>/prototypes`      | `<dir>/<kebab>.vue`                          | `<dir>/index.vue`                             | automatic       |
| `@sveltejs/kit`                                 | `<routesDir>/prototypes`     | `<dir>/<kebab>/+page.svelte`                 | `<dir>/+page.svelte`                          | automatic       |
| `vue` + `vue-router`                            | `<srcDir>/prototypes`        | `<dir>/<kebab>/<Pascal>.vue`                 | `<dir>/PrototypeIndex.vue`                    | router edit     |
| anything else                                   | **stop and ask**             | **stop and ask**                             | **stop and ask**                              | **stop and ask**|

### App root fallback

Use `app/` if present, else `src/`, else the repository root. For a framework with a dedicated router directory (`pages/`, `routes/`), use that directory instead.

### Shared components

`<prototypesDir>/_shared/`. If the framework would expose `_shared` as a route, stop and ask for a safe location.

### Unknown framework

Stop and ask:

1. What framework and router are in use?
2. Which directory should hold routable pages?
3. How are new routes registered?

---

## 6. Phase 2 — Preflight

Nothing is written before every step is complete and the plan is approved.

### Step 1 — Requirements (only if ambiguous)

Restate a numbered list: screen name, purpose, key regions. Ask for confirmation.

### Step 2 — UI library

Scan dependencies for: `@mantine/core`, `@mui/material`, `@chakra-ui/react`, `antd`, `primereact`, `@nextui-org/react`, `@heroui/react`, `daisyui`, `tremor`, `@radix-ui/*` + `class-variance-authority` + `components.json` (shadcn/ui).

- Exactly one installed → state it and use it.
- Several installed → ask which.
- None installed → ask:

  1. Mantine
  2. Material UI
  3. shadcn/ui
  4. Chakra UI
  5. Tailwind-only, no component library
  6. Other

### Step 3 — Styling

Detect Tailwind, CSS Modules, styled-components/emotion, vanilla CSS, or library-native styling.

- Exactly one → state it and use it.
- Several → ask which.
- None → **ask, never assume.**

### Step 4 — Design direction

Ask for: accent color(s), light/dark/both, density, corner radius, font, and reference screenshots or URLs.

If `design.md` already has these, show them and ask only for changes.

### Step 5 — Packages

List every package required: UI library, icons, styling dependencies, class utilities.

For each one show: `name@latest`, why it is needed, the alternative, and the exact install command.

Ask for approval **per package**. Do not install anything unapproved. If refused, adapt using what is installed or plain CSS/HTML. Never silently substitute a different library.

### Step 6 — Plan summary

Show:

- exact directory tree,
- every file to create or update,
- the router diff, if any,
- which docs will be written.

If a target folder already exists, ask: overwrite, rename, or skip.

Wait for an explicit **yes** before writing.

---

## 7. Phase 3 — Generate

For each screen:

1. Create the folder.
2. Write one screen file, plus a styles file only if the styling system requires it.
3. Put static mock data in a local `const` at the top of that file.
4. Build the complete UI.

### Screen quality checklist

- All regions from the requirements: header, nav, toolbar, content, side panels, footer.
- Realistic copy in the request's language. No lorem ipsum unless asked.
- Lists have 5–10 realistic rows. Tables have headers, alignment, and row actions.
- Forms have labels, placeholders, help text, required marks, disabled/readonly examples, and a submit button.
- Visual states where relevant: empty, loading (skeleton), error — rendered statically.
- Responsive and usable at 375px, 768px, and 1280px.
- Accessible: semantic landmarks, one `h1`, labels tied to inputs, alt text, visible focus, `aria-label` on icon buttons, sufficient contrast.
- Dark mode if chosen.
- UI-library idioms used correctly (Mantine props, MUI `sx`, shadcn primitives, etc.).

### Allowed

- Local `const` arrays/objects as mock data.
- `useState` for UI-only interactions.
- `onClick` that only changes UI state.
- `preventDefault` on forms.

### Banned

- Any network call or data client.
- Server actions, route handlers, API routes.
- DB, ORM, auth, cookies, storage, env vars.
- Redux, Zustand, Jotai, Recoil, XState.
- Real validation, submission, or upload.
- Data loaders/actions in the router.
- `useEffect` used to simulate loading data.

If a screen needs to look data-driven, use the local `const`. If the user asks for real data, use the refusal playbook in §10.

### Shared components

If two or more screens need the same piece, extract it to `<prototypesDir>/_shared/<Pascal>.tsx` (or the framework's equivalent). Shared components follow the same UI-only rules and may import only the framework, the UI library, and the styling system.

### Index route

Maintain a static index at the location from §5. It lists every screen as `{ name, path, description }` and is updated on every run.

### Router registration

Only for frameworks that do not auto-route.

1. Show the minimal diff.
2. Wait for approval.
3. Apply only the added lines. Do not reformat the file.

### Language

UI copy follows the request language. Code stays English.

---

## 8. Documentation

Create or update these files on every run.

### `prototypes/project.md`

```md
# Project Understanding

_Last updated: YYYY-MM-DD_

## What

## Why

## Who

## Domain model

| Entity | Meaning | Key fields | Relations |
| ------ | ------- | ---------- | --------- |

## Key journeys

## Screen inventory

| Screen | Route | Status | Source |
| ------ | ----- | ------ | ------ |

## Terminology

## Constraints

## Out of scope

## TBD
```

### `prototypes/prototype.md`

```md
# Prototype Rules

_Last updated: YYYY-MM-DD_

## Environment

- Framework:
- Router:
- App root:
- Prototypes directory:
- Language:

## UI

- UI library:
- Styling:
- Icons:

## Conventions

- Screen folder:
- File naming:
- Shared components:
- Route registration:

## Approved packages

| Package | Version | Why | Date |
| ------- | ------- | --- | ---- |

## Hard rules

R1–R15 from the `prototype` skill. UI only, static mock data, one `prototypes/` folder, no existing-UI pattern copying, package approval, library and styling questions, overwrite approval, request language, no stubs, typecheck/lint only.

## Index

- Route:
```

### `prototypes/design.md`

```md
# Design Decisions

_Last updated: YYYY-MM-DD_

## Direction

- Mood:
- References:

## Tokens

| Token | Light | Dark |
| ----- | ----- | ---- |
| Background | | |
| Surface | | |
| Text | | |
| Muted text | | |
| Accent | | |
| Border | | |
| Radius | | |
| Shadow | | |

## Typography

## Layout patterns

## Screen decisions

### <screen-name>

- Purpose:
- Layout:
- Components:
- Notes:
```

---

## 9. Phase 4 — Verify

- Detect the project's `typecheck`, `lint`, and `format:check` scripts.
- Run only the ones that exist, from the correct package directory.
- Fix errors in prototype files only.
- Report pre-existing errors outside `prototypes/`; do not fix them.
- Never start a dev server. Never run a production build unless asked.

---

## 10. Refusal playbook

| Request | Response |
| ------- | -------- |
| "Connect this screen to the API" | Refuse. This skill is UI-only. Offer the static screen and note what wiring would be needed later. |
| "Fetch real data" | Refuse. Use a local `const` with realistic data. |
| "Add form validation" | Render validation visuals only (error text, disabled states). No logic. |
| "Store this in the database" | Refuse. Out of scope. |
| "Add authentication" | Render a static login/profile screen only. No auth logic. |
| "Install this package" | Run the package approval flow in Phase 2, Step 5. |
| "Match the existing design" | Refuse to read existing UI. Ask for brand tokens or references instead. |

---

## 11. Anti-patterns

- Asking for a UI library when one is already installed.
- Reading existing components or styles.
- Creating `mock.ts`, `data.json`, or fixtures.
- Adding `useEffect` to fake loading.
- Writing anywhere outside `prototypes/` without approval.
- Reformatting the router file.
- Producing a skeleton instead of a complete screen.
- Re-asking decisions already recorded in the docs.
- Installing a package without explicit approval.

---

## 12. Completion report

When finished, report:

- screens created and their paths,
- routes,
- shared components,
- docs updated,
- packages installed (approved only),
- verification results,
- open `TBD` items.
