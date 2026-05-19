# Anti-Patterns : shadcn Core Architecture

The four canonical mental-model failures. Each entry follows : symptom, root cause, fix.

## 1. `npm install shadcn-ui` (or any variant)

### Symptom

```bash
npm install shadcn-ui
# npm error code E404
# npm error 404 Not Found - GET https://registry.npmjs.org/shadcn-ui - Not found
```

Or, in some cases, an unrelated tarball with a similar name installs and produces a confusing dependency.

### Root cause

There is **no runtime npm package** called `shadcn-ui`. The shadcn project is :
- A CLI npm package called `shadcn` (currently `4.7.0`), invoked via `dlx`/`npx`.
- A registry of component source files at https://ui.shadcn.com/registry.

The CLI copies source files into the consumer's repository at install time. There is nothing to install at runtime.

### Fix

ALWAYS use the CLI invocation pattern :

```bash
# Initialise once
pnpm dlx shadcn@latest init

# Add components
pnpm dlx shadcn@latest add button dialog dropdown-menu

# Or with npm / yarn / bun
npx shadcn@latest add button
yarn dlx shadcn@latest add button
bunx shadcn@latest add button
```

The `dlx`/`npx` invocation downloads the CLI, runs it once, and discards it after the command finishes. The CLI's job is to write files into your project, not to remain installed as a runtime dependency.

## 2. Importing from `shadcn-ui` or `@shadcn/ui`

### Symptom

```tsx
import { Button } from 'shadcn-ui'                   // Module not found
import { Button } from '@shadcn/ui'                  // Module not found
import { Button } from '@shadcn/ui/components'       // Module not found
```

Build fails with `Cannot find module 'shadcn-ui'` or `Module not found: Error: Can't resolve '@shadcn/ui'`.

### Root cause

After `shadcn add button`, the component source lives at `components/ui/button.tsx` inside the consumer's repository. The import path resolves via the path alias configured in `components.json` (`aliases.ui`, default `@/components/ui`) and the project's `tsconfig.json` / `jsconfig.json` paths.

There is no external module to import from. The components ARE the project's source code.

### Fix

ALWAYS import from the LOCAL alias :

```tsx
import { Button } from '@/components/ui/button'
import { Dialog, DialogTrigger, DialogContent } from '@/components/ui/dialog'
import { cn } from '@/lib/utils'
```

Verify `tsconfig.json` (or `tsconfig.app.json` for Vite) has :

```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["./*"]
    }
  }
}
```

For Vite projects, ALSO add the alias to `vite.config.ts` :

```ts
import path from "path"
import react from "@vitejs/plugin-react"
import { defineConfig } from "vite"

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: { "@": path.resolve(__dirname, "./") },
  },
})
```

Vite ignores `tsconfig` paths for runtime resolution; the alias MUST be duplicated in both files (issue : "two tsconfig files", verified at https://ui.shadcn.com/docs/installation/vite).

## 3. Expecting `npm update` to upgrade shadcn components

### Symptom

A developer wants to pick up a recent improvement to the `Button` component. They run :

```bash
npm update
# or
pnpm update
```

Nothing changes. The local `components/ui/button.tsx` is byte-identical to yesterday. The developer concludes "shadcn is broken".

### Root cause

`npm update` upgrades versioned packages declared in `package.json`. The shadcn components are NOT versioned dependencies; they are application source code. They have no version in `package.json`. They live in `components/ui/*.tsx` as the consumer's own files.

The closest analogue : `npm update` will upgrade the underlying primitives (`radix-ui`, `class-variance-authority`, `tailwind-merge`, `lucide-react`) according to their semver ranges. But the shadcn-authored wrapper code that USES those primitives is frozen at the time of `shadcn add`.

### Fix

ALWAYS upgrade components by re-running the CLI :

```bash
# Preview what would change
pnpm dlx shadcn@latest add button --diff

# Apply (overwriting local edits ; use git to recover them if needed)
pnpm dlx shadcn@latest add button --overwrite
```

For multiple components in one pass :

```bash
pnpm dlx shadcn@latest add button dialog dropdown-menu form --overwrite
```

ALWAYS commit the local component files BEFORE running `--overwrite` so `git diff` shows the changes the CLI made. ALWAYS run `--diff` first if there are local customisations to preserve.

If the goal is to stay on the "current" canonical source forever, an automated script can run :

```bash
# In CI or a scheduled job
pnpm dlx shadcn@latest add --all --overwrite
# Then run tests + visual regression to detect breakage
```

NEVER expect this to be hands-off; merging local customisations is unavoidable when the canonical source changes.

## 4. Running `shadcn add <component> --overwrite` without `--diff` first

### Symptom

A developer customised `components/ui/button.tsx` to add a `loading` variant. They re-run :

```bash
pnpm dlx shadcn@latest add button --overwrite
```

The local file is replaced. The `loading` variant is GONE. The developer's customisation is lost. If the file was not committed, the work is irrecoverable.

### Root cause

`--overwrite` is destructive by design : it tells the CLI "I know there is a local file; replace it with the canonical source". It does NOT merge. It does NOT preserve customisations. It is a one-shot copy.

The CLI does not warn loudly about customisations because it cannot know what is "the consumer's edit" versus "a stylistic choice from a previous shadcn version".

### Fix

ALWAYS run `--diff` BEFORE `--overwrite` when local customisations may exist :

```bash
pnpm dlx shadcn@latest add button --diff
```

Read the output carefully. Categorise each change :
- A : Canonical fix or improvement worth taking.
- B : A breaking change in the API of the underlying primitive.
- C : A conflict with a local customisation.

For each category :
- Category A : ALWAYS apply.
- Category B : Review the upgrade path; sometimes a primitive upgrade requires non-trivial migration.
- Category C : ALWAYS choose : skip the overwrite entirely, OR overwrite then manually re-apply the local customisation from `git diff`.

ALWAYS commit local component files BEFORE running `--overwrite`. The shadcn ownership doctrine treats those files as application source code; they belong in version control.

For a one-shot recovery from an accidental overwrite that destroyed uncommitted work :

```bash
# If the file was tracked but not committed since the customisation :
git checkout HEAD -- components/ui/button.tsx

# If the file was untracked at the time of customisation :
# The work is lost. Re-create from memory / a tab in the editor / a previous backup.
```

See [shadcn-errors-cli-sync-mismatch](../../../shadcn-errors/shadcn-errors-cli-sync-mismatch/SKILL.md) for the full recovery playbook including diff-and-merge workflows for teams.

## Why the four anti-patterns matter together

These four anti-patterns share one root cause : **mistaking shadcn for a traditional component library**. Every fix points the developer back at the ownership doctrine :

1. There is no runtime package to install.
2. There is no module path to import from.
3. There is no semver-driven upgrade.
4. There is no automatic merge on re-install.

ALWAYS treat shadcn components as your source code, never as a vendored dependency. NEVER apply library-style mental models : they will produce one of the above failures every time.

## Sources

- https://ui.shadcn.com/docs (5 pillars, ownership doctrine)
- https://ui.shadcn.com/docs/cli (CLI flag semantics)
- https://ui.shadcn.com/docs/installation/vite (alias-in-two-files note)
- https://ui.shadcn.com/docs/components-json (schema)
- https://github.com/shadcn-ui/ui (CLI source)

Verified 2026-05-19.
