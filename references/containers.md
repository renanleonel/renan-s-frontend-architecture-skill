# Containers and Suspense

Part of `renan-frontend-architecture`. Read `SKILL.md` for placement, naming, and
typing rules. The hooks themselves live in `queries.md` and `mutations.md`.

A container fetches, composes, and owns the boundaries. Everything below it takes
props.

## Suspense is the default for reads

**Every read is `useSuspenseQuery`, and every loading region is a skeleton.** A screen
that fetches is a shell plus the content that reads the data: the shell renders what
never loads — heading, search field, "add" button — then an `ErrorBoundary` wrapping a
`<Suspense>` whose fallback is a skeleton shaped like the content it stands in for.

Suspense is still chosen **per query**, never switched on globally. In v5 there's no
`suspense: true` option — it was removed — so the choice *is* the hook you import, and
a reader sees it at the call site. Name the hooks for it: `useUserSuspense`,
`useStockSuspense`. Reach for plain `useQuery` only for a read the screen renders
without — a background refresh, a count in a corner, a poll — and then the component
owns its own fallback.

```ts
export function useUserSuspense({ id }: { id: string }) {
  return useSuspenseQuery<UserSchema, QueryError, User>({
    queryKey: [USER_BY_ID_QUERY_KEY, id],
    queryFn: ({ signal }) => UserRepository.getById({ id }, signal),
    select: schema => new User(schema),
    staleTime: MINUTE * 3,
  })
}
```

`data` is non-nullable, so no `if (!user) return null` and no optional chaining
downstream — that's most of the benefit.

**`useSuspenseQuery` has no `enabled`, `placeholderData`, or `throwOnError`** — they
don't exist on its options type and won't compile. `skipToken` is rejected too: its
`queryFn` type explicitly excludes it. Conditional fetching becomes conditional
*rendering*, so the container decides whether the child exists at all.

## Composing

```tsx
function UserPanelContent({ id }: { id: string }) {
  const { data: user } = useUserSuspense({ id })
  return <UserCard initials={user.initials} />
}

export function UserPanel({ id }: { id?: string }) {
  if (!id) return null                     // this replaces `enabled`
  return (
    <ErrorBoundary fallback={<UserError />}>
      <Suspense fallback={<UserCardSkeleton />}>
        <UserPanelContent id={id} />
      </Suspense>
    </ErrorBoundary>
  )
}
```

The component that suspends must sit **below** the boundary, not render it — a
component can't catch its own suspension, so a `useSuspenseQuery` next to its own
`<Suspense>` bubbles to whatever boundary is further up and blanks more of the screen
than intended. That split is why a container is two components' worth of thinking: the
boundary shell and the content that reads the data.

The split also decides where conditional fetching goes. `useSuspenseQuery` has no
`enabled` and rejects `skipToken`, so a param that isn't ready yet is the shell's
problem: it decides whether the content component exists at all, and the content
component takes the param as a required prop. The empty state lives with the data
too — `if (items.length === 0)` belongs in the content component, which is the only
one that knows.

## Skeletons, not spinners

The fallback is a **skeleton with the shape of the content it replaces** — the same
rows, the same card, the same number of columns. A skeleton says what is coming and
holds the layout still; a spinner says only "wait", and then the page jumps when the
real thing lands at a different size. Give a list skeleton roughly the row count the
list usually has.

Skeletons are components like any other, so they live beside what they mirror in
`components/` (`list-skeleton/`, `recommendation-skeleton/`) and get reused by every
boundary that shows that shape. A one-off shape can be a `<Skeleton />` primitive with
the content's dimensions on it, inline in the fallback.

A spinner is still right in two places, and only these: **inside a control waiting on a
mutation** (a submit button, a row's stepper) and **as an in-place activity hint on
something already rendered** (the icon slot of a search field that's re-querying). Both
sit on top of a UI that already exists. The moment a spinner stands in for content that
hasn't arrived, it should have been a skeleton.

## Placing boundaries

Put the boundary where the fallback should appear, one per independently-loading
region — wrapping a whole page in a single `<Suspense>` makes fast data wait on the
slowest query, and a heading that could have painted immediately waits behind a
network call. Keep the shell above the boundary as large as it honestly can be.

Error boundaries go outside suspense boundaries at the same granularity: a region that
can load on its own should be able to fail on its own. A secondary region takes
`fallback={null}` so its failure costs nothing — the screen degrades to what it was
before that region existed, instead of the whole route going blank.

**An error boundary needs a way back.** It stays in the error state until it remounts,
so key it on whatever the user changes to retry — `key={query}`, `key={id}` — and
typing or navigating recovers on its own. Where the retry is a button rather than a new
param, pair the boundary with `QueryErrorResetBoundary` so the retry refetches instead
of re-throwing the cached error.

## Filtering a list

Three rules, and they hold whether the list is ten rows or ten thousand.

**The server filters.** Filters go to the endpoint as query params it parses; the
client does not fetch everything and narrow it in memory. In-memory filtering is a
decision that expires the day the list stops fitting in one response, and then it has
to be rewritten on both sides. The filters become part of the query key, so each
combination caches on its own:

```ts
export function useStockSuspense(filters: StockFilters) {
  return useSuspenseQuery({
    queryKey: [STOCK_QUERY_KEY, filters],
    queryFn: ({ signal }) => StockRepository.list(filters, signal),
    staleTime: MINUTE,
  })
}
```

A plain object of primitives hashes stably, so a fresh identity each render is fine —
a `Date` or a function in there is not. Invalidating the bare `[STOCK_QUERY_KEY]`
prefix still covers every filtered variant.

**Open sets come from the API; only closed sets are enums.** A value the user types in
(a brand, a grade, a standard) or that grows with the data cannot be enumerated in the
frontend, so the API serves the options — `/api/stock/options` next to `/api/stock`.
Closed sets stay enums in the contract and ship in the same response, so one request
describes the whole filter bar. Options are a read like any other: their own hook, its
own boundary, a filter-shaped skeleton, and invalidated by the mutations that can
change them (adding the first Petronas bottle puts Petronas in the brand list).

**Filter state lives in the URL**, not in `useState`. A filtered view is then
bookmarkable and shareable, the back button behaves, and a reload doesn't silently drop
the filters while leaving the results looking authoritative. Write with `replace: true`
so a filter bar doesn't stack a history entry per keystroke, and pass a text filter
through `useDeferredValue` before it reaches the query key. Narrow a URL string to an
enum member by parsing it — an unknown value means "no filter", never a cast.

**A filter bar collapses.** It opens with the two filters the screen is usually
narrowed by, as unset chips, and a `+` button holding the rest in a flat menu — not a
hover submenu, which hides the options behind a gesture a thumb doesn't have. Picking
an attribute attaches its chip and opens that chip's own menu, which holds the values
and nothing else — a chip is dropped with the unpin icon that appears on it on hover
and stays while its menu is open, kept visible on a coarse pointer where no hover exists
to reveal it. Animate the trigger's padding so the chip grows into that icon rather than
reserving dead space for it at rest.

Make the values multi-select unless one of them genuinely excludes the others: values
within a filter are OR-ed, filters are AND-ed, and they ride the URL as repeated params
(`?brand=a&brand=b`) so `getAll` and `append` round-trip them without a delimiter to
escape. Watch the parse on the way in: a `z.coerce.number()` sitting first in a
`union([item, array(item)])` swallows `[]` as `0`, because `Number([])` is `0` — put the
array branch first. The bar is one row until
someone adds to it, and only the filters in play take space. A grid of labelled selects spends three rows of
a phone screen before anything is filtered. Build the menus from `DropdownMenu`, not
`Select`: a select panel aligns itself over its trigger and shifts the page as it
opens, while a dropdown floats above the layout and never moves it. Where a `Select` is
still right — inside a form — give it `position="popper"` so it drops below the field.

An empty result under active filters is **not** the same empty state as an empty
collection. Say which one it is and offer to clear the filters; a screen that reports
"you have nothing" when the user has merely filtered everything out is lying to them.

## Pitfalls

- **A filter bar built from `Select`s** — it eats the screen before it filters
  anything, and each panel shifts the layout as it opens. Collapse it to chips built
  from `DropdownMenu`.
- **Filtering fetched data in the component** — the endpoint filters. The client
  version breaks the first time the list outgrows one response.
- **A hard-coded list of filter options** — anything the data owns (brands, grades,
  standards) has to come from the API; only a closed set is an enum.
- **"Nothing here" shown for a filtered-empty list** — distinguish the two empty
  states, and give the user a way back to the unfiltered view.
- **A spinner where content is loading** — a skeleton of that content holds the layout
  and says what's coming. Spinners belong in a button waiting on a mutation.
- **A fallback that doesn't match the content** — a 3-row skeleton in front of a card,
  or a bare `<Spinner />` in front of a list. The layout jumps on resolve, which is the
  thing the skeleton existed to prevent.
- **A read left on `useQuery` with an `isPending` branch** — that's the old shape. Reads
  are `useSuspenseQuery`; the branch becomes a boundary and a skeleton.
- **An error boundary with no key and no reset** — it's stuck on the failed render
  forever, so the screen stays broken after the user has already fixed the input.
- **Blank screen instead of a skeleton** — the suspending hook is in the same component
  as its `<Suspense>`, so it bubbled to an ancestor boundary.
- **Reached for `enabled` on a suspense query** — doesn't exist. Gate the render, or use
  `useQuery` with `skipToken`.
- **Everything waits on one slow query** — a single page-level boundary. Split per
  region.
- **Waterfall** — sibling suspense queries render sequentially when each sits behind its
  own nested boundary. Put independent reads in one component under one boundary so
  they start together.
