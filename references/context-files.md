# Context files

Part of `renan-frontend-architecture`. Read `SKILL.md` for the folder structure these
files sit in.

Two formats, both created **lazily** — only when there's something real to write. An
empty or stale context file is worse than none, because it gets trusted.

## `AGENTS.md` — how to work here

Constraints, non-obvious decisions, things that look wrong but aren't. Put one at a
feature root when the feature has rules a reader couldn't infer from the code. Good
entries answer "why is this like this?" or "what will I break?":

```md
# containers/vehicle-search

- The search field takes structured input only. No natural-language parsing:
  the typeahead already absorbs "onix 18" and this has to feel instant.
- Result, Disambiguation and Miss are three states of one route, not three
  routes — they are one question being answered.
- `lib/parse-plate.ts` is deliberately permissive; the API rejects bad plates
  and its error message is better than ours.
```

Skip anything the code already says. "This folder contains the search container" is
noise; the folder name said that.

## `CONTEXT.md` — vocabulary only

Terms, one or two sentences each, with an `_Avoid_` list — the format the
`domain-modeling` skill owns. Invoke that skill when adding terms rather than
freestyling, and keep implementation detail out entirely.

```md
**Recommendation**:
The viscosity a Manual specifies for a Version, plus any permitted Alternates.
_Avoid_: suggestion, result

**Miss**:
A search that resolved to a Version with no Recommendation on file.
_Avoid_: not found, empty
```

## Which one, and where

If it changes when the **code** changes, it's `AGENTS.md`. If it changes when the
**business** changes, it's `CONTEXT.md`. Don't create `DOMAIN.md` — a third format
competing with `CONTEXT.md` for the same job, and the folder is already `domain/`.

`AGENTS.md` belongs wherever the local rules are, usually a feature container.
`CONTEXT.md` is the opposite: one at the repo root covers most projects, and a feature
gets its own only when it has genuinely separate vocabulary — at which point
`domain-modeling`'s `CONTEXT-MAP.md` lists them and how they relate.
