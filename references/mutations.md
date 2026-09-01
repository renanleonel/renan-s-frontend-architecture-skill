# Mutations

Part of `renan-frontend-architecture`. Read `SKILL.md` for placement, naming, and
typing rules. Reads live in `queries.md`; the request itself lives in `domain.md`.

## One file per domain

`mutations/index.ts` holds every mutation for its domain and exports each by name —
no file per hook, same as `queries/index.ts`. Import the keys you invalidate from the
queries file:

```ts
// containers/user-profile/mutations/index.ts
import { USER_BY_ID_QUERY_KEY, LIST_USERS_QUERY_KEY } from "@/containers/user-profile/queries"

export function useUpdateUser() { ... }
export function useCreateUser() { ... }
```

The dependency runs mutations → queries, never back. A query that needs to reach into a
mutation is a design problem, not an import problem — it usually wants the mutation's
result passed down as a prop instead.

## Rules

- **Pass all four generics, or none.** Explicit generics disable inference for the
  rest, so naming three leaves the `onMutate` result typed `{}` and every
  `ctx?.previous` fails with `Property 'previous' does not exist`.
- **Write schemas to the cache, never entities.** A structural merge over a class
  instance keeps the fields and silently drops the methods.
- **Every optimistic write needs a rollback path.** If you can't snapshot it, don't
  write it optimistically.

## Optimistic pattern

```ts
type UpdateVars = { id: string; payload: Partial<UserSchema> }
type UpdateCtx = { previous: UserSchema | undefined }

export function useUpdateUser() {
  const queryClient = useQueryClient()

  return useMutation<UserSchema, QueryError, UpdateVars, UpdateCtx>({
    mutationFn: params => UserRepository.update(params),
    onMutate: async ({ id, payload }) => {
      const key = [USER_BY_ID_QUERY_KEY, id]
      await queryClient.cancelQueries({ queryKey: key })
      const previous = queryClient.getQueryData<UserSchema>(key)
      if (previous)
        queryClient.setQueryData<UserSchema>(key, { ...previous, ...payload })
      return { previous }
    },
    onError: (_err, { id }, ctx) => {
      if (ctx?.previous)
        queryClient.setQueryData([USER_BY_ID_QUERY_KEY, id], ctx.previous)
    },
    onSettled: (_data, _err, { id }) =>
      queryClient.invalidateQueries({ queryKey: [USER_BY_ID_QUERY_KEY, id] }),
  })
}
```

Three things carry the weight. **`await cancelQueries` first** — an in-flight fetch
landing after your optimistic write silently undoes it. **Return the snapshot** from
`onMutate` so `onError` has something to roll back to. **Prefer `onSettled: invalidate`
over `onSuccess: setQueryData`** — invalidating refetches the server's truth and
self-heals when the response differs from what you guessed; writing the response in is
only better when the mutation returns the full entity and you want to skip the round
trip. Doing both is redundant.

`mutationFn` and the callbacks also receive `{ client, meta, mutationKey }`, so
`(vars, { client }) => …` avoids needing `useQueryClient()`.

## Not optimistic

Most mutations shouldn't be. Optimism buys perceived speed and costs a rollback path
plus a divergent-state bug class — worth it for high-frequency, low-stakes writes that
almost always succeed (a toggle, a reorder, a like). For anything a user would be
upset to see revert, just invalidate:

```ts
export function useCreateUser() {
  const queryClient = useQueryClient()
  return useMutation<UserSchema, QueryError, CreateUserParams>({
    mutationFn: params => UserRepository.create(params),
    onSuccess: () =>
      queryClient.invalidateQueries({ queryKey: [LIST_USERS_QUERY_KEY] }),
  })
}
```

## Pitfalls

- **Optimistic update reverts a moment later** — missing `await cancelQueries` in
  `onMutate`.
- **Rollback that refetches instead of restoring** — `onError` should put the snapshot
  back synchronously; a refetch leaves the wrong value on screen meanwhile.
- **Invalidating everything** — `invalidateQueries({})` with no key refetches the whole
  cache. Name the key you actually invalidated.
