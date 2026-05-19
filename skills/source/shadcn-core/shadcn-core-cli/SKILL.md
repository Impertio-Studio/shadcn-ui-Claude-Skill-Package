---
name: shadcn-core-cli
description: >
  Use when running any shadcn CLI command (init, add, view, search, apply,
  preset, build, docs, info, migrate), when writing or editing components.json,
  when choosing between tsconfig path aliases and package.json#imports aliases
  (new in shadcn@4.7.0), when wiring a custom or private registry, or when a
  shadcn invocation overwrote local component edits unexpectedly. Prevents the
  silent-overwrite trap from running `add` without `--diff` after local
  modifications, the "Cannot find baseColor" failure from a malformed
  components.json, the registry-resolution miss from forgetting the `{name}`
  placeholder, and the immutable-field surprise from trying to change `style`,
  `baseColor`, or `cssVariables` after init. Covers every CLI command and flag,
  the complete components.json schema, registry resolution semantics, and the
  package.json#imports alternative to tsconfig paths.
  Keywords: shadcn cli, npx shadcn add, pnpm dlx shadcn, shadcn init, shadcn
  diff, shadcn update, shadcn migrate, shadcn migrate icons, shadcn migrate
  radix, shadcn migrate rtl, components.json, components json schema, aliases,
  package.json imports, custom registry, registries field, shadcn search,
  shadcn view, shadcn preset, shadcn apply, shadcn build, shadcn docs, shadcn
  info, CLI overwrote my changes, how do I update shadcn components, how do I
  add a private registry, what is components.json, getting started with shadcn,
  shadcn config file, monorepo shadcn, rtl shadcn, new-york style, baseColor,
  cssVariables, lucide icons swap, radix unified package, --diff flag,
  --overwrite flag, --dry-run, --all components.
license: MIT
compatibility: "Designed for Claude Code. Requires shadcn ui evergreen-2026 (CLI shadcn@4.7.0)."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# shadcn CLI + components.json

The shadcn CLI distributes component source code from a registry into the
consumer project. Files are COPIED, not installed, so the consumer owns and
modifies them. The CLI is the only sanctioned way to add a component ;
hand-copy from the docs site is supported but loses dependency resolution
and registry namespace addressing.

Companion skills :

- `shadcn-core-architecture` : code-distribution paradigm and ownership model
- `shadcn-core-registry` : registry item schema and authoring custom registries
- `shadcn-errors-cli-sync-mismatch` : recovering after the CLI overwrote local edits

## Quick Reference : Command at a Glance

| Command | Purpose | Most common invocation |
|---------|---------|------------------------|
| `init` | Bootstrap components.json + deps | `pnpm dlx shadcn@latest init` |
| `add` | Copy components into project | `pnpm dlx shadcn@latest add button` |
| `view` | Preview registry item before install | `pnpm dlx shadcn@latest view dialog` |
| `search` | Browse a registry | `pnpm dlx shadcn@latest search @shadcn -q "form"` |
| `apply` | Apply a preset (theme / font) | `pnpm dlx shadcn@latest apply a2r6bw --only theme` |
| `preset decode` | Inspect a preset code | `pnpm dlx shadcn@latest preset decode a2r6bw` |
| `build` | Generate registry JSON (publishers) | `pnpm dlx shadcn@latest build --output ./public/r` |
| `docs` | Fetch component docs | `pnpm dlx shadcn@latest docs button` |
| `info` | Print resolved project config | `pnpm dlx shadcn@latest info` |
| `migrate icons` | Swap icon library | `pnpm dlx shadcn@latest migrate icons` |
| `migrate radix` | Move to unified `radix-ui` | `pnpm dlx shadcn@latest migrate radix` |
| `migrate rtl` | Add RTL support | `pnpm dlx shadcn@latest migrate rtl` |

NEVER use `npm install shadcn-ui` : that ships a stale legacy package.
ALWAYS use `pnpm dlx shadcn@latest <cmd>` (or `npx shadcn@latest <cmd>`).
The `@latest` tag matters : without it, npm caches the older binary.

## Quick Reference : There Is NO `diff` Subcommand

The raw masterplan and many community blog posts mention a `shadcn diff`
SUBCOMMAND. It does NOT exist. The diff workflow is :

- Preview changes : `pnpm dlx shadcn@latest add button --diff`
- Apply changes : `pnpm dlx shadcn@latest add button --overwrite`

There is also NO `shadcn update` subcommand. The "update a component"
workflow IS the diff + overwrite pair above.

## Quick Reference : Add Flow Decision Tree

```
Do you have a components.json yet?
├── NO  → run `pnpm dlx shadcn@latest init` FIRST
└── YES → Does the target component already exist in components/ui/?
    ├── NO  → `pnpm dlx shadcn@latest add <name>` (writes the file)
    └── YES → Have you modified the local copy?
        ├── NO  → `pnpm dlx shadcn@latest add <name> --overwrite`
        │        (re-copies cleanly from registry)
        └── YES → `pnpm dlx shadcn@latest add <name> --diff`
                  (show what would change, then merge by hand OR
                  accept rewrite with --overwrite + manual reapply)
```

ALWAYS run `--diff` before `--overwrite` on any component you have edited.
NEVER assume `add <name>` is safe to re-run : when the file exists, the CLI
SKIPS by default ; when you force it with `--overwrite`, your edits are gone.

## init : Bootstrap a Project

```bash
pnpm dlx shadcn@latest init
```

Interactive prompts ask for : framework template, base library (radix vs base),
preset, name (if scaffolding new project), baseColor, cssVariables. Outputs
`components.json` at the project root and installs runtime dependencies
(`tailwind-merge`, `clsx`, `class-variance-authority`, `lucide-react`).

Common flags :

| Flag | Purpose |
|------|---------|
| `-d, --defaults` | next template + nova preset, no prompts |
| `-t next \| vite \| start \| react-router \| laravel \| astro` | Framework template |
| `-b radix \| base` | Base component library |
| `-p <preset>` | Apply a preset during init |
| `-f, --force` | Overwrite existing components.json |
| `--monorepo` | Scaffold for a monorepo layout |
| `--rtl` | Enable RTL CSS during init |
| `--no-css-variables` | Inline color utilities instead of semantic tokens |
| `--pointer` | Add `cursor-pointer` styling to buttons (April 2026) |
| `--reinstall` | Re-install UI components from scratch |

NEVER pass `--no-css-variables` lightly : `cssVariables` is IMMUTABLE after
init. Changing it later requires a full re-init of every component file.

## add : Copy Components Into the Project

```bash
pnpm dlx shadcn@latest add button card dialog
pnpm dlx shadcn@latest add @shadcn/sidebar-07          # block
pnpm dlx shadcn@latest add https://example.com/r/x.json # remote URL
pnpm dlx shadcn@latest add @myorg/datepicker            # custom registry
pnpm dlx shadcn@latest add ./local/registry-item.json   # local path
```

Important flags :

| Flag | Purpose |
|------|---------|
| `-y, --yes` | Skip confirmation |
| `-o, --overwrite` | Overwrite existing files (DESTROYS local edits) |
| `-a, --all` | Add every component in the registry |
| `-p, --path <path>` | Override `aliases.components` destination |
| `--dry-run` | Print what would happen, write nothing |
| `--diff [path]` | Show file diffs against current local copy |
| `--view [path]` | Print file contents that would be written |

ALWAYS chain `add` to a `cn` import check : after every `add`, verify the
new file imports from `aliases.utils` (default `@/lib/utils`). If the alias
resolution failed, the new file ships with broken imports.

## view, search, docs, info

```bash
pnpm dlx shadcn@latest view button card                  # preview file content
pnpm dlx shadcn@latest search @shadcn -q "table"         # registry search
pnpm dlx shadcn@latest search @shadcn -q "button" -l 10  # limit results
pnpm dlx shadcn@latest docs dialog                       # fetch doc snippets
pnpm dlx shadcn@latest docs dialog --json                # JSON for tooling
pnpm dlx shadcn@latest info                              # resolved config dump
```

`search` accepts ONE or MORE registry namespaces : `search @shadcn @myorg`.
`view` and `docs` are read-only and write nothing to disk.

## apply + preset : Themes and Fonts

Presets are short codes that encode a theme + font combination. Use them to
apply a curated palette without hand-editing CSS variables.

```bash
pnpm dlx shadcn@latest apply a2r6bw                # full preset
pnpm dlx shadcn@latest apply a2r6bw --only theme   # theme only
pnpm dlx shadcn@latest apply a2r6bw --only font    # font only
pnpm dlx shadcn@latest preset decode a2r6bw        # show what the code expands to
pnpm dlx shadcn@latest preset decode a2r6bw --json # JSON form
pnpm dlx shadcn@latest preset url a2r6bw           # share-link URL
pnpm dlx shadcn@latest preset open a2r6bw          # open in browser
pnpm dlx shadcn@latest preset resolve              # show currently active preset
```

## build : Author a Registry (Publishers Only)

```bash
pnpm dlx shadcn@latest build                          # default ./registry.json -> ./public/r
pnpm dlx shadcn@latest build ./custom.json -o ./out  # custom paths
```

Consumes a `registry.json` that describes items, emits one JSON per item
under the output dir. Consumers fetch those JSONs via the `registries`
field of their components.json.

## migrate : Three Available Migrations

```bash
pnpm dlx shadcn@latest migrate -l                     # list migrations
pnpm dlx shadcn@latest migrate icons                  # swap icon library
pnpm dlx shadcn@latest migrate radix                  # consolidate to `radix-ui`
pnpm dlx shadcn@latest migrate rtl                    # add RTL support
pnpm dlx shadcn@latest migrate radix src/components/ui/dialog.tsx  # single file
pnpm dlx shadcn@latest migrate rtl "src/components/ui/**"         # glob
```

`migrate icons` prompts for the target library (lucide, radix, tabler, etc.)
and updates `iconLibrary` in components.json plus rewrites imports.
`migrate radix` rewrites scattered `@radix-ui/react-*` imports to the
unified `radix-ui` package introduced February 2026.

## components.json : Complete Schema

Source : https://ui.shadcn.com/docs/components-json (verified 2026-05-19).

| Field | Type | Default | Required | Mutable after init |
|-------|------|---------|----------|--------------------|
| `$schema` | string (URL) | `https://ui.shadcn.com/schema.json` | No | Yes |
| `style` | string : `new-york` (recommended), `sera`, `luma`, `default` (deprecated) | `new-york` | Yes | **NO** |
| `rsc` | boolean | `false` | No | Yes |
| `tsx` | boolean | `true` | No | Yes |
| `tailwind.config` | string (path) OR omit | `tailwind.config.js` (v3) OR omit (v4) | Conditional | Yes |
| `tailwind.css` | string (path) | none | Yes | Yes |
| `tailwind.baseColor` | string : `neutral \| stone \| zinc \| mauve \| olive \| mist \| taupe` | none | Yes | **NO** |
| `tailwind.cssVariables` | boolean | none | Yes | **NO** |
| `tailwind.prefix` | string | none | No | Yes |
| `aliases.utils` | string (alias) | `@/lib/utils` | Yes | Yes |
| `aliases.components` | string (alias) | `@/components` | Yes | Yes |
| `aliases.ui` | string (alias) | `@/components/ui` | Yes | Yes |
| `aliases.lib` | string (alias) | `@/lib` | Yes | Yes |
| `aliases.hooks` | string (alias) | `@/hooks` | Yes | Yes |
| `iconLibrary` | string (`lucide` / `radix` / `tabler` / ...) | `lucide` | No (managed by `migrate icons`) | Yes |
| `registries` | object | none | No | Yes |

IMMUTABLE fields (`style`, `tailwind.baseColor`, `tailwind.cssVariables`)
can be CHANGED in the file, but already-installed components do NOT re-render
themselves. ALWAYS re-run `init -f` + `add --overwrite --all` if you change
an immutable field, or commit to a full migration.

## Tailwind v3 vs v4 in components.json

| Tailwind version | `tailwind.config` field | Notes |
|------------------|-------------------------|-------|
| v3.x | REQUIRED, points to `tailwind.config.js` or `.ts` | Both `cssVariables: true` and `cssVariables: false` supported |
| v4.0+ | OMIT the field entirely (CSS-first config) | shadcn detects v4 via `tailwind.css` import shape |

NEVER mix : with v4 the `config` field should be blank ; with v3 omitting it
breaks the CLI's ability to resolve theme tokens.

## Aliases : tsconfig Paths vs package.json#imports (4.7.0+)

Aliases tell the CLI WHERE to write component files AND how to rewrite
imports inside those files.

### Approach A : tsconfig.json paths (traditional)

```json
// tsconfig.json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    }
  }
}
```

```json
// components.json (excerpt)
{
  "aliases": {
    "utils": "@/lib/utils",
    "components": "@/components",
    "ui": "@/components/ui",
    "lib": "@/lib",
    "hooks": "@/hooks"
  }
}
```

### Approach B : package.json#imports (NEW in shadcn@4.7.0)

```json
// package.json (excerpt)
{
  "imports": {
    "#components/*": "./src/components/*",
    "#lib/*": "./src/lib/*",
    "#hooks/*": "./src/hooks/*"
  }
}
```

```json
// components.json (excerpt)
{
  "aliases": {
    "utils": "#lib/utils",
    "components": "#components",
    "ui": "#components/ui",
    "lib": "#lib",
    "hooks": "#hooks"
  }
}
```

Source : https://ui.shadcn.com/docs/changelog (May 2026 entry : "The `shadcn`
CLI now supports `package.json#imports` for installing components, rewriting
imports, and resolving third-party registries.").

When to use which :

- USE tsconfig paths if you are on TypeScript with an existing `@/*` setup
  and have no need for ESM-native private imports.
- USE package.json#imports if you want runtime-native private imports
  (Node.js `#prefix` ESM subpath), no build-tool transform needed, or you
  are publishing your own components-as-a-package and want consumers to use
  the same `#`-prefixed names.

NEVER define the SAME alias in BOTH places ; the resolution order is
implementation-defined and silent. Pick one approach per project.

## Custom Registries : Resolution Rules

```json
{
  "registries": {
    "@shadcn": "https://ui.shadcn.com/r/{name}.json",
    "@myorg": "https://registry.myorg.com/{name}.json",
    "@private": {
      "url": "https://registry.private.io/{name}.json",
      "headers": {
        "Authorization": "Bearer ${REGISTRY_TOKEN}"
      },
      "params": {
        "version": "stable"
      }
    }
  }
}
```

Rules verified at https://ui.shadcn.com/docs/registry :

- `{name}` placeholder REQUIRED in the URL ; the CLI substitutes the item
  name at fetch time.
- `${ENV_VAR}` interpolation supported in headers and params ; the CLI
  reads from `process.env` at invocation time.
- Custom registries appear as `@namespace` prefixes : `shadcn add @myorg/datepicker`.
- The default `@shadcn` namespace is implicit ; you only need to declare it
  if you want to override the URL or attach headers.

NEVER hard-code secrets in the URL or headers : use `${VAR_NAME}`
interpolation and store secrets in environment-level config (.env, CI
secret store, OS keychain).

## Add Command : Argument Forms

The `add` argument accepts FOUR distinct forms, resolved in this order :

1. **Bare name** : `shadcn add button` -> resolves against the default
   `@shadcn` registry as `https://ui.shadcn.com/r/button.json`.
2. **Namespaced name** : `shadcn add @myorg/datepicker` -> resolves against
   the `@myorg` entry in `registries`.
3. **Remote URL** : `shadcn add https://example.com/items/x.json` -> fetches
   directly, no namespace resolution.
4. **Local path** : `shadcn add ./registry-items/datepicker.json` -> reads
   from disk, useful for monorepo internal items.

The `-a / --all` flag works ONLY with registry items (forms 1 and 2). It
fetches the registry index and installs everything.

## Blocks : Multi-file Compositions

Blocks are full-page scaffolds (login pages, dashboards, sidebars) installed
through the SAME `add` command :

```bash
pnpm dlx shadcn@latest add login-01
pnpm dlx shadcn@latest add sidebar-07
pnpm dlx shadcn@latest add dashboard-01
```

A block installs multiple component files AND lays them out under
`aliases.components` (NOT `aliases.ui`). The block index is browsed at
https://ui.shadcn.com/blocks. Each block name maps to a single registry
item that describes the full file set.

## CLI Sync Workflow (Long-Lived Projects)

Once a project has been running shadcn for months, components drift from
upstream as bugs are fixed and new variants are added. The sync workflow :

```bash
# 1. See what changed for one component
pnpm dlx shadcn@latest add button --diff

# 2. See what changed for ALL components (heavy output)
pnpm dlx shadcn@latest add -a --diff

# 3. After reviewing : accept changes for specific components
pnpm dlx shadcn@latest add button card dialog --overwrite

# 4. Re-apply local customisations on top (git diff helps)
git diff HEAD~1 components/ui/button.tsx | review-and-port-by-hand
```

ALWAYS commit the project before `--overwrite`. The CLI does not snapshot
the previous content, and recovery without git history is manual.

## Reference Files

- `references/methods.md` : every command + flag, complete components.json
  schema fields, registries object schema
- `references/examples.md` : init flow per framework, add with --overwrite vs
  --diff, custom registry config, minimal + full components.json,
  package.json#imports alternative
- `references/anti-patterns.md` : the silent-overwrite trap, the missing
  baseColor trap, the components.json field-immutability surprises,
  v3-vs-v4 config-field gotchas

## Verified Sources

- https://ui.shadcn.com/docs/cli (command surface, all flags)
- https://ui.shadcn.com/docs/components-json (schema)
- https://ui.shadcn.com/docs/registry (registry resolution)
- https://ui.shadcn.com/docs/changelog (4.7.0 package.json#imports, sera
  style, pointer flag, partial preset apply)
