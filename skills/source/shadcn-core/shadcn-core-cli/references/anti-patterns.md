# shadcn CLI : Anti-Patterns

Every entry pairs a real failure mode with its root cause and the verified
fix. Sources : https://ui.shadcn.com/docs/cli,
https://ui.shadcn.com/docs/components-json, and the shadcn changelog.

## AP-1 : Running `add` Without `--diff` After Local Modifications

### Symptom
Component file in `components/ui/<name>.tsx` no longer contains the local
customisations after running `shadcn add <name>` again. Git status shows a
large diff.

### Root cause
The default behaviour of `add` against an existing file is to SKIP, but
many CI pipelines or developer scripts include `--overwrite` or `-y` flags
that bypass the skip. In either case, local edits are not preserved.

### NEVER
```bash
# Local edits in src/components/ui/button.tsx
pnpm dlx shadcn@latest add button --overwrite   # local edits gone
```

### ALWAYS
```bash
pnpm dlx shadcn@latest add button --diff        # preview
# Review the diff
pnpm dlx shadcn@latest add button --overwrite   # then accept knowingly
# Reapply local edits from git
git diff HEAD~1 src/components/ui/button.tsx
```

Commit before every `--overwrite` so the previous content is recoverable
via `git show HEAD~1:path`. The CLI does NOT snapshot.

## AP-2 : Believing `shadcn diff` Or `shadcn update` Is a Subcommand

### Symptom
Running `shadcn diff button` or `shadcn update button` errors with
"unknown command".

### Root cause
Neither subcommand exists in shadcn@4.7.0. The diff workflow is a FLAG on
`add` : `add <name> --diff`. There is no `update` subcommand at all.

### NEVER
```bash
pnpm dlx shadcn@latest diff button       # not a command
pnpm dlx shadcn@latest update button     # not a command
```

### ALWAYS
```bash
pnpm dlx shadcn@latest add button --diff
pnpm dlx shadcn@latest add button --overwrite
```

Source : https://ui.shadcn.com/docs/cli (no `diff` or `update` listed).

## AP-3 : Forgetting `aliases.utils` Resolution

### Symptom
After `add`, the new file contains `import { cn } from "@/lib/utils"` but
the IDE flags it as unresolved. The build fails with "Cannot find module
'@/lib/utils'".

### Root cause
`aliases.utils` in components.json points to a path that does NOT resolve.
Either `tsconfig.json#paths` is missing the `@/*` entry, or the project
uses package.json#imports but components.json still has `@/`-style aliases.

### NEVER assume the CLI added the tsconfig paths for you. `init` does NOT
edit tsconfig.json.

### ALWAYS verify after init :

```jsonc
// tsconfig.json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": { "@/*": ["./src/*"] }
  }
}
```

OR for the package.json#imports approach (shadcn@4.7.0+) :

```jsonc
// package.json
{
  "imports": {
    "#lib/*": "./src/lib/*",
    "#components/*": "./src/components/*"
  }
}
```

```jsonc
// components.json
{ "aliases": { "utils": "#lib/utils", "components": "#components", ... } }
```

## AP-4 : Mixing `@/`-paths and `#`-imports in the Same Project

### Symptom
After upgrade to shadcn@4.7.0 and switching aliases to `#`-form, some
generated files still contain `@/` imports. Build errors are inconsistent
across components.

### Root cause
Two-step migrations on a live codebase leave a mix of old and new aliases.
The CLI only rewrites NEWLY added components ; previously installed files
keep their original imports.

### NEVER leave the project in a half-migrated state.

### ALWAYS migrate in one commit :
1. Update components.json `aliases` to the new form.
2. Re-run `pnpm dlx shadcn@latest add -a --overwrite -y` to regenerate all
   component files with the new alias.
3. Manually update any non-shadcn code that imported from `@/...`.
4. Remove the `@/*` entry from tsconfig if no longer needed.

## AP-5 : Trying to Change `baseColor` or `style` After Init

### Symptom
Changed `tailwind.baseColor` from `zinc` to `slate` in components.json.
Existing components still show `zinc` palette. Adding a new component shows
the new palette. Visual inconsistency across the app.

### Root cause
`style`, `tailwind.baseColor`, and `tailwind.cssVariables` are IMMUTABLE
after init in the sense that already-installed components do NOT regenerate
themselves when the field changes. Each component holds its colour palette
inline (when `cssVariables: false`) or via CSS variables resolved at
install time (when `cssVariables: true`).

### NEVER change an immutable field and expect a hot reload to fix it.

### ALWAYS treat a baseColor change as a full re-install :
```bash
# Backup uncommitted edits first
git stash
# Change components.json
pnpm dlx shadcn@latest init -f                # re-confirm config
pnpm dlx shadcn@latest add -a --overwrite -y  # regenerate every component
# Then restore your custom edits manually
git stash pop
```

Source : https://ui.shadcn.com/docs/components-json (immutability note on
`style`, `baseColor`, `cssVariables`).

## AP-6 : `tailwind.config` Field Set When Using Tailwind v4

### Symptom
After upgrading the project to Tailwind v4, `shadcn add` succeeds but
generated components show wrong colours or fail at runtime with "unknown
utility class".

### Root cause
v4 uses CSS-first config via `@theme { ... }` in the stylesheet. There is
no `tailwind.config.js` file. components.json's `tailwind.config` field
should be EMPTY or omitted entirely. Leaving it pointing at a stale
`tailwind.config.js` confuses the CLI's theme resolution.

### NEVER

```jsonc
// components.json on Tailwind v4
{
  "tailwind": {
    "config": "tailwind.config.js",   // WRONG : file doesn't exist
    "css": "src/index.css"
  }
}
```

### ALWAYS

```jsonc
// components.json on Tailwind v4
{
  "tailwind": {
    "config": "",                      // empty string OR omit
    "css": "src/index.css"
  }
}
```

For Tailwind v3, the opposite : `config` is REQUIRED and must point at the
actual config file.

## AP-7 : Wrong `style` Enum Value

### Symptom
`shadcn add button` fails with "invalid style". Or : a community guide tells
you to set `"style": "stone-shadows"` and that fails.

### Root cause
Only specific values are accepted. Verified set as of shadcn@4.7.0 :

- `new-york` (recommended, current default)
- `sera` (typography-first, since 4.3.0)
- `luma` (since 4.1.2)
- `default` (DEPRECATED, no new components ship against it)

### NEVER guess a style name from a screenshot or third-party preset.

### ALWAYS verify against `pnpm dlx shadcn@latest preset decode <code>` if
applying a preset, or against https://ui.shadcn.com/docs/components-json.

## AP-8 : Missing `{name}` Placeholder in Registry URL

### Symptom
Custom registry is configured but `shadcn add @myorg/datepicker` fetches
the wrong URL or returns 404.

### Root cause
The `{name}` placeholder is REQUIRED in the registry URL. Without it, the
CLI cannot substitute the item name.

### NEVER

```jsonc
{
  "registries": {
    "@myorg": "https://registry.myorg.com/datepicker.json"  // hard-coded
  }
}
```

### ALWAYS

```jsonc
{
  "registries": {
    "@myorg": "https://registry.myorg.com/{name}.json"
  }
}
```

Or use the object form :

```jsonc
{
  "registries": {
    "@myorg": {
      "url": "https://registry.myorg.com/{name}.json",
      "headers": { "Authorization": "Bearer ${TOKEN}" }
    }
  }
}
```

## AP-9 : Hard-coded Secrets in Registry Headers

### Symptom
`components.json` was committed to a public repo with an `Authorization: Bearer
<actual-token>` header. The token is now leaked.

### Root cause
Headers in the `registries` object should reference environment variables,
not embed secret values directly.

### NEVER

```jsonc
{
  "registries": {
    "@private": {
      "url": "https://registry.private.io/{name}.json",
      "headers": {
        "Authorization": "Bearer sk_live_8f7a3..."     // LEAKED on commit
      }
    }
  }
}
```

### ALWAYS

```jsonc
{
  "registries": {
    "@private": {
      "url": "https://registry.private.io/{name}.json",
      "headers": {
        "Authorization": "Bearer ${REGISTRY_TOKEN}"
      }
    }
  }
}
```

Store `REGISTRY_TOKEN` in `.env.local` (gitignored) or CI secret store.
The CLI interpolates at fetch time.

## AP-10 : Using `npm install shadcn-ui` Instead of the CLI

### Symptom
`npm install shadcn-ui` succeeds and adds it to `dependencies`. But
`import { Button } from 'shadcn-ui'` does not exist. Documentation does not
describe a runtime import path.

### Root cause
`shadcn-ui` on npm is a legacy stub. shadcn is NOT a runtime library : it
is a CLI distribution tool that COPIES source files into your project.
There is no Button to import from a node_modules package.

### NEVER

```bash
npm install shadcn-ui          # legacy stub
npm install @shadcn/ui         # does not exist
```

### ALWAYS

```bash
pnpm dlx shadcn@latest init
pnpm dlx shadcn@latest add button
# Then : import { Button } from "@/components/ui/button"
```

Source : https://ui.shadcn.com/docs (paradigm section).

## AP-11 : Forgetting Tailwind v3 Plugin Installs Before `init`

### Symptom
On Tailwind v3 projects, `shadcn add form` succeeds but form layouts look
broken. Inputs are unstyled.

### Root cause
Some shadcn components depend on `@tailwindcss/forms` or `@tailwindcss/typography`
v3 plugins. The CLI does NOT auto-install them ; the consumer must add the
plugins to `tailwind.config.js` manually.

### ALWAYS install required Tailwind v3 plugins explicitly :

```bash
npm install -D @tailwindcss/forms @tailwindcss/typography
```

```js
// tailwind.config.js
module.exports = {
  plugins: [
    require('@tailwindcss/forms'),
    require('@tailwindcss/typography'),
  ],
}
```

On Tailwind v4 the equivalent is `@plugin "@tailwindcss/forms";` inside the
main stylesheet. Same need : the shadcn CLI does NOT add it.

## AP-12 : Running `init` Twice Without `--force`

### Symptom
Running `pnpm dlx shadcn@latest init` a second time prompts "components.json
already exists". User answers `n` (no), invocation exits, but the new
template flag they passed is not applied.

### Root cause
`init` is a one-shot bootstrap. Re-running it requires `-f` and overwrites
components.json wholesale ; it does NOT merge.

### NEVER expect `init` to merge new flags into an existing components.json.

### ALWAYS
- Edit components.json directly for incremental config changes.
- Use `init -f` only when committing to a full reset (and commit beforehand).

## AP-13 : `--all` Combined With a Custom Registry That Has No Index

### Symptom
`pnpm dlx shadcn@latest add @myorg -a` fails with "registry index not found".

### Root cause
The `-a / --all` flag requires the registry to publish an index document at
the same URL pattern, listing every item. Many small custom registries
publish individual item JSONs but no index.

### ALWAYS publish a `registry.json` index when authoring a custom registry
that consumers will fetch with `-a`. The `shadcn build` command writes one
automatically.

## AP-14 : Treating `add` Output Path as Permanent

### Symptom
A team agrees to move components from `src/components/ui/` to
`src/ui/components/`. Files are moved, but next `add` invocation writes the
new component back into `src/components/ui/`.

### Root cause
`add` reads the destination from `aliases.components` (or
`aliases.ui` for ui-tagged items) at INVOCATION time. Moving files on disk
does not update components.json.

### ALWAYS update components.json's `aliases` BEFORE moving files :

```jsonc
{
  "aliases": {
    "components": "@/ui/components",
    "ui": "@/ui/components/ui"
  }
}
```

Then move existing files into the new path, then run `add --overwrite` to
verify the CLI writes to the new destination.

## AP-15 : Migrate `radix` Without Reading the Diff

### Symptom
After `pnpm dlx shadcn@latest migrate radix`, several Radix primitives
import from a new namespace style and a few custom integrations break.

### Root cause
`migrate radix` rewrites scattered `@radix-ui/react-<part>` imports to the
unified `radix-ui` namespace package introduced February 2026. Custom code
that imported types or sub-namespaces in non-standard ways may not be
rewritten cleanly.

### ALWAYS
- Run on a clean branch.
- Review the diff before committing.
- Pass an explicit glob to scope the migration : `migrate radix "src/components/ui/**"`
  so non-shadcn integrations are left untouched.
