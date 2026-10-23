---
name: react-frontend
description: Enterprise standards for React + TypeScript + Vite single-page apps built with npm - feature-based folders, typed API client, TanStack Query for server state, React Router, React Hook Form + Zod forms, Tailwind styling, accessibility, and loading/empty/error states. Use when generating or modifying any frontend code.
---

# React Frontend Standards

Code patterns to copy are in [references/code-patterns.md](references/code-patterns.md).

## Stack (install with npm)

| Concern | Library |
|---|---|
| Build | Vite + TypeScript (`strict: true`) |
| Routing | `react-router-dom` |
| Server state | `@tanstack/react-query` |
| Forms | `react-hook-form` + `zod` + `@hookform/resolvers` |
| Styling | Tailwind CSS (unless the design doc names a component library) |
| Types from API | `openapi-typescript` (see `api-contract`) |
| Lint/format | ESLint (Vite's default config) + Prettier |
| Tests | Vitest + Testing Library + MSW (see `testing-quality`) |

## Folder structure: by feature

```
frontend/src/
├── main.tsx
├── app/
│   ├── App.tsx               providers: QueryClient, Auth, Router
│   ├── routes.tsx            route table, ProtectedRoute wrappers
│   └── layout/               AppShell, NavBar, Sidebar
├── features/
│   ├── auth/                 LoginPage, AuthContext, useAuth
│   └── products/
│       ├── api.ts            typed calls: listProducts, createProduct...
│       ├── hooks.ts          useProducts, useCreateProduct (TanStack Query)
│       ├── schemas.ts        zod schemas for forms
│       ├── ProductListPage.tsx
│       ├── ProductFormPage.tsx
│       └── components/       ProductTable, ProductForm
└── shared/
    ├── api/                  http.ts, ApiError, schema.d.ts (generated)
    ├── components/           Button, Input, Table, Pagination, ConfirmDialog, EmptyState, ErrorState, Spinner
    └── lib/                  formatters (money, dates)
```

A feature imports from `shared/` and its own folder. It never imports from another
feature's internals. If two features need the same thing, move it to `shared/`.

## Rules

- **Components:** function components only, named exports, one component per file,
  props typed with an interface. Keep files under ~150 lines and extract when they grow.
- **Server data lives in TanStack Query, not `useState`.** Query keys look like
  `['products', params]`. Mutations invalidate the related keys on success.
- **Every data view handles four states:** loading (skeleton/spinner), error (message +
  retry), empty (helpful message + primary action), and success.
- **Forms:** a zod schema is the source of validation. Show field errors inline. Map the
  backend's `errors[]` from a 400 response onto fields with `setError`. Disable submit
  while a request is pending.
- **Config:** the API base URL is `import.meta.env.VITE_API_BASE_URL`, defaulting to `''`
  (same origin, proxied in Docker and by the Vite dev proxy). No other URLs are hard-coded.
- **Dev proxy:** `vite.config.ts` proxies `/api` to `http://localhost:8080` so `npm run dev`
  works without CORS issues.
- **Auth:** see `app-security`. `ProtectedRoute` redirects to `/login`. Role-gated routes
  check `user.roles`.
- **Accessibility:** semantic HTML (`button`, `nav`, `main`, `table`), a `label` for every
  input, visible focus states, AA color contrast, and keyboard-operable dialogs.
- **Formatting:** money with `Intl.NumberFormat`, dates with `Intl.DateTimeFormat`, both
  in the user's locale.
- **No `any`.** Use `unknown` and narrow. No `console.log` in committed code.

## npm scripts (package.json)

```json
{
  "dev": "vite",
  "build": "tsc -b && vite build",
  "lint": "eslint .",
  "format": "prettier --write src",
  "test": "vitest run",
  "test:watch": "vitest",
  "gen:api": "openapi-typescript http://localhost:8080/v3/api-docs -o src/shared/api/schema.d.ts"
}
```

## Definition of done for each feature

- [ ] Pages reachable from the navigation, and role gating matches `docs/plan.md`
- [ ] All four data states implemented
- [ ] Forms validate on the client and show server errors
- [ ] `npm run lint`, `npm test` and `npm run build` all pass
