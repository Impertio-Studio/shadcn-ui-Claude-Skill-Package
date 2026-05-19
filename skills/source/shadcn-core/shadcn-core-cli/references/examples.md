# shadcn CLI : Examples

All examples verified against https://ui.shadcn.com/docs/cli and
https://ui.shadcn.com/docs/components-json (2026-05-19).

## 1. Init Per Framework

### Next.js (App Router)

```bash
pnpm dlx shadcn@latest init -t next
```

Interactive prompts then write components.json + install
`tailwind-merge`, `clsx`, `class-variance-authority`, `lucide-react`. Adds
`lib/utils.ts` with the `cn()` helper.

### Vite + React

```bash
pnpm dlx shadcn@latest init -t vite
```

### TanStack Start

```bash
pnpm dlx shadcn@latest init -t start
```

### React Router (v7+)

```bash
pnpm dlx shadcn@latest init -t react-router
```

### Astro

```bash
pnpm dlx shadcn@latest init -t astro
```

### Laravel (Inertia + React)

```bash
pnpm dlx shadcn@latest init -t laravel
```

### Defaults, no prompts (fast path)

```bash
pnpm dlx shadcn@latest init -d
```

Selects next template + nova preset.

### Force re-init (overwrite existing components.json)

```bash
pnpm dlx shadcn@latest init -f
```

ALWAYS commit before `init -f` ; the existing config is overwritten without
backup.

## 2. Add : Common Workflows

### Single component

```bash
pnpm dlx shadcn@latest add button
```

### Multiple components in one invocation

```bash
pnpm dlx shadcn@latest add button card dialog input form
```

### Every component (`-a` / `--all`)

```bash
pnpm dlx shadcn@latest add -a
```

### Block (multi-file scaffold)

```bash
pnpm dlx shadcn@latest add login-01
pnpm dlx shadcn@latest add sidebar-07
pnpm dlx shadcn@latest add dashboard-01
```

### From a remote URL

```bash
pnpm dlx shadcn@latest add https://example.com/registry-items/datepicker.json
```

### From a local JSON file

```bash
pnpm dlx shadcn@latest add ./local-registry/items/datepicker.json
```

### From a namespaced custom registry

```bash
pnpm dlx shadcn@latest add @myorg/datepicker @myorg/timepicker
```

## 3. Diff and Overwrite : the Update Workflow

### Step 1 : Preview what would change

```bash
pnpm dlx shadcn@latest add button --diff
```

Output : a unified diff of the local file vs the registry source. Nothing
is written.

### Step 2 : Preview multiple components

```bash
pnpm dlx shadcn@latest add button card dialog --diff
```

### Step 3 : Preview EVERY component (heavy)

```bash
pnpm dlx shadcn@latest add -a --diff
```

### Step 4 : Accept changes for one component

```bash
pnpm dlx shadcn@latest add button --overwrite
```

### Step 5 : Accept changes for many

```bash
pnpm dlx shadcn@latest add button card dialog --overwrite -y
```

### Step 6 : Reapply local customisations

After `--overwrite`, local edits are GONE. Use git to recover the previous
content :

```bash
git diff HEAD~1 src/components/ui/button.tsx
```

Then port the relevant changes by hand onto the freshly written file.

## 4. Dry-Run : See What Would Happen Without Writing

```bash
pnpm dlx shadcn@latest add button --dry-run
```

Useful in CI to detect drift without mutating the working tree.

## 5. Custom Path Override

```bash
pnpm dlx shadcn@latest add button -p ./packages/ui/src/components/ui
```

Useful in monorepos where `aliases.components` does not match the desired
destination for one particular invocation.

## 6. View Without Writing

```bash
pnpm dlx shadcn@latest view button
pnpm dlx shadcn@latest view button --path ./inspect.tsx
```

Prints the file contents that `add` would write. With `--path`, writes to
the inspect file instead of stdout.

## 7. Search a Registry

```bash
pnpm dlx shadcn@latest search @shadcn -q "table"
pnpm dlx shadcn@latest search @shadcn @myorg -q "calendar" -l 20
pnpm dlx shadcn@latest search @shadcn -q "form" -l 50 -o 50    # next page
```

## 8. Apply a Preset (Theme + Font)

```bash
pnpm dlx shadcn@latest apply a2r6bw                       # both theme + font
pnpm dlx shadcn@latest apply a2r6bw --only theme          # theme only
pnpm dlx shadcn@latest apply a2r6bw --only font           # font only
pnpm dlx shadcn@latest preset decode a2r6bw               # see what's inside
pnpm dlx shadcn@latest preset decode a2r6bw --json        # machine-readable
pnpm dlx shadcn@latest preset url a2r6bw                  # shareable URL
pnpm dlx shadcn@latest preset open a2r6bw                 # browser preview
pnpm dlx shadcn@latest preset resolve                     # active preset
```

## 9. Build a Custom Registry (Publishers)

Project root has a `registry.json` describing the items :

```jsonc
{
  "$schema": "https://ui.shadcn.com/schema/registry.json",
  "name": "@myorg",
  "homepage": "https://registry.myorg.com",
  "items": [
    {
      "name": "datepicker",
      "type": "registry:ui",
      "files": [{ "path": "src/datepicker.tsx", "type": "registry:ui" }],
      "registryDependencies": ["button", "calendar"]
    }
  ]
}
```

Then :

```bash
pnpm dlx shadcn@latest build
# Writes ./public/r/registry.json + ./public/r/datepicker.json
```

```bash
pnpm dlx shadcn@latest build ./other.json -o ./out
```

Host `./public/r/` behind any static server ; consumers reference it via
their `components.json#registries`.

## 10. Migrations

### List available

```bash
pnpm dlx shadcn@latest migrate -l
```

### Swap icon library

```bash
pnpm dlx shadcn@latest migrate icons
```

Prompts for target library (`lucide`, `radix`, `tabler`, `heroicons`, etc.)
and rewrites every icon import + updates `iconLibrary` in components.json.

### Move to unified `radix-ui`

```bash
pnpm dlx shadcn@latest migrate radix
```

Rewrites `import * from '@radix-ui/react-dialog'` to
`import * from 'radix-ui'` namespace form. Optional file or glob :

```bash
pnpm dlx shadcn@latest migrate radix src/components/ui/dialog.tsx
pnpm dlx shadcn@latest migrate radix "src/components/ui/**"
```

### Add RTL support

```bash
pnpm dlx shadcn@latest migrate rtl
```

## 11. components.json : Minimal Sample (Vite + React + Tailwind v4)

```json
{
  "$schema": "https://ui.shadcn.com/schema.json",
  "style": "new-york",
  "rsc": false,
  "tsx": true,
  "tailwind": {
    "config": "",
    "css": "src/index.css",
    "baseColor": "zinc",
    "cssVariables": true
  },
  "aliases": {
    "utils": "@/lib/utils",
    "components": "@/components",
    "ui": "@/components/ui",
    "lib": "@/lib",
    "hooks": "@/hooks"
  },
  "iconLibrary": "lucide"
}
```

Note : `tailwind.config` is an empty string for v4 (CSS-first config).

## 12. components.json : Full Sample (Next.js + Tailwind v3 + custom registries)

```json
{
  "$schema": "https://ui.shadcn.com/schema.json",
  "style": "new-york",
  "rsc": true,
  "tsx": true,
  "tailwind": {
    "config": "tailwind.config.ts",
    "css": "app/globals.css",
    "baseColor": "zinc",
    "cssVariables": true,
    "prefix": ""
  },
  "aliases": {
    "utils": "@/lib/utils",
    "components": "@/components",
    "ui": "@/components/ui",
    "lib": "@/lib",
    "hooks": "@/hooks"
  },
  "iconLibrary": "lucide",
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

## 13. package.json#imports Alternative to tsconfig Paths (shadcn@4.7.0+)

### Step A : declare imports in package.json

```jsonc
{
  "name": "my-app",
  "imports": {
    "#components/*": "./src/components/*",
    "#components/ui/*": "./src/components/ui/*",
    "#lib/*": "./src/lib/*",
    "#hooks/*": "./src/hooks/*"
  }
}
```

### Step B : reference them in components.json

```json
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

### Step C : add a component

```bash
pnpm dlx shadcn@latest add button
```

Generated `src/components/ui/button.tsx` will contain
`import { cn } from "#lib/utils"` instead of
`import { cn } from "@/lib/utils"`.

Source : https://ui.shadcn.com/docs/changelog (May 2026, 4.7.0).

## 14. Monorepo Init

```bash
pnpm dlx shadcn@latest init --monorepo
```

Creates a workspace-aware components.json. Pair with an `-p` path override
in subsequent `add` invocations to target a specific package :

```bash
pnpm dlx shadcn@latest add button -p ./packages/ui/src/components/ui
```

## 15. CI Smoke Test : Confirm Project Is Healthy

```bash
pnpm dlx shadcn@latest info --json > /tmp/shadcn-info.json
```

If exit code is 0 and the JSON contains `style`, `tailwind.baseColor`, and
resolvable `aliases`, the project is correctly configured. Use this in a CI
step to fail fast if a contributor checked in a broken components.json.
