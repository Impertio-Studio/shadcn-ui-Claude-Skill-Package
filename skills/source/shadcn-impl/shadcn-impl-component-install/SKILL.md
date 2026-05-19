---
name: shadcn-impl-component-install
description: >
  Use when adding a new component to a project that already uses shadcn, when
  bootstrapping shadcn for the first time in a fresh app, when deciding
  between `shadcn add`, a hand-built primitive, or a pre-built block, when
  wiring path aliases (tsconfig paths or the new package.json#imports
  resolver from shadcn@4.7.0), when re-running `add` against a file you
  have already customised, or when an `add` invocation has surfaced broken
  imports or missing radix dependencies. Prevents the silent-overwrite
  destruction from running `add --overwrite` without `--diff` first, the
  broken-imports trap from running `add` before `init`, the alias-mismatch
  bug where `tsconfig` paths and `components.json` aliases disagree, the
  bypass-the-CLI mistake of pip-installing `@radix-ui/react-*` packages by
  hand instead of letting `add` resolve them, and the long-tail stale-code
  problem from never re-running the CLI after upstream bug fixes ship.
  Covers the full add-or-customise-or-block decision tree, the
  init-then-add-then-customise workflow, components.json key fields with
  pointers into shadcn-core-cli, the two alias strategies (tsconfig vs
  package.json#imports) with selection criteria, post-add manual fixups
  (dependency verification, type-import paths, where customisation lives),
  the diff-before-overwrite update workflow for long-lived projects, the
  block-add shortcut (forwarded to shadcn-core-blocks), and the
  custom-registry add form with the `--registry` flag.
  Keywords: shadcn install, how do I add a component, npx shadcn add,
  pnpm dlx shadcn add, install button shadcn, install dialog shadcn,
  components.json setup, components json configure, aliases setup,
  tsconfig paths shadcn, package.json imports shadcn, hash-prefix imports,
  shadcn customize component, shadcn customise component, edit shadcn
  button, override shadcn variant, post-add fixup, CLI update workflow,
  shadcn diff before overwrite, --diff flag, --overwrite flag, sync shadcn
  components, refresh shadcn components, what changed in shadcn registry,
  shadcn add block, shadcn add login page, shadcn add dashboard,
  --registry flag, custom registry add, third party registry shadcn,
  shadcn missing dependency, broken imports after add, where do components
  live shadcn, getting started shadcn, first time shadcn setup.
license: MIT
compatibility: "Designed for Claude Code. Requires shadcn ui evergreen-2026 (CLI shadcn@4.7.0)."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# shadcn ui : End-to-End Component Install Workflow

This skill is the workflow on top of the CLI. It tells you the SEQUENCE of
steps from empty repo to customised, version-controlled component file, and
the SAFE re-run pattern when upstream changes ship. For raw CLI command
surface (every flag, every subcommand), go to `shadcn-core-cli`. For
registry authoring and resolution rules, go to `shadcn-core-registry`. For
multi-file page scaffolds, go to `shadcn-core-blocks`.

Companion skills :

- `shadcn-core-cli` : every CLI command, every flag, components.json schema
- `shadcn-core-architecture` : code-distribution paradigm, ownership model
- `shadcn-core-registry` : registry item schema, resolution semantics
- `shadcn-core-blocks` : multi-file page scaffolds (dashboards, login pages)

## Quick Reference : The Three Beats

ALWAYS install a component in this order. NEVER skip a beat.

```
1. init       -> pnpm dlx shadcn@latest init
                 (once per project ; creates components.json + deps)
2. add        -> pnpm dlx shadcn@latest add <name>
                 (per component ; copies source into components/ui/)
3. customise  -> edit components/ui/<name>.tsx in place
                 (variants, props, default styles ; commit to git)
```

The output of step 2 is OWNED code. After it lands, you are the
maintainer. There is no upgrade path through npm ; the only upgrade
mechanism is re-running `add` with `--diff` and merging by hand.

## Quick Reference : Add or Customise or Block ?

```
What are you building ?
├── A single primitive that does not exist in shadcn yet (e.g., a price-tag
│   pill, a domain-specific badge variant)
│   └── COMPOSE shadcn primitives first. NEVER hand-roll a new "Button" or
│       "Dialog". Custom code lives in `components/<name>.tsx` (NOT
│       components/ui/, that path is reserved for CLI-managed files).
│
├── A primitive shadcn already ships (Button, Dialog, Form, Input, ...)
│   └── `pnpm dlx shadcn@latest add <name>` -> then customise in place.
│       NEVER hand-write a re-implementation of a shadcn primitive.
│
├── A whole page or large UI surface (dashboard, login, sidebar shell)
│   └── Use a BLOCK. `pnpm dlx shadcn@latest add sidebar-07` (or browse
│       https://ui.shadcn.com/blocks). See `shadcn-core-blocks` for the
│       full block doctrine.
│
└── A component from a third-party / private design system
    └── Configure the `registries` entry in components.json, then
        `pnpm dlx shadcn@latest add @myorg/<name>`. See
        `shadcn-core-registry` for registry config rules.
```

ALWAYS prefer `add` over re-implementing. The CLI resolves runtime
dependencies (radix primitives, cva, sonner, vaul, cmdk, react-day-picker,
input-otp) automatically. Hand-rolling skips that resolution and ships a
file with broken imports.

## init : Bootstrap Once Per Project

```bash
# Default Next.js + nova preset, no prompts :
pnpm dlx shadcn@latest init -d

# Interactive (recommended for first-time projects) :
pnpm dlx shadcn@latest init

# Vite + React project (after the manual tsconfig setup ; see below) :
pnpm dlx shadcn@latest init -t vite

# Astro project :
pnpm dlx shadcn@latest init -t astro
```

Interactive `init` asks for : framework template, base library (radix vs
base), preset, baseColor, cssVariables, monorepo layout, RTL. Outputs
`components.json` at the project root and installs runtime deps
(`tailwind-merge`, `clsx`, `class-variance-authority`, `lucide-react`).

ALWAYS commit `components.json` to git. It is the single source of truth
for where the CLI writes files and which import-alias style your project
uses. Without it the next `add` invocation will prompt you for the entire
config again.

## components.json : the Fields You Touch Most

| Field | Purpose | Common value |
|-------|---------|--------------|
| `style` | Visual language (immutable after init) | `"new-york"` (recommended) |
| `tailwind.baseColor` | Neutral palette (immutable) | `"neutral"` / `"zinc"` / `"stone"` |
| `tailwind.cssVariables` | Semantic tokens (immutable) | `true` for theming, `false` for inline utilities |
| `aliases.utils` | Where `cn()` lives | `"@/lib/utils"` |
| `aliases.ui` | Where shadcn copies primitives | `"@/components/ui"` |
| `aliases.components` | Where custom components live (also blocks) | `"@/components"` |
| `registries` | Custom / private registry namespaces | `{ "@myorg" : "..." }` |

Full schema with every field, every default, every mutability rule, and
every Tailwind v3-vs-v4 difference : see `shadcn-core-cli` "components.json
: Complete Schema".

NEVER change `style`, `baseColor`, or `cssVariables` post-init by hand-
editing the file alone. Those fields are IMMUTABLE in the sense that
already-installed components do NOT regenerate themselves. The supported
"change my mind" path is : `init -f` (rewrites components.json) THEN
`add --overwrite --all` (rewrites every installed component file).

## Aliases : tsconfig Paths or package.json#imports ?

shadcn rewrites import paths inside the copied files so they match YOUR
project's alias scheme. Two schemes are supported as of shadcn@4.7.0.

### Approach A : tsconfig paths (traditional, most projects)

```json
// tsconfig.json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": { "@/*": ["./src/*"] }
  }
}

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

ALWAYS use tsconfig paths when : the project is already TypeScript-based,
`@/*` is already wired, you do not need ESM-native subpath imports, the
tooling chain (Next.js, Vite, Astro, etc.) is the import resolver.

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

USE package.json#imports when : the project needs runtime-native private
imports (Node.js `#prefix` ESM subpaths, no build-tool transform), you are
publishing your components as a package and want consumers to use the same
`#`-prefixed names, you want a single resolver source-of-truth without
maintaining tsconfig paths in parallel.

NEVER define the same alias in BOTH systems for the same project. The
resolution order is implementation-defined and silent. Pick one approach
per project at init time.

Source : https://ui.shadcn.com/docs/changelog (May 2026 entry).

### Vite-specific : Two tsconfig Files

Vite splits TypeScript config across `tsconfig.json` AND `tsconfig.app.json`.
ALWAYS add the alias entry to BOTH files. Missing one of them leaves either
editor IntelliSense broken (resolves at runtime but red squiggle in VS Code)
or runtime broken (compiles in editor but fails at vite-build). Then add
the alias to `vite.config.ts` `resolve.alias` as well. The full three-file
setup is shown in `examples.md` "Vite + React greenfield setup".

## add : The Per-Component Command

```bash
# Bare name : resolves against @shadcn (default) registry :
pnpm dlx shadcn@latest add button

# Multiple at once :
pnpm dlx shadcn@latest add button card dialog form

# Namespaced (custom registry) :
pnpm dlx shadcn@latest add @myorg/datepicker

# Remote URL :
pnpm dlx shadcn@latest add https://example.com/r/x.json

# Local registry item :
pnpm dlx shadcn@latest add ./registry/datepicker.json

# Block (multi-file page scaffold) :
pnpm dlx shadcn@latest add sidebar-07
```

What happens during `add` :

1. CLI reads `components.json` to know aliases and base.
2. CLI fetches the registry item JSON.
3. CLI installs npm runtime dependencies declared by the item
   (radix primitives, cva, lucide, sonner, vaul, etc.).
4. CLI writes file(s) into `aliases.ui` (single primitives) or
   `aliases.components` (blocks).
5. CLI rewrites import paths inside the file(s) to match your aliases.

NEVER hand-install `@radix-ui/react-*`, `cmdk`, `vaul`, `react-day-picker`,
`input-otp`, `sonner` packages on your own when adding a shadcn component.
The CLI knows the exact set the component depends on and the exact versions
the registry has verified. Hand-installing skips this and frequently
mismatches versions.

If `add` encounters a file that already exists at the target path, it
SKIPS by default. Pass `--overwrite` to force a fresh copy.

## Post-Add Fixups (Manual Steps After `add`)

For most components `add` is end-to-end. The five cases where you DO need
to touch something by hand :

1. **`cn` import sanity check** : open the new file, verify the first
   imports look like `import { cn } from "@/lib/utils"` (or your alias).
   If the import path is wrong, your `aliases.utils` is misconfigured ;
   fix components.json AND re-run `add <name> --overwrite`.

2. **Tailwind v4 CSS animation imports** : if the component animates
   (Accordion, Sheet, Dialog, Drawer, Sidebar) and your project is on
   Tailwind v4, your global CSS file MUST import `tw-animate-css` or
   the keyframes are missing. `init` adds this for new projects, but
   migrating a v3 project to v4 leaves the import absent. See
   `shadcn-core-stack` "Tailwind v3 vs v4".

3. **Server-Component boundary** : when `rsc: true` in components.json,
   `add` omits `"use client"` from pure-presentation files (Card, Badge,
   Alert, Skeleton, Separator). Components with state or refs (Dialog,
   Sheet, Drawer, Popover, Sidebar, Form, Calendar) keep `"use client"`.
   Verify the directive is present where the component uses hooks.

4. **`react-day-picker` version (Calendar only)** : Calendar wraps
   `react-day-picker` v9 since mid-2025. If your project pinned v8 from
   an older shadcn install, `add calendar --overwrite` updates the file
   but does NOT bump the dep. Run `pnpm up react-day-picker` after.

5. **First add to `aliases.ui` directory** : the first `add` invocation
   creates the `components/ui/` directory. ALWAYS commit the directory
   shell to git after init even if empty, so the alias resolves on fresh
   clones before any components are added.

## Customisation : Edit in Place, Commit, Move On

The owned-code model means customisation IS editing the file in place :

```tsx
// components/ui/button.tsx (post add, post customise)
const buttonVariants = cva(
  "inline-flex items-center justify-center ...",
  {
    variants: {
      variant: {
        default: "bg-primary text-primary-foreground ...",
        destructive: "bg-destructive text-white ...",
        outline: "border bg-background ...",
        // ADDED for this project :
        brand: "bg-brand-500 text-white hover:bg-brand-600 ...",
      },
      size: {
        default: "h-9 px-4 py-2",
        sm: "h-8 px-3 ...",
        lg: "h-10 px-6 ...",
        icon: "size-9",
        // ADDED for this project :
        xl: "h-12 px-8 text-lg",
      },
    },
    defaultVariants: { variant: "default", size: "default" },
  }
)
```

ALWAYS commit customisations to git in a separate commit from the raw
`add` output. This is the ONLY way to identify your edits when a later
`--diff` reveals registry changes (the diff base is the registry's
current shape, NOT your last `--overwrite`). The git log between the
"add" commit and your "customise" commits IS your local-change history.

NEVER add files to `components/ui/` that did not come from the CLI. That
path is reserved for CLI-managed files. Custom compositions live in
`components/<name>.tsx` (under `aliases.components`).

## Update Workflow : Diff Before Overwrite

Components drift from upstream over time : new variants ship, bugs get
fixed, accessibility wiring improves. Refusing to ever re-run the CLI is
the slow-rot anti-pattern : your local Button is six months behind, the
new `cursor-pointer` flag, the new `data-icon` slot, the new `size-icon-sm`,
the new aria-attribute, never arrive.

The safe sync workflow :

```bash
# 1. Preview ONE component :
pnpm dlx shadcn@latest add button --diff

# 2. Preview ALL components (heavy output, capture to file) :
pnpm dlx shadcn@latest add -a --diff > shadcn-diff.txt

# 3. Decide per component : accept rewrite, or merge by hand.
#    If you accept rewrite :
pnpm dlx shadcn@latest add button --overwrite
#    Then re-apply your custom variants from the previous commit (git diff
#    against the pre-overwrite commit shows exactly what to port).

# 4. Commit BEFORE running --overwrite. Always.
git add . && git commit -m "Snapshot before shadcn sync"
pnpm dlx shadcn@latest add button card dialog --overwrite
```

ALWAYS run `--diff` before `--overwrite` on any file you have edited.
NEVER assume re-running `add` is safe : when the file exists, `add` skips
by default, `--overwrite` replaces wholesale. There is no automatic merge.

NEVER skip the pre-overwrite commit. The CLI does not snapshot the previous
content ; recovery without a git commit is manual.

There is no `shadcn diff` SUBCOMMAND despite some blog posts claiming
otherwise. `--diff` is a FLAG on `add`. There is no `shadcn update` either ;
the diff-then-overwrite pair IS the update workflow.

## Block-Add : Forward to shadcn-core-blocks

```bash
# Login pages :
pnpm dlx shadcn@latest add login-01
pnpm dlx shadcn@latest add login-03

# Sidebars :
pnpm dlx shadcn@latest add sidebar-07

# Dashboards :
pnpm dlx shadcn@latest add dashboard-01
```

Blocks are added through the SAME `add` command, but write multiple files
to `aliases.components` (NOT `aliases.ui`). Block file ownership semantics
match component ownership : edit in place, commit, re-run with `--diff`
when you want to pull upstream improvements. Full block doctrine, gallery
landscape, style enum guidance, and the block-vs-component decision tree
live in `shadcn-core-blocks`.

## Custom Registry Add : the `--registry` Form

For a third-party or internal design system, configure the `registries`
field in components.json :

```json
{
  "registries": {
    "@myorg": "https://registry.myorg.com/{name}.json",
    "@private": {
      "url": "https://private.registry.io/{name}.json",
      "headers": { "Authorization": "Bearer ${REGISTRY_TOKEN}" },
      "params": { "version": "stable" }
    }
  }
}
```

Then add by namespace :

```bash
pnpm dlx shadcn@latest add @myorg/datepicker
pnpm dlx shadcn@latest add @private/billing-form
```

Resolution rules (verified at https://ui.shadcn.com/docs/registry) :

- `{name}` placeholder REQUIRED in the URL ; the CLI substitutes the item
  name at fetch time.
- `${ENV_VAR}` interpolation supported in headers and params ; the CLI
  reads from `process.env` at invocation time.
- Custom registries appear as `@namespace` prefixes.
- The default `@shadcn` is implicit ; declare it only to override the URL.

NEVER hard-code secrets in URL or headers. ALWAYS interpolate via
`${VAR_NAME}` and store the secret in `.env` / CI secret store / OS keychain.

Full registry item schema, authoring rules, and the `build` command for
publishers : see `shadcn-core-registry`.

## Reference Files

- `references/methods.md` : full init prompts, every `add` flag, the
  diff-and-overwrite workflow commands, the aliases comparison table
- `references/examples.md` : Next.js greenfield setup, Vite greenfield
  setup with both tsconfig files, add Button + customise variant,
  add Form + integrate with existing context, update workflow with diff,
  package.json#imports setup, custom registry add
- `references/anti-patterns.md` : init-skip trap, missing-diff trap,
  alias mismatch, hand-installed radix versions, never-re-running-the-CLI

## Verified Sources

- https://ui.shadcn.com/docs/cli (CLI command surface, flags ; verified 2026-05-19)
- https://ui.shadcn.com/docs/installation (per-framework setup index ; verified 2026-05-19)
- https://ui.shadcn.com/docs/installation/vite (Vite two-tsconfig requirement ; verified 2026-05-19)
- https://ui.shadcn.com/docs/components-json (schema ; verified 2026-05-19)
- https://ui.shadcn.com/docs/registry (registry resolution ; verified 2026-05-19)
- https://ui.shadcn.com/docs/changelog (4.7.0 package.json#imports entry ; verified 2026-05-19)
