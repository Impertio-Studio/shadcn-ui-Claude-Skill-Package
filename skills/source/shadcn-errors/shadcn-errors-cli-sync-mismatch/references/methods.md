# shadcn CLI Sync Mismatch : Methods and Signatures

All commands verified at https://ui.shadcn.com/docs/cli on 2026-05-19
against shadcn CLI evergreen (shadcn@4.7.0).

## `shadcn add` : Full Flag Reference (relevant to sync)

Signature :

```
pnpm dlx shadcn@latest add [components...] [flags]
```

`components...` accepts component names (`button`), block names
(`login-01`), preset codes (`a2r6bw`), full registry URLs, and the
sentinel `--all`.

| Flag | Short | Default | Behavior relevant to sync |
|------|-------|---------|----------------------------|
| `--overwrite` | `-o` | false | Replace existing files WITHOUT prompting. Destructive. |
| `--yes` | `-y` | false | Skip confirmation prompts. Combined with `--overwrite` makes a destructive run fully silent. |
| `--dry-run` | | false | Print planned operations. Writes NOTHING. Returns 0 on success. |
| `--diff [path]` | | (off) | Print a unified diff of the registry version against the on-disk file. Writes NOTHING. Accepts an optional file path to diff a specific local file. |
| `--all` | `-a` | false | Add every component in the registry. Rarely correct ; never combine with `--overwrite` without `--dry-run` first. |
| `--path` | `-p` | from components.json `aliases.ui` | Write files to a different directory. Useful for fork pattern. |
| `--view` | | (off) | Print the registry item JSON, then exit. No file writes. |

Verbatim source quotes (https://ui.shadcn.com/docs/cli, 2026-05-19) :

- `-o, --overwrite` : "overwrite existing files. (default: false)"
- `-y, --yes` : "skip confirmation prompt. (default: false)"
- `--dry-run` : "preview changes without writing files. (default: false)"
- `--diff [path]` : "show diff for a file."

There is NO `shadcn diff` subcommand. There is NO `shadcn update`
subcommand. The diff capability is the `--diff` FLAG on `add`.

## Recommended Sync Workflow (deterministic)

```bash
# 1. Clean working tree.
git status                                           # must be clean
git stash push -m "pre-shadcn-sync" || true          # stash if needed

# 2. New branch.
git switch -c chore/shadcn-sync-$(date +%Y%m%d)

# 3. Probe : list what would change, no writes.
pnpm dlx shadcn@latest add <name> --dry-run

# 4. Inspect the actual diff.
pnpm dlx shadcn@latest add <name> --diff

# 5. Decide :
#    a) Diff is acceptable AND extensions are in *-extensions.tsx :
pnpm dlx shadcn@latest add <name> --overwrite

#    b) Diff conflicts with inline edits : abort. Refactor first.
#       Move custom variants to <name>-extensions.tsx, commit, retry.

# 6. Review CLI write.
git diff HEAD -- components/ui/<name>.tsx

# 7. Run tests, then commit.
pnpm test
git add components/ui/<name>.tsx
git commit -m "chore: sync <name> with shadcn@latest"
```

The `--dry-run` step is cheap and catches one common surprise : `add
<name>` may also install or update SIBLING components (e.g. `add
dialog` may also touch `button` because the registry lists it as a
`registryDependency`). `--dry-run` enumerates every file the run
would touch.

## Custom-Variants-Extension Pattern : Method Signature

The pattern : every custom variant or composition lives in
`components/ui/<name>-extensions.tsx`. The base file
`components/ui/<name>.tsx` stays byte-identical to the registry version.

Minimal extension file template :

```ts
// components/ui/<name>-extensions.tsx
import { cva, type VariantProps } from "class-variance-authority"
import {
  <Name> as Base<Name>,
  <name>Variants as base<Name>Variants,
} from "@/components/ui/<name>"
import { cn } from "@/lib/utils"
import * as React from "react"

const ext<Name>Variants = cva("", {
  variants: {
    // Your additional variants here.
  },
})

type Ext<Name>Props = React.ComponentProps<typeof Base<Name>> &
  VariantProps<typeof ext<Name>Variants>

const Ext<Name> = React.forwardRef<HTMLButtonElement, Ext<Name>Props>(
  ({ className, ...extProps }, ref) => {
    return (
      <Base<Name>
        ref={ref}
        className={cn(ext<Name>Variants(extProps), className)}
        {...extProps}
      />
    )
  }
)
Ext<Name>.displayName = "Ext<Name>"

export { Ext<Name>, ext<Name>Variants }
```

Rules for the extension file :

- ALWAYS import the base component from `@/components/ui/<name>`.
  NEVER copy-paste the base component into the extension file.
- ALWAYS merge the base class string with `cn(base, ext, className)`.
  NEVER concatenate with `+` ; that breaks `tailwind-merge` ordering.
- ALWAYS expose your extension under a distinct name (`ExtButton`,
  `WarningButton`, `BrandedCard`). NEVER re-export under the same name
  as the base ; that hides which import is which.

## `shadcn migrate` : Sync-Relevant Subcommands

Signature :

```
pnpm dlx shadcn@latest migrate <subcommand> [flags]
```

Available subcommands (verified 2026-05-19) :

| Subcommand | Effect | Flags |
|------------|--------|-------|
| `icons` | Replace icon library imports in every `components/ui/` file | `-l/--list`, `-y` |
| `radix` | Move `@radix-ui/react-*` per-primitive imports to unified `radix-ui` | `-y` |
| `rtl` | Add right-to-left attributes and helpers | `-y` |

Common flags :

- `-l, --list` : list available migrations and exit (only on `migrate
  icons`, lists available icon libraries).
- `-y, --yes` : skip per-file confirmation. Use ONLY after a clean
  preview via git or a no-op verification.

Behavior expectations :

- `migrate icons` rewrites import lines and JSX `<IconName />` usages
  to the target library's symbols. Custom JSX you wrote that uses the
  old icon library WILL also be rewritten (which is usually correct).
- `migrate radix` only rewrites `import` lines, not call sites
  (`<Primitive.Root>` shapes are identical between the per-primitive
  packages and the unified package).
- `migrate rtl` adds `dir`-aware utility classes ; safe but
  highly diff-noisy.

## Git Hygiene : Pre-Overwrite Snapshot

Before any `add --overwrite` or `migrate`, capture an explicit
snapshot commit so you can `git revert` cleanly :

```bash
git add -A
git commit -m "snapshot: pre-shadcn-sync of $(git rev-parse --short HEAD)"
git tag pre-shadcn-sync-$(date +%Y%m%dT%H%M%S)
```

This makes the rollback trivial :

```bash
git reset --hard pre-shadcn-sync-<timestamp>
```

The tag is local-only by default ; push it with `git push --tags` only
if your team conventionally tracks sync points.

## Detection : Which Component Files the CLI Wrote vs Customized

```bash
# List likely shadcn-managed files.
git ls-files 'components/ui/*.tsx'

# Per-file commit count (single commit = likely vendored / not touched).
git log --oneline -- components/ui/button.tsx | wc -l

# Per-file authorship spread.
git shortlog -sn -- components/ui/button.tsx
```

Heuristics :

- 1 commit total -> file likely never edited beyond `add`.
- Commits authored by mixed people, multiple over time -> customized.
- Last commit message starts with `chore: re-add` or `chore: sync
  shadcn` -> the file has been re-CLI-written ; check if any
  customization was intentionally re-applied after.

## Version Pin Reality : `components.json` Has No Per-Component Lock

`components.json` schema fields (verified at
https://ui.shadcn.com/docs/components-json, 2026-05-19) :

```json
{
  "$schema": "...",
  "style": "new-york",
  "rsc": true,
  "tsx": true,
  "tailwind": { ... },
  "aliases": { ... },
  "iconLibrary": "lucide",
  "registries": { ... }
}
```

There is NO `installedVersion`, NO `componentLocks`, NO `dependencies`
section listing per-component versions. The CLI does NOT write a
lockfile to track what version of `button.tsx` you currently have.
Treat the on-disk file content as the ONLY source of truth, and use
git to version it explicitly.

The closest practical "pin" is :

1. Commit your `components/ui/<name>.tsx` to git after every CLI run.
2. Add a header comment in the file recording the date and shadcn CLI
   version used.
3. NEVER run `add --all`. Always name components explicitly so you
   know exactly what will change.
