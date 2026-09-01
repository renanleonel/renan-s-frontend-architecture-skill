# Domain

Part of `renan-frontend-architecture`. Read `SKILL.md` for placement, naming, and
typing rules. The hooks that consume this live in `queries.md` and `mutations.md`.

`domain/` holds the feature's model and how it reaches the wire: enums, zod schemas,
entities, repositories. It never imports React.

Each kind gets its own folder and the folder carries the kind, so the filename doesn't
repeat it:

```
domain/
├── entities/
│   └── user.ts          # export class UserEntity
├── repositories/
│   └── user.ts          # export const UserRepository
├── schemas/
│   └── user.ts          # UserSchema, ListUsersParamsSchema
├── enums/
│   └── fuel.ts
└── labels.ts            # rules and label maps that aren't one of those kinds
```

Loose files stay at the root of `domain/` — don't create a folder to hold one file.

## Enums

Any closed set of values is an enum. A value belonging to a fixed set never appears as
a bare string in application code — bare strings don't rename, don't autocomplete, and
don't fail the build when the set changes.

```ts
export enum Fuel {
  FLEX = "flex",
  GASOLINE = "gasoline",
  ETHANOL = "ethanol",
}
```

Members are UPPERCASE; **wire values stay exactly as the API sends them**. The value
is a contract with the backend, not a style choice — changing it is a backend change.

Schemas derive from the enum rather than restating it, so the two can't drift:

```ts
export const UserSchema = z.object({ fuel: z.enum(Fuel) })
```

For a discriminated union keep `z.literal(...)`, but pass the member:
`z.literal(Status.RESOLVED)`. Compare against members (`fuel === Fuel.FLEX`), never
literals.

**Not just wire values.** Any fixed set is an enum, and the ones people miss aren't
the API fields — they're the UI ones: component variant and size props, route paths,
storage keys, query-param names, discriminated-union tags.

```ts
// wrong
type Variant = "primary" | "ghost"
navigate("/historico")

// right
export enum Variant { PRIMARY = "primary", GHOST = "ghost" }
navigate(AppRoute.HISTORY)
```

An inline string union looks harmless because it's typed, but it doesn't rename, it
can't be iterated to build a picker, and every consumer restates the literals. These
live in `domain/enums/` alongside the wire enums — a fixed value set is domain
vocabulary whether or not it crosses the network.

## Schemas and entities

The schema is the wire shape. The entity is the behavior.

```ts
export const UserSchema = z.object({
  id: z.string(),
  name: z.string(),
  fuel: z.enum(Fuel),
})
export type UserSchema = z.infer<typeof UserSchema>

export class User {
  constructor(private readonly schema: UserSchema) {}
  get initials() { return this.schema.name.slice(0, 2).toUpperCase() }
  runsOnEthanol() { return this.schema.fuel === Fuel.FLEX }
}
```

Entities are built in `select`, never stored. The cache holds plain parsed objects,
which keeps devtools, SSR hydration, and optimistic `setQueryData` merges working — a
structural merge over a class instance keeps the fields and silently drops the
methods. So: the repository parses to a schema, `select` wraps it in an entity,
optimistic updates write schemas.

Logic belonging to one record goes on the entity. Logic across records goes in the
feature's `lib/`.

## Repositories

One function per endpoint. Parses params in, parses the response out, knows nothing
about React.

```ts
export const UserRepository = {
  async getById(params: GetUserParams, signal: AbortSignal): Promise<UserSchema> {
    const { id } = GetUserParamsSchema.parse(params)
    const { data } = await http.get(`/users/${id}`, { signal })
    return UserSchema.parse(data)
  },

  async list(params: ListUsersParams, signal: AbortSignal): Promise<ListUsersResponse> {
    const query = ListUsersParamsSchema.parse(params)
    const { data } = await http.get("/users", { params: query, signal })
    return ListUsersResponseSchema.parse(data)
  },

  async update(params: UpdateUserParams): Promise<UserSchema> {
    const { id, ...body } = UpdateUserParamsSchema.parse(params)
    const { data } = await http.patch(`/users/${id}`, body)
    return UserSchema.parse(data)
  },
}
```

Destructuring the parsed value is what splits it into path, query, and body — `id`
goes in the URL, the rest becomes the payload.

- **Parse params first, then use only the parsed value.** The classic bug is parsing
  into a variable and sending the original object anyway, so defaults and coercions
  never apply. Destructure immediately and the mistake can't compile.
- **Parse the whole response envelope** (`{ items, total }`), not just the payload
  inside it — `z.array(UserSchema).parse(data.items)` lets a changed wrapper through.
- **Type public params with `z.input`, not `z.infer`.** With `.default()` the input and
  output types differ, and `z.infer` gives the output, forcing callers to pass values
  the schema exists to fill in.
- **Reads take a required `signal`.** `queryFn` always has one, so requiring it makes
  cancellation a compile error rather than a review note. Mutations take none;
  TanStack doesn't give `mutationFn` a signal, and a half-applied write is worse than
  a slow one.
- **Return the schema, never an entity.** `select` builds the entity; an entity in the
  cache loses its methods on the next merge.
- **Never `safeParse` with a fallback.** It turns a broken contract into a successful
  query holding junk. Let `ZodError` propagate.

`z.object` strips unknown keys by default. That's deliberate: what the frontend
declared is what the frontend gets, so a new backend field can't quietly change
behavior downstream.

## Pitfalls

- **Bare string where an enum exists** — no rename safety, no exhaustiveness check.
- **`signal?: AbortSignal` under `exactOptionalPropertyTypes`** — axios declares
  `signal?: GenericAbortSignal` without `| undefined`, so `{ signal }` fails to
  compile. Make it required on reads, or spread it: `...(signal && { signal })`.
- **Response schema reused as the param schema** — they drift the moment the API
  accepts a field it doesn't return. Keep them separate even when identical today.
- **Enum member renamed and the wire value with it** — the value is the contract.
  Rename the member freely; changing the string is a backend change.
- **A flat `domain/` with `user.entity.ts`, `user.repository.ts`** — the kind is the
  folder: `domain/entities/user.ts`, `domain/repositories/user.ts`.
