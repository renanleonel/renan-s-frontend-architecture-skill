# Queries

Part of `renan-frontend-architecture`. Read `SKILL.md` for placement, naming, and
typing rules. Suspense queries live in `containers.md`; writes live in `mutations.md`.

A query hook owns the key, the caching, and the shape the component sees. It does not
own the request — that's the repository (`domain.md`).

**Reads default to `useSuspenseQuery`** — name the hook `useThingSuspense`, and let the
container hold the boundary and the skeleton (`containers.md`). The `useQuery` patterns
below are for the exception: a read the screen renders without.

## One file per domain

`queries/index.ts` holds every query for its domain and exports each by name. There is
no file per hook.

```ts
// containers/user-profile/queries/index.ts
export const USER_BY_ID_QUERY_KEY = "user-by-id" as const
export const LIST_USERS_QUERY_KEY = "list-users" as const

export function useUser(...) { ... }
export function useUsers(...) { ... }
```

Consumers import only what they need — `import { useUser } from "@/containers/user-profile/queries"` —
and bundlers drop the untouched exports, so the aggregate costs nothing at runtime.

Keys go at the top, exported. Every key partitioning this domain's cache is then
visible in one screen, which is what makes a collision obvious rather than invisible
across a dozen modules. `mutations/index.ts` imports the keys it invalidates from here;
that dependency runs mutations → queries and never back, which is what keeps the two
files from forming a cycle.

## Rules

- **`Omit`, not `Exclude`,** to strip `queryKey`/`queryFn` from an options param.
  `Exclude` is a no-op on object types, so `queryKey` stays required on callers and
  `useUser({ id }, { enabled: false })` fails to compile.
- **Set `staleTime` explicitly.** The default `0` refetches on every mount — not a
  neutral default, a choice to refetch constantly. Pick a value from the data's
  volatility.
- **Keys live in one place, as literal feature-scoped strings.** Two hooks sharing
  `['users']` invalidate each other. Never derive the key from the hook name —
  renaming then silently repartitions the cache.
- **Transform in `select`, never in the component.** `select` results are cached; a
  transform in the component re-derives every render, and skipping `select` leaves the
  component holding a raw schema with none of the entity's methods.

## Loading flags

Only for `useQuery` and mutations. A suspense read has no loading branch — `data` is
there or the boundary is showing the skeleton — but it still exposes `isFetching` for a
background refetch.

| Flag | True when | Use for |
|---|---|---|
| `isPending` | No data cached yet | "nothing to show" branch |
| `isLoading` | `isPending && isFetching` | initial skeleton |
| `isFetching` | Any fetch, including background | refresh indicator |

`isInitialLoading` is deprecated — use `isLoading`.

`useIsFetching({ queryKey })` reads that state from *outside* the boundary, which is how
a shell keeps an activity hint (a spinner in the search field) for a query that now
lives in a child.

## Pattern

```ts
export const USER_BY_ID_QUERY_KEY = "user-by-id" as const

export function useUser(
  { id }: { id?: string } = {},
  options?: Omit<UseQueryOptions<UserSchema, QueryError, User>, "queryKey" | "queryFn">
) {
  return useQuery<UserSchema, QueryError, User>({
    queryKey: [USER_BY_ID_QUERY_KEY, id],
    queryFn: id ? ({ signal }) => UserRepository.getById({ id }, signal) : skipToken,
    select: schema => new User(schema),
    staleTime: MINUTE * 3,
    ...options
  })
}
```

Generics are `<TQueryFnData, TError, TData>` — what the repository returns, what it
throws, what the component gets after `select`. `select` is where the schema becomes
an entity: the cache keeps the plain object, the component gets the class with methods
on it (`domain.md`).

`skipToken` is how a query waits for a param without a cast. `enabled: !!id` also
stops the fetch, but it doesn't narrow types, so `queryFn` would still need `id!` to
compile. Returning `skipToken` puts the check where TypeScript can see it — inside the
branch `id` is a `string` — and the query reports `isPending` until the param arrives.
Keep `enabled` for conditions unrelated to the params.

Drop the `options` param unless callers actually override things.

## Infinite

```ts
const INITIAL_CURSOR: string | null = null

export function useUsers() {
  return useInfiniteQuery({
    queryKey: [USERS_PAGINATED_QUERY_KEY],
    queryFn: ({ pageParam, signal }) => UserRepository.list({ cursor: pageParam }, signal),
    initialPageParam: INITIAL_CURSOR,
    getNextPageParam: last => last.nextCursor,
    staleTime: MINUTE,
  })
}
```

`initialPageParam` is required and types `pageParam` for the whole hook. Inline `null`
would infer as `null` and reject every later cursor, which is what tempts people into
`null as string | null` — annotate the const instead and no cast is needed.
`getNextPageParam` returning `undefined`/`null` ends pagination; a stale cursor loops
forever.

## Pitfalls

- **`select` runs every render** — the memo is keyed on the `select` function's
  identity, so an inline arrow misses it. Fine when cheap; hoist when not.
- **Infinite refetch loop** — a non-primitive in the key (`{ filters }`, `new Date()`)
  getting a fresh identity each render. Serialize or hoist it.
- **Query-level `onSuccess`/`onError`** — removed in v5; they fired once per observer,
  so a query used by three components ran the callback three times. Use `select` to
  derive, an effect to sync, or the global `QueryCache` callback to log.
