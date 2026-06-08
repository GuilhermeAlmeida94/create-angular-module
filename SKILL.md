---
name: create-angular-module
description: >
  Creates a new lazy-loaded Angular application inside an existing Angular workspace.
  Use this skill whenever the user wants to add a new app, sub-application, or grouped
  feature to an Angular project — even if they say "new section", "new module", or
  "add an app" without mentioning lazy loading or monorepo.
---

# Create Angular Module

A skill to scaffold a new Angular application inside an existing workspace, configure it
with a scoped package name, and wire it as a lazy-loaded route in the main app.

## Prerequisites

Before starting, verify:
- Node.js 18+ is installed (`node -v`)
- Angular CLI 20+ is installed (`ng version`)
- The current or parent directory contains `angular.json` (Angular workspace root)

## Step 1 — Locate the Angular workspace

Search for `angular.json` starting at the provided path or the current directory,
then walk up parent directories. If not found, stop and tell the user:
> "No Angular workspace found. Run this skill from inside an Angular project."

Read `angular.json` and `package.json` to extract:
- The workspace name (used to derive the default scope)
- Existing project names (to avoid collisions)

## Step 2 — Gather inputs

Ask the user for:

1. **App name** (required) — e.g., `admin`, `dashboard`, `reports`
   - Must be lowercase, no spaces, no leading numbers
   - Must not already exist in `angular.json` projects
2. **Scope** (optional, default derived from workspace name) — e.g., `@poupa-guara`
   - The full scoped package name will be `@<scope>/<app-name>`
3. **Wire lazy route?** (default: yes) — whether to add a lazy-loaded route to the
   main app's `app.routes.ts`

If all were provided as arguments, skip prompting and proceed.

## Step 3 — Generate the app

From the workspace root, run:

```bash
ng generate application <app-name> --style=scss
```

> Use `--style=scss` to match the main app convention. If the main app uses a different
> style format, match it. Angular 20+ generates `app.routes.ts` by default.

This creates `projects/<app-name>/` and registers the app in `angular.json`.
It also adds references to `projects/<app-name>/tsconfig.*.json` inside the root
`tsconfig.json` — **remove those references** after generation (see note below).

> **TypeScript 6 + project references:** Angular 22 uses TypeScript 6, which crashes
> (`Cannot destructure property 'pos'`) when the root `tsconfig.json` references
> sub-project tsconfigs. Remove the added references from the root `tsconfig.json`
> after every `ng generate application`. The per-project tsconfig files stay — only
> remove the entries inside the root `"references": [...]` array.

## Step 4 — Add root route to the new app

Update `projects/<app-name>/src/app/app.routes.ts` to define a root route that renders
the app's `App` component. Without this, Angular loads the lazy chunk but finds no
matching route and renders nothing:

```typescript
import { Routes } from '@angular/router';
import { App } from './app';

export const routes: Routes = [
  { path: '', component: App },
];
```

## Step 5 — Create public entry point and scoped package name

Angular CLI does not generate a `package.json` or `index.ts` for application projects.
Create both manually:

**`projects/<app-name>/package.json`**
```json
{ "name": "@<scope>/<app-name>" }
```

**`projects/<app-name>/src/index.ts`**
```typescript
export { routes } from './app/app.routes';
```

This `index.ts` is the public API for the lazy-loaded app.

## Step 6 — Register path alias in tsconfig.app.json

In the main app's `tsconfig.app.json`, add a `paths` entry (TypeScript 6 style —
no `baseUrl` needed; use relative paths from the workspace root):

```json
{
  "compilerOptions": {
    "paths": {
      "@<scope>/<app-name>": ["./projects/<app-name>/src/index.ts"]
    },
    "types": []
  }
}
```

> Do NOT add `baseUrl` — it is deprecated in TypeScript 6 and causes a build error.
> Do NOT remove `rootDir` from `tsconfig.app.json` if it was already absent; if it
> was present, remove it since it conflicts with resolving paths outside `./src`.

## Step 7 — Wire lazy route (if requested)

In the main app's route file (default: `src/app/app.routes.ts`), add:

```typescript
{
  path: '<app-name>',
  loadChildren: () => import('@<scope>/<app-name>').then(m => m.routes),
}
```

If the routes array is empty, replace it. If it already has routes, append.

## Step 8 — Report success

Print a summary:

```
✓ Created app: @<scope>/<app-name>
  Location:    projects/<app-name>/

Commands:
  Standalone dev server:     ng serve --project=<app-name>
  With main app (port 4200): npm start  →  http://localhost:4200/<app-name>
  Build only this app:       ng build --project=<app-name>
```

## Edge Cases

- **App name collision:** If the name already exists in `angular.json`, stop and suggest
  a different name.
- **Invalid name:** If the name contains uppercase, spaces, or special characters,
  sanitize to kebab-case and confirm with the user before proceeding.
- **No routes file found:** If `src/app/app.routes.ts` doesn't exist, warn the user and
  skip route wiring. Tell them what to add manually.
- **Dry-run mode:** If the user says "what would this do" or "preview", describe each
  step without executing any commands.
