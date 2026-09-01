---
name: renan-frontend-architecture
description: >
  Renan's React frontend architecture and stack defaults — the
  containers/components/domain/queries/mutations/lib/hooks structure, TanStack Query
  hooks over zod-parsing repositories, entities, enums, suspense with skeleton
  fallbacks, absolute imports, and types derived from schemas. Use when creating a feature, writing or reviewing
  a data-fetching hook or API call, deciding where a file goes or what to name it,
  picking a library, or bootstrapping a project — including bare asks like "fetch
  this", "add a mutation", "add a table", "add global state", "set up a new
  project", "add a loading state", "add filters to this list", or "why is this
  refetching". Also when a cast, an `any`, a relative
  import, or a schema shape written twice shows up. The `tanstack-query` skill has
  the raw API reference; this is how to structure and write the layer.
---

# Renan Frontend Architecture

```
container  →  query hook  →  repository  →  http  →  API
   ↓          key, cache      parse params ↑  ↓ parse response
components (props only)       select → entity
```

Each layer owns exactly one thing. The container fetches and decides. The hook owns
the key and the caching. The repository owns the URL and the schemas. Components take
props and nothing else.

Reads are `useSuspenseQuery`: the container is a shell plus a suspending child, and
every loading region shows a skeleton shaped like the content it replaces, never a
spinner. `containers.md` has the shape. Nobody catches errors along the way — a `ZodError` from the
repository or an `AxiosError` from axios travels untouched to the hook, which is why
a query's error type is `AxiosError | ZodError`.

**Match the repo first.** Read a neighboring feature and copy its layout; a repo's own
`AGENTS.md` wins over this skill, and imposing a second convention is worse than none.

## Where to read next

This page holds what applies everywhere: placement, naming, imports, typing,
boundaries. The patterns live in `references/`. **Read the one file for the layer
you're touching**, not the others — each stands alone.

| Working on                                                                   | Read                          |
| ---------------------------------------------------------------------------- | ----------------------------- |
| `useQuery`, query keys, `staleTime`, `select`, loading flags, infinite lists | `references/queries.md`       |
| `useMutation`, optimistic updates, rollback, invalidation                    | `references/mutations.md`     |
| enums, zod schemas, entities, repositories, any API call                     | `references/domain.md`        |
| containers, `Suspense`, skeletons, error boundaries, filters, composing a screen | `references/containers.md` |
| an `AGENTS.md` or `CONTEXT.md` for a feature                                 | `references/context-files.md` |
| picking a library, bootstrapping a project, Tailwind values                  | `references/tech-stack.md`    |
| path aliases, `tsconfig` paths, bundler alias, import lint rules             | `references/imports.md`       |

A task spanning two layers reads both files. Most tasks span one.

## The shape

```
src/
├── containers/                  # fetch data, compose, own boundaries
│   └── user-profile/            # folder names it, index.tsx is the file
│       ├── components/          # same folders, minus containers
│       │   └── user-summary/
│       │       ├── index.tsx
│       │       └── user-summary.test.tsx
│       ├── domain/
│       │   ├── entities/
│       │   │   └── user.ts
│       │   ├── repositories/
│       │   │   └── user.ts
│       │   ├── schemas/
│       │   │   └── user.ts
│       │   └── enums/
│       │       └── user-status.ts
│       ├── queries/
│       │   └── index.ts         # every query for this domain
│       ├── mutations/
│       │   └── index.ts         # every mutation for this domain
│       ├── lib/
│       │   ├── format-plate.ts
│       │   └── format-plate.test.ts
│       ├── hooks/
│       │   └── use-plate-input.ts
│       └── index.tsx
├── components/                  # global, presentational, never fetch
├── domain/                      # one folder per kind: entities, repositories,
│                                #   schemas, enums; loose rules stay at the root
├── queries/
├── mutations/
├── lib/                         # http client, generic helpers
└── hooks/                       # generic React hooks
```

A feature container repeats the same folders **minus `containers`** — containers
don't nest. Everything starts inside the feature that needs it and moves up to the
root only when a _second_ feature needs it. Promoting early is how a shared folder
turns into a junk drawer, so let the second use case prove the abstraction.

## Boundaries

| Folder        | Holds                                                                      | Never                                                           |
| ------------- | -------------------------------------------------------------------------- | --------------------------------------------------------------- |
| `containers/` | feature roots: fetch, decide, compose, own `Suspense` and error boundaries | gets imported by `components/`                                  |
| `components/…-skeleton/` | the fallback shapes those boundaries render while data loads   | a bare spinner standing in for content                          |
| `components/` | presentational, prop-driven, reusable                                      | imports anything from `queries/`, `mutations/`, or a repository |
| `domain/`     | enums, zod schemas, entities, repositories                                 | imports React                                                   |
| `queries/`    | one `index.ts`: every `useQuery` / `useSuspenseQuery` for the domain      | contains fetch logic — that's the repository                    |
| `mutations/`  | one `index.ts`: every `useMutation` and its invalidation                  | writes to the cache without a rollback path                     |
| `lib/`        | the http client, formatters, generic utilities                             | knows about a feature                                           |
| `hooks/`      | generic React hooks (`useDebounce`, `useMediaQuery`)                       | fetches data                                                    |

### Components never fetch

The rule the rest of the architecture hangs off, and it has no exceptions: **a file
under `components/` imports nothing from `queries/`, `mutations/`, or a repository.**
It receives everything it renders as props. The container does the fetching and the
deciding — it calls the hooks, derives what the screen needs, handles the mutations,
and hands each component a plain prop.

**What "logic" means here.** Logic deciding _what_ is shown belongs to the container:
fetching, deriving, branching on server state, coordinating mutations. Logic deciding
_how_ it looks stays in the component: a conditional class, a formatted string,
sorting the list it was handed. Reusable calculations go in `lib/`, behavior tied to
one record goes on the entity — neither is a reason to move a fetch downward.

**What to pass.** A feature-local component can take the entity itself; one prop
beats eight and it already belongs to that domain. A component in root `components/`
takes primitives, because the point of it living there is that it doesn't know your
domain.

The check: **if you can't render it from its props alone, it's a container in the
wrong folder** — and it can no longer be dropped into a test, a story, or another
feature.

## File naming

- **Components and containers are folders.** The folder carries the name, the file is
  `index.tsx`, so `components/user-card/index.tsx` imports as `@/components/user-card`
  and the folder has room for the test plus anything private to it.
- **`queries/` and `mutations/` are one `index.ts` each**, holding every hook for that
  domain as a named export. One file per hook scatters a domain across a dozen
  near-identical modules and hides the query keys from each other; the aggregate keeps
  the keys that partition one cache visible in one place.
- **Each kind inside `domain/` gets its own folder**, and the folder carries the
  kind so the filename doesn't repeat it: `domain/entities/user.ts`,
  `domain/repositories/user.ts`, `domain/schemas/user.ts`, `domain/enums/fuel.ts` —
  never `domain/user.entity.ts`. One import path names the kind and the subject, and
  every entity sits next to every other entity instead of interleaved with the
  repositories. Rules and label maps that aren't one of those kinds stay directly in
  `domain/` (`labels.ts`, `is-miss.ts`); don't invent a folder to hold one file.
- **Everything else is a file named after itself** — `hooks/use-debounce.ts`,
  `lib/format-plate.ts`. A folder around one export is ceremony.
- **Tests sit beside what they test**, never in a central `__tests__` or mirrored
  `tests/` tree — a test in a parallel tree rots the first time a file moves.
- **Test files take the subject's name, not `index`** — `user-card/user-card.test.tsx`.
  Runner output and editor tabs are unreadable when every file is `index`.

## Imports

**Always absolute, through the alias.** `@/components/user-card`, never
`../../components/user-card`. A relative path encodes where the _importing_ file
happens to sit, so it breaks on every move and names no layer. An aliased path is the
same string from everywhere, which also makes the boundary rules greppable: a
`@/queries/` import inside `components/` is a visible violation.

Colocated tests use the alias too — `user-card.test.tsx` imports
`@/components/user-card`, not `./index`. The rule is enforced by lint, not goodwill;
alias and lint wiring is in `references/imports.md`.

## Typing

**Derive types from schemas; never write the same shape twice.**

```ts
export const UserSchema = z.object({ id: z.string(), name: z.string() });
export type UserSchema = z.infer<typeof UserSchema>;
export type ListUsersParams = z.input<typeof ListUsersParamsSchema>;
```

A hand-written interface mirroring a schema is a second source of truth and it drifts
silently — the schema keeps validating while the type lies. The one exception is
**component props**: they describe a rendering contract, not a wire shape, and routing
them through a schema would couple presentation to the API.

**Props are written inline, in the component's own file.**

```tsx
export function UserCard({ user, onSelect }: { user: User; onSelect: (id: string) => void }) {
```

Never a `types/` folder, a `user-card.types.ts`, or a `ScreenProps` shared by four
screens. A props type is the component's signature: it's read while reading the markup
it describes, it changes in the same edit, and it dies with the component. Extracting
it puts the contract one file away from every consumer of it, and a shared props type
welds unrelated screens together — the second screen that needs one more prop either
widens the type for everyone or intersects its way around it. Name the type only when
it's genuinely long enough to hurt the signature, and then declare it directly above
the component in the same file.

**Never cast.** No `as`, no `!`, no `as unknown as`, no `any`. A cast is a claim the
compiler can't check, and here it's always a symptom with a real fix:

| Reaching for                          | Do instead                               |
| ------------------------------------- | ---------------------------------------- |
| `as` on an API response               | parse it with the schema                 |
| `!` on a possibly-missing query param | `skipToken`                              |
| `as` to hit a literal type            | annotate the const rather than assert it |
| `any` for an awkward third-party type | narrow it once in `lib/`, typed          |

`as const` and `satisfies` are fine — they narrow and check. They don't assert.

## Pitfalls

- **A component imports a query, mutation, or repository** — it's a container now. It
  stopped being reusable and it refetches on every mount elsewhere. Lift the fetch and
  pass props, even if that means drilling three of them.
- **Promoted to root on first use** — wait for the second consumer. One caller isn't
  evidence of a shared abstraction.
- **A container that only fetches and renders one component** — merge them; the split
  earns its keep only when something composes.
- **A `__tests__` or `tests/` directory** — tests belong beside their subject.
- **`user-card.tsx` instead of `user-card/index.tsx`** — components and containers are
  folders, so the test and anything private have somewhere to live.
- **A relative import** — use the `@/` alias. If lint didn't catch it, the config is
  wrong; see `references/imports.md`.
- **Any `as`, `!`, or `any` appeared** — each has a fix: parse it, `skipToken` it, or
  annotate the const. A cast asserts a type rather than establishing it.
- **A hand-written interface mirroring a schema** — derive it with `z.infer` /
  `z.input`. Two declarations of one shape drift, and the type is the half that lies.
- **Props derived from a schema** — the allowed exception runs the other way. Props are
  a rendering contract; deriving them welds the component to the API.
- **A props type in its own file, or shared across components** — write props inline in
  the signature, in the component's own file. A `types/` folder for props puts the
  contract a file away from the markup, and a shared props type couples every screen
  that uses it.
- **`domain/user.entity.ts` instead of `domain/entities/user.ts`** — the kind is the
  folder. A flat `domain/` interleaves entities, repositories and schemas until you
  can't see one kind at a glance.
