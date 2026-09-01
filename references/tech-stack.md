# Tech Stack

Part of `renan-frontend-architecture`. This file answers *which library*; the rest of
the skill answers how to structure and write it.

Defaults for bootstrapping or extending a project. In an **existing** repo, match what
it already uses before falling back to these — never introduce a second library that
does a job one already installed does.

## Bootstrap defaults

| Concern | Default | Notes |
|---|---|---|
| Language | TypeScript | always |
| Framework | Vite + React | Next.js only if the user asks for SSR, SSG, or server components |
| Styling | Tailwind CSS | |
| Components | shadcn/ui | skill `shadcn` |
| Routing | react-router-dom | skip for Next.js — use its file-based routing |
| Validation | Zod | skill `zod`; schemas and derived types in `domain.md` |
| Forms | React Hook Form | pair with Zod via `@hookform/resolvers` |
| Data fetching | TanStack Query | patterns in `queries.md` and `mutations.md` |
| HTTP client | Axios | transport for the repository, never called from a hook — `domain.md` |
| Global state | React Context API | Zustand only if the user explicitly names an external state library |
| Utilities | lodash | |

## Ask before picking

Two valid options each — stop and ask before scaffolding:

- **Charts** — Highcharts or Recharts?
- **Tables/grids** — AG Grid or TanStack Table? (skill `tanstack-table` if TanStack)

## Tailwind: use the scale

Reach for bracket syntax only when the design genuinely has no scale equivalent. A
one-off value that keeps reappearing is a signal it belongs in `@theme` as a named
token, not that the bracket syntax is fine.

```
wrong: text-[15px]  text-[22px]  size-[18px]  hover:bg-black/[0.02]
right: text-sm      text-xl      size-5       hover:bg-ink/5
```

Same for colour: define semantic tokens in `@theme` (`--color-ink`, `--color-line`)
and use `text-ink` / `border-line`. Never inline a hex value.

## Reference docs

| Library | Docs | Installed skill |
|---|---|---|
| React | react.dev | — |
| Vite | vitejs.dev | — |
| Next.js | nextjs.org/docs | `vercel:nextjs` |
| TypeScript | typescriptlang.org/docs | — |
| Tailwind CSS | tailwindcss.com/docs | — |
| shadcn/ui | ui.shadcn.com | `shadcn` |
| Zod | zod.dev | `zod` |
| TanStack Query | tanstack.com/query | `tanstack-query` |
| TanStack Virtual | tanstack.com/virtual | `tanstack-virtual` |
| TanStack Table | tanstack.com/table | `tanstack-table` |
| React Hook Form | react-hook-form.com | — |
| Axios | axios-http.com | — |
| React Router | reactrouter.com | — |
| Zustand | github.com/pmndrs/zustand | — |
| lodash | lodash.com/docs | — |
| Highcharts | highcharts.com/docs | — |
| Recharts | recharts.org | — |
| AG Grid | ag-grid.com/react-data-grid | — |
