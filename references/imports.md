# Imports and path aliases

Part of `renan-frontend-architecture`. The rule — always absolute through `@/` — is in
`SKILL.md` and applies to every file. This file is the wiring behind it, needed once
per repo and again whenever the alias misbehaves.

## Alias setup

Two files that have to agree. TypeScript resolves types from `paths`; the bundler
resolves the actual module from `alias`. Set only one and you get a build that
typechecks then fails at runtime, or the reverse — so when the alias works in the
editor but not at runtime (or vice versa), these two have drifted.

```jsonc
// tsconfig.json
{ "compilerOptions": { "paths": { "@/*": ["./src/*"] } } }
```

```ts
// vite.config.ts
resolve: { alias: { "@": path.resolve(import.meta.dirname, "./src") } }
```

## Lint enforcement

An unchecked convention decays in a week, so the linter owns this rule:

```jsonc
// .oxlintrc.json
{
  "rules": {
    "no-restricted-imports": [
      "error",
      {
        "patterns": [
          {
            "group": ["./**", "../**"],
            "message": "Use the @/ alias instead of a relative path."
          }
        ]
      }
    ]
  }
}
```

Use those two globs exactly. oxlint does not treat `../*` as matching `../lib/thing`,
so the intuitive `["./*", "../*"]` catches sibling imports only and lets every parent
traversal through — the case that actually matters, and it looks like it's working.
Verified on oxlint 1.80.

ESLint uses the same rule name with a different glob engine, so before trusting a
pattern there, test it against a fixture containing both `./x` and `../x`.

## Pitfalls

- **Lint passes but relative imports are everywhere** — the pattern is `../*` instead
  of `../**`, so it's matching nothing.
- **Alias works in the editor but not at runtime**, or the reverse — `tsconfig.json`
  `paths` and the bundler `alias` disagree. They're two separate resolvers.
- **A colocated test importing `./index`** — tests use the alias too:
  `user-card.test.tsx` imports `@/components/user-card`.
