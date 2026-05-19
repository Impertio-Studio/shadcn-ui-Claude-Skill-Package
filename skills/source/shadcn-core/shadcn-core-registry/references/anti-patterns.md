# Reference : Registry + components.json Anti-Patterns

Six recurring registry and configuration mistakes. For each : the broken
shape, WHY it fails, and the verified fix. All anti-patterns derived from
the official docs and the CLI's actual resolution behaviour, verified
2026-05-19 against `shadcn@4.7.0`.

## 1. Changing `style` After Init

### Broken

`components.json` was initialised with `style: "new-york"`. Two months
later, a designer wants to try the new `sera` style. Someone edits
`components.json` :

```diff
- "style": "new-york",
+ "style": "sera",
```

Then runs `pnpm dlx shadcn@latest add badge` and expects the project
to look sera-styled. The badge file IS fetched with sera content, but
every previously-installed component (Button, Card, Dialog, etc.) is
still on new-york.

### Why it fails

The CLI fetches per-style content from the registry. The `{style}`
URL placeholder resolves at fetch time and ONLY affects the request that
is happening NOW. Components already on disk are not re-rendered.

`style` is one of the three IMMUTABLE fields in components.json
(`style`, `tailwind.baseColor`, `tailwind.cssVariables`), per
https://ui.shadcn.com/docs/components-json. The CLI does not enforce
immutability ; it relies on the publisher's discipline.

Visual drift compounds : the new badge subtly clashes with old buttons,
and as more components are added or re-added, the codebase ends up with
a mixed-style heritage that no diff can fully reconcile.

### Fix

If a style change is genuinely required :

1. Commit the current state.
2. Edit `style` in components.json.
3. Run `pnpm dlx shadcn@latest add --all --overwrite` to re-fetch every
   component under the new style.
4. Manually re-apply any local customisations (use the git diff from
   step 1).
5. Visually QA every screen.

ALWAYS commit to the `style` choice BEFORE running `init`. NEVER toggle
`style` casually ; the cost is a full sweep plus manual reconciliation.

## 2. Missing `aliases.utils` When Code Imports `@/lib/utils`

### Broken

A developer hand-edits `components.json` to remove what looks like a
redundant field :

```diff
  "aliases": {
    "components": "@/components",
-   "utils": "@/lib/utils",
    "ui": "@/components/ui",
    "lib": "@/lib",
    "hooks": "@/hooks"
  }
```

The reasoning : "We have `lib`, why repeat `lib/utils`?" The next
`shadcn add button` succeeds, but TypeScript complains about a missing
`cn` import in the generated `button.tsx`.

### Why it fails

`aliases.utils` and `aliases.lib` serve DIFFERENT purposes. `aliases.lib`
is the DESTINATION for `registry:lib` items. `aliases.utils` is the
IMPORT PATH the CLI writes into every generated component file when it
imports the `cn` helper.

When `aliases.utils` is missing, the CLI falls back to a default that
may not match the project's actual `cn` location. The generated file
imports from a non-existent module ; TypeScript flags it, the dev server
fails the build.

### Fix

Re-add `aliases.utils` with the exact path your `cn` helper lives at :

```json
{
  "aliases": {
    "components": "@/components",
    "utils": "@/lib/utils",
    "ui": "@/components/ui",
    "lib": "@/lib",
    "hooks": "@/hooks"
  }
}
```

If the project uses `package.json#imports`, the equivalent is :

```json
{
  "aliases": {
    "utils": "#lib/utils"
  }
}
```

ALWAYS keep all five alias fields populated. NEVER assume one is
derivable from another ; they map to distinct CLI behaviours.

## 3. Alias Mismatch : Config Says `src/components` But Code Imports `@/components/ui`

### Broken

```json
// components.json
{
  "aliases": {
    "components": "src/components",
    "ui": "src/components/ui"
  }
}
```

```tsx
// app/page.tsx
import { Button } from "@/components/ui/button"
```

The dev server reports : "Cannot resolve module '@/components/ui/button'".
The developer goes hunting in node_modules.

### Why it fails

The `aliases` field in components.json must match the IMPORT-PATH form
that the consuming code uses. shadcn writes generated files relative to
the alias VALUE, and rewrites internal imports against the same value.

In the broken example, `aliases.components = "src/components"` is a
file-system-style path, not an import path. The CLI writes the file
to `src/components/ui/button.tsx`, then writes the import statement as
`import ... from "src/components/ui/button"`, which is not resolvable
by any bundler.

The format MUST be an import-resolvable alias. With tsconfig paths, it
is `@/components/ui` (where `@/*` maps to `./src/*`). With
`package.json#imports`, it is `#components/ui` (where
`#components/*` maps to `./src/components/*`).

### Fix

Use the alias form, never the raw path :

```json
// components.json
{
  "aliases": {
    "components": "@/components",
    "ui": "@/components/ui"
  }
}
```

And ensure the underlying alias mapping exists in `tsconfig.json` :

```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": { "@/*": ["./src/*"] }
  }
}
```

ALWAYS use the import-resolvable form (`@/components/ui` or
`#components/ui`). NEVER use raw file-system paths
(`src/components/ui`, `./src/components/ui`) in components.json.

## 4. Custom Registry Without `{name}` Placeholder

### Broken

```json
{
  "registries": {
    "@acme": "https://registry.acme.com/components.json"
  }
}
```

Running `shadcn add @acme/button` fetches
`https://registry.acme.com/components.json` and gets the index file back
instead of the button item. The CLI sees an unexpected shape and errors :
"Registry item validation failed".

### Why it fails

The `{name}` placeholder is REQUIRED in every URL template (verified at
https://ui.shadcn.com/docs/registry/namespace). The CLI does NOT
automatically append the item name to the URL. The placeholder is the
substitution point that maps an `add` argument to a specific item URL.

Without `{name}`, every namespace install hits the same URL regardless
of which item was requested. The registry has no way to disambiguate.

### Fix

Add the `{name}` placeholder at the correct position in the URL :

```json
{
  "registries": {
    "@acme": "https://registry.acme.com/{name}.json"
  }
}
```

If the registry serves per-style variants too :

```json
{
  "registries": {
    "@acme": "https://registry.acme.com/{style}/{name}.json"
  }
}
```

ALWAYS include `{name}` in every URL template. NEVER hand-craft URLs
that do not have a substitution point.

## 5. Committing Headers With Literal Token Instead of Env-Var

### Broken

```json
{
  "registries": {
    "@private": {
      "url": "https://api.company.com/registry/{name}.json",
      "headers": {
        "Authorization": "Bearer EXAMPLE_TOKEN_DO_NOT_COMMIT_REAL_SECRETS"
      }
    }
  }
}
```

The developer commits the change. CI passes. Two weeks later the token
shows up in a security scan, gets rotated, and the registry stops
working in every environment.

### Why it fails

`components.json` is in source control. Committing literal secrets
exposes them in the git history forever, even after rotation. The token
also leaks via :

- Branch pushes (visible to every collaborator with read access)
- Mirrored caches (GitHub forks, gh search results, CI cache layers)
- Anyone who clones a snapshot before the rotation

The env-var expansion mechanism exists exactly for this : the CLI
substitutes `${VAR_NAME}` against `process.env` at invocation time, so
the secret never enters source control.

### Fix

Use `${ENV_VAR}` expansion :

```json
{
  "registries": {
    "@private": {
      "url": "https://api.company.com/registry/{name}.json",
      "headers": {
        "Authorization": "Bearer ${REGISTRY_TOKEN}"
      }
    }
  }
}
```

Store the real token in :

- `.env.local` for local development (gitignored)
- CI/CD secret store for build pipelines
- OS keychain or `direnv` for shell sessions

If a secret has already been committed :

1. Rotate the secret immediately at the registry vendor.
2. Replace the literal with `${VAR_NAME}` in components.json.
3. Commit the fix.
4. Optionally rewrite git history with `git filter-repo` for older
   exposures (best-effort ; assume the old token is fully compromised).

ALWAYS use `${VAR_NAME}` for any secret in components.json. NEVER commit
literal tokens, even temporarily.

## 6. Mixing `tailwind.config` Field (v3 Shape) With v4 Setup

### Broken

A project migrated from Tailwind v3 to v4. The v4 install dropped the
JS/TS config in favour of CSS-first `@theme inline { ... }`. But
`components.json` still reads :

```json
{
  "tailwind": {
    "config": "tailwind.config.ts",
    "css": "app/globals.css",
    "baseColor": "neutral",
    "cssVariables": true
  }
}
```

Running `shadcn add button` either silently writes a button with v3-shape
classes (`bg-[hsl(var(--background))]`) or fails with a Tailwind config
parse error, depending on whether `tailwind.config.ts` is still present.

### Why it fails

The `tailwind.config` field signals to the CLI which Tailwind generation
the project uses. Pre-v4 projects MUST point at a real config file. v4
projects MUST omit the field (or set it to `""`) because v4 has no JS/TS
config to load ; theme is declared inside CSS via `@theme inline`.

If the field points at a non-existent or stale config, the CLI cannot
resolve the project's theme tokens. Generated components may reference
`hsl(var(--background))` (v3 form) while the project's CSS only exposes
`oklch(...)` values inside `@theme inline` (v4 form), producing silent
style failures.

### Fix

For a v4 project, edit components.json to clear the field :

```json
{
  "tailwind": {
    "config": "",
    "css": "app/globals.css",
    "baseColor": "neutral",
    "cssVariables": true
  }
}
```

Then run `pnpm dlx shadcn@latest add button --overwrite` to re-fetch
the button with v4-shape classes. Sweep every previously-installed
component the same way OR accept the visual drift.

Cross-reference :
[shadcn-errors-tailwind-v3-v4-migration](../../shadcn-errors/shadcn-errors-tailwind-v3-v4-migration/SKILL.md)
covers the full migration including `@theme inline` and the
HSL-to-oklch token format change.

ALWAYS keep `tailwind.config` in sync with the actual Tailwind
generation. NEVER leave a stale `config` value pointing at a deleted
or unused v3 config.
