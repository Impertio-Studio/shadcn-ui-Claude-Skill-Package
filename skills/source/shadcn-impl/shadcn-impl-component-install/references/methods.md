# Methods : init, add, customise, sync

Verbatim CLI surface used by this skill. Cross-reference `shadcn-core-cli`
for the complete command catalogue ; this file is the workflow-scoped
subset.

## init : All Prompts and Flags

Interactive form (verified at https://ui.shadcn.com/docs/cli) :

```
? Framework template ? next / vite / start / react-router / laravel / astro / manual
? Base component library ? radix / base
? Name (only if scaffolding a new project) ? <project-name>
? Preset ? nova / minimal / <preset-code> / none
? Style ? new-york / sera / luma / default (deprecated)
? Base color ? neutral / stone / zinc / mauve / olive / mist / taupe
? Use CSS variables for colors ? yes / no
? Monorepo layout ? yes / no
? Enable RTL support ? yes / no
? Pointer cursor on buttons ? yes / no
```

Flags (every documented one) :

| Flag | Long form | Purpose |
|------|-----------|---------|
| `-t` | `--template <name>` | next, vite, start, react-router, laravel, astro, manual |
| `-b` | `--base <name>` | radix (default) or base |
| `-p` | `--preset <name>` | Apply a preset (theme + font combo) during init |
| `-d` | `--defaults` | next template + nova preset, no prompts |
| `-f` | `--force` | Overwrite existing components.json |
| `-n` | `--name <name>` | Project name for new-project scaffolding |
|      | `--css-variables` | Use semantic CSS variables (default `true`) |
|      | `--no-css-variables` | Inline color utilities instead of semantic tokens |
|      | `--monorepo` | Scaffold for pnpm workspace layout |
|      | `--no-monorepo` | Force single-package layout |
|      | `--rtl` | Enable RTL support during init |
|      | `--pointer` | Add `cursor-pointer` to Button (April 2026) |
|      | `--reinstall` | Re-install UI components from scratch |

`init` writes :

- `components.json` at the project root
- `lib/utils.ts` (or `aliases.utils` path) with the `cn` helper
- `tailwind.config.{js,ts}` for Tailwind v3, OR a CSS-first config for v4
- `app/globals.css` (Next.js) or equivalent with theme tokens
- Adds runtime deps via the project's package manager :
  `tailwind-merge`, `clsx`, `class-variance-authority`, `lucide-react`

## add : Every Documented Flag

| Flag | Long form | Purpose |
|------|-----------|---------|
| `-y` | `--yes` | Skip confirmation prompts |
| `-o` | `--overwrite` | Overwrite existing files (DESTROYS local edits) |
| `-a` | `--all` | Add every component in the registry |
| `-p` | `--path <path>` | Override `aliases.components` destination for this invocation |
|      | `--dry-run` | Print what would happen, write nothing to disk |
|      | `--diff [path]` | Show file diffs vs current local copy ; optional path scopes to one file |
|      | `--view [path]` | Print file contents that WOULD be written (no diff, just the content) |

Argument forms (resolution order) :

1. Bare name `add button` -> `https://ui.shadcn.com/r/button.json`
2. Namespaced `add @myorg/datepicker` -> looks up `@myorg` in
   `components.json#registries`
3. Remote URL `add https://example.com/r/x.json` -> direct fetch
4. Local path `add ./registry-items/datepicker.json` -> file read

Block names use the bare-name form too : `add sidebar-07` resolves the
block under `@shadcn` and writes multiple files.

`-a / --all` works only with forms 1 and 2 (registry-backed). It fetches
the registry index and installs everything.

## Side-effects of `add` per File-Write

For each component, `add` does this :

1. Fetch the registry JSON (item descriptor)
2. Install npm runtime deps declared by the item (per project's package
   manager : `npm install` / `pnpm install` / `yarn add` / `bun add`)
3. Write the component file(s) to `aliases.ui` (single primitive) or
   `aliases.components` (block)
4. Rewrite import paths inside the file(s) to match your aliases
5. Print a summary of files written and deps installed

If the target file already exists :

- Without `--overwrite` : the CLI SKIPS the file write (deps still install)
- With `--overwrite` : the CLI replaces the file wholesale (local edits gone)
- With `--diff` : the CLI prints a unified diff to stdout, writes nothing

## Diff-then-Overwrite Workflow

There is NO `shadcn diff` subcommand. There is NO `shadcn update` either.
The update workflow IS the `--diff` flag on `add` followed by selective
`--overwrite` :

```bash
# Step 1 : commit current state (recovery boundary).
git add . && git commit -m "Snapshot before shadcn sync"

# Step 2 : preview every component that ships in this registry.
pnpm dlx shadcn@latest add -a --diff > shadcn-diff.txt

# Step 3 : OR preview one component.
pnpm dlx shadcn@latest add button --diff

# Step 4 : decide per component. Two options each :
#   a) accept rewrite -> overwrite + reapply local changes
#   b) reject changes -> do nothing, your local file stays

# Step 5 : apply accepted rewrites.
pnpm dlx shadcn@latest add button card dialog --overwrite

# Step 6 : reapply your custom variants from the previous commit.
git diff HEAD~1 -- components/ui/button.tsx
# (manual port of your additions onto the new shape)

# Step 7 : commit the post-overwrite state.
git add . && git commit -m "Sync shadcn components + reapply custom variants"
```

## Aliases : tsconfig vs package.json#imports

| Dimension | tsconfig paths | package.json#imports |
|-----------|---------------|----------------------|
| Prefix | `@/` (or any string) | `#` REQUIRED (ESM spec) |
| Resolver | Build tool (Next.js, Vite, Astro) | Node.js native ESM |
| Source of truth | `tsconfig.json#compilerOptions.paths` | `package.json#imports` |
| Editor IntelliSense | TypeScript Language Server reads tsconfig | Same TS LS reads package.json#imports since TS 5.x |
| Build-tool transform | Required (Webpack/Vite/Turbopack resolve aliases) | NOT required (Node resolves natively) |
| Available since | shadcn 1.x (always) | shadcn 4.7.0 (May 2026) |
| Best for | Browser-only apps using a bundler | Library publishing, Node-shared code, ESM-pure projects |

Pick ONE per project. Never define the same alias in both ; the resolution
order is implementation-defined and silent.

Source : https://ui.shadcn.com/docs/changelog (May 2026 entry).

## Custom Registry : `--registry` flag and `registries` field

The `--registry` flag is a per-invocation override of which registry the
CLI talks to. The `registries` field in components.json is the persistent
declaration. The persistent form is preferred ; `--registry` is for
one-off testing.

```bash
# Per-invocation override :
pnpm dlx shadcn@latest add datepicker --registry https://my.registry.io/r/{name}.json

# Persistent (preferred) : declare in components.json, then add by namespace :
pnpm dlx shadcn@latest add @myorg/datepicker
```

Persistent declaration form :

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

Resolution rules : `{name}` placeholder REQUIRED in the URL. `${ENV_VAR}`
expansion supported in headers and params. The CLI reads `process.env` at
invocation time.

## View, Search, Docs, Info (Read-Only Inspections)

```bash
# Preview file content before installing :
pnpm dlx shadcn@latest view button card

# Search a registry :
pnpm dlx shadcn@latest search @shadcn -q "table"
pnpm dlx shadcn@latest search @shadcn -q "form" -l 10

# Fetch component docs :
pnpm dlx shadcn@latest docs dialog
pnpm dlx shadcn@latest docs dialog --json

# Print resolved project config (good for debugging "why did add do X") :
pnpm dlx shadcn@latest info
pnpm dlx shadcn@latest info --json
```

`view` and `docs` write nothing to disk. They are the safe way to preview
before `add`. `info` dumps the FINAL resolved config (after merging
components.json with defaults), the most reliable answer when a `add`
invocation seems to write to the wrong path.

## Verified Sources

- https://ui.shadcn.com/docs/cli (verified 2026-05-19 ; covers every flag listed above)
- https://ui.shadcn.com/docs/installation (verified 2026-05-19 ; per-framework init flags)
- https://ui.shadcn.com/docs/installation/vite (verified 2026-05-19 ; two-tsconfig requirement)
- https://ui.shadcn.com/docs/changelog (verified 2026-05-19 ; package.json#imports entry)
- https://ui.shadcn.com/docs/registry (verified 2026-05-19 ; `{name}` placeholder, ENV expansion)
