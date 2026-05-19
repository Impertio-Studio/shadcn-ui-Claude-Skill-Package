---
name: shadcn-errors-cli-sync-mismatch
description: >
  Use when re-running `shadcn add <component>` would overwrite a file in
  `components/ui/` that you have edited locally, when a teammate reports
  that "the CLI wiped my changes", when deciding whether to update a shadcn
  component to pick up upstream fixes (Calendar v9, cmdk break, security
  patch) without losing custom variants, or when you need a deterministic
  strategy to keep custom code alive across `--overwrite` runs. Prevents
  the silent-destruction trap where `pnpm dlx shadcn@latest add button
  --overwrite` blasts local edits with zero confirmation, the
  miss-the-upstream-fix trap where teams freeze a component and stop
  receiving critical patches, and the inline-custom-variants trap where
  edits embedded directly in `components/ui/<name>.tsx` are unrecoverable
  after a re-add. Covers the diff-first workflow with `add --diff`, the
  three resolution strategies (vendor / diff-merge / fork), the
  custom-variants-in-a-separate-file pattern that survives any re-add,
  pre-update git hygiene, the `migrate icons` and `migrate radix` flows,
  and why `components.json` does NOT pin per-component versions.
  Keywords: shadcn cli overwrite, CLI wiped my changes, shadcn diff,
  shadcn add --diff, shadcn add --overwrite, shadcn add --dry-run,
  custom variants lost, vendor pattern, fork shadcn component, diff
  merge, how do I preserve customizations, shadcn add --overwrite
  danger, never update shadcn, components.json version pin, evergreen
  components, shadcn migrate icons, shadcn migrate radix, lucide to
  radix icons, @radix-ui to radix-ui, button-extensions, separate
  variants file, my customizations are gone, lost local edits, re-add
  component, update shadcn component, shadcn update workflow, pin
  shadcn component version, snapshot shadcn, fork shadcn ui, copy
  shadcn to lib, security fix shadcn, react-day-picker v9 upgrade,
  cmdk break recovery, git status before shadcn add, clean working
  tree shadcn, shadcn cli destructive, preserve customizations
  shadcn, shadcn no version lock.
license: MIT
compatibility: "Designed for Claude Code. Requires shadcn ui evergreen-2026 (CLI shadcn@4.7.0)."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# shadcn CLI Sync Mismatch : Recovering and Preventing Overwrite of Local Edits

shadcn copies component source into your repo. Once copied, the file
in `components/ui/<name>.tsx` is YOUR code. The CLI does NOT remember
that you edited it. Re-running `shadcn add <name> --overwrite` (or
running `add <name>` with `-y` and accepting the prompt) replaces the
file byte-for-byte with the current registry version. Local edits are
gone. There is no undo inside the CLI ; recovery is git-only.

This skill defines the diff-first workflow that prevents that loss,
three resolution strategies for picking up upstream changes, and the
custom-variants-in-a-separate-file pattern that makes re-adds safe.

Companion skills :

- `shadcn-core-cli` : full reference for every CLI command and flag
- `shadcn-impl-component-install` : initial component installation flow
- `shadcn-errors-styling-conflicts` : when re-adds expose Tailwind
  class-order conflicts in your customizations

## Quick Reference : The Problem in One Paragraph

`components.json` does NOT record which version of each component you
have. The CLI is evergreen : every `add` fetches the CURRENT registry
item. There is no per-component lockfile, no `components.lock.json`,
no `installedVersion` field. The single source of truth for "what
version do I have" is the file content on disk. If you edited that
file and lost the prior commit, the only recovery is to re-add (which
discards your edits) or restore from git history. ALWAYS treat every
file in `components/ui/` as code you own and version-control like any
other source.

## Quick Reference : The Diff-First Workflow (ALWAYS)

NEVER run `add <name> --overwrite` blind. ALWAYS preview with `--diff`
first. The canonical sequence is :

1. `git status` MUST be clean (or all `components/ui/` changes staged
   and committed). NO uncommitted edits to the target file.
2. `pnpm dlx shadcn@latest add <name> --diff` to preview the upstream
   diff against your current file.
3. Decide on a resolution strategy (see next section).
4. If applying : `pnpm dlx shadcn@latest add <name> --overwrite`.
5. `git diff HEAD` to review what the CLI changed.
6. Re-apply any necessary customizations (or rely on
   custom-variants-in-separate-file).
7. Commit with a message that names the registry version source
   (e.g. "chore: re-add Calendar from shadcn@latest for v9 fix").

If `git status` is dirty when you start, STOP. Commit or stash first.
A failed merge plus a dirty tree means you cannot tell which lines are
"your edit" and which are "shadcn just wrote that".

`--dry-run` is a safer-than-`--diff` first probe : it prints the
planned operations without writing anything AND without producing a
patch. Use `--dry-run` to confirm what files will be touched, then
`--diff` to see the actual content delta.

## Quick Reference : The `--diff` Flag, NOT a `diff` Subcommand

There is no `shadcn diff` subcommand. The diff capability ships as a
FLAG on the `add` command. Verified at
https://ui.shadcn.com/docs/cli (2026-05-19) :

- `pnpm dlx shadcn@latest add button --diff` is correct.
- `pnpm dlx shadcn@latest diff button` is WRONG. It throws "unknown
  command".

There is also no `shadcn update` subcommand. The update workflow IS
the diff + overwrite pair above.

## Quick Reference : The Three Resolution Strategies

When `--diff` shows that upstream has changed, you choose ONE strategy
per component. The choice is per-component, not per-project.

| Strategy | When to use | Cost | Risk |
|----------|-------------|------|------|
| Vendor (snapshot) | Component is stable, your edits are heavy, no upstream fixes needed | Zero | Miss security and bugfixes forever |
| Diff-merge | Some local edits + upstream has fixes worth pulling | Manual merge per update | Merge mistakes if edits and upstream lines collide |
| Fork (copy out) | Heavy customization AND you still want to track upstream | Two paths to maintain | Drift from registry conventions |

Most projects use a mix : Button and Card are vendored (rarely
change), Calendar and Command are diff-merged (cmdk / react-day-picker
break often), a heavily-extended Sidebar is forked to
`lib/components/sidebar/`.

### Strategy 1 : Vendor (treat add as a one-time snapshot)

ALWAYS run `shadcn add <name>` exactly once per component. Edit the
file freely afterwards. NEVER re-run `add <name>` for that component.
The registry version is irrelevant ; the on-disk version is canonical.

Trade-off : you stop receiving upstream bugfixes and security
patches. For low-risk components (Card, Badge, Skeleton, Separator,
Alert) this is fine. For components with active maintenance (Calendar,
Command/Combobox, Sidebar, Sheet) the risk is real.

Mark vendored components with a top-of-file comment so the next
maintainer knows :

```tsx
// VENDORED 2026-05-15 from shadcn@latest. Do NOT re-run `shadcn add card`.
// Local edits below.
```

### Strategy 2 : Diff-merge (recommended default)

You keep up with upstream AND retain customizations. Workflow :

1. Commit all local work to clean the tree.
2. `pnpm dlx shadcn@latest add <name> --diff` to see upstream changes.
3. If the diff is small (1-3 lines, no overlap with your edits) :
   apply with `--overwrite`, then re-apply your custom variant or
   prop in a follow-up commit.
4. If the diff overlaps your edits : do NOT use `--overwrite`. Instead,
   manually copy the upstream lines from the diff output and apply
   them in your editor with three-way reasoning (upstream / yours /
   merged).
5. Commit with subject "chore: sync <name> with shadcn@latest" and
   body listing both upstream changes pulled and local changes
   preserved.

Diff-merge fails fastest when you have inline custom variants. See
the "Custom Variants in a Separate File" pattern below for the fix.

### Strategy 3 : Fork (copy upstream to a different path)

You treat the registry file as an upstream reference and maintain a
forked copy at a different path that imports the same primitives.

```bash
pnpm dlx shadcn@latest add sidebar          # writes components/ui/sidebar.tsx
mkdir -p lib/components/sidebar
cp components/ui/sidebar.tsx lib/components/sidebar/sidebar.tsx
# Edit lib/components/sidebar/sidebar.tsx freely.
# Update imports : import { Sidebar } from "@/lib/components/sidebar/sidebar"
```

Keep the original `components/ui/sidebar.tsx` ONLY as a reference for
diffs against upstream ; OR delete it if your fork is permanent. To
pull upstream changes : `add sidebar --overwrite` updates the
reference copy ; then `diff lib/components/sidebar/sidebar.tsx
components/ui/sidebar.tsx` shows what to consider porting.

Trade-off : you lose the convention that `@/components/ui/<name>`
points to the canonical component. Document the fork in a top-level
README or a `lib/components/README.md`.

## Quick Reference : The Custom-Variants-in-a-Separate-File Pattern

This pattern makes `--overwrite` safe. The rule : NEVER add a new
`variant` value or `size` value to `components/ui/<name>.tsx`. ALWAYS
put extensions in a sibling file.

```ts
// components/ui/button-extensions.tsx
import { cva, type VariantProps } from "class-variance-authority"
import { Button as BaseButton, buttonVariants as baseVariants } from "@/components/ui/button"
import { cn } from "@/lib/utils"

const extendedVariants = cva("", {
  variants: {
    intent: {
      warning: "bg-amber-500 text-amber-50 hover:bg-amber-600",
      success: "bg-emerald-600 text-emerald-50 hover:bg-emerald-700",
    },
  },
})

type Props = React.ComponentProps<typeof BaseButton> &
  VariantProps<typeof extendedVariants>

export function ExtButton({ intent, className, ...props }: Props) {
  return (
    <BaseButton
      className={cn(extendedVariants({ intent }), className)}
      {...props}
    />
  )
}
```

Now `shadcn add button --overwrite` is safe : `button-extensions.tsx`
is untouched, the new `warning` and `success` intents survive.

The same pattern applies to forms (`form-extensions.tsx`), data tables
(`data-table-extensions.tsx`), and any other component you commonly
augment. See `references/examples.md` for full working code.

## Quick Reference : Pre-Update Hygiene Checklist

Before ANY `add <name> --overwrite` or `add <name>` without `-y` :

- [ ] `git status` clean OR all `components/ui/` edits committed.
- [ ] On a feature branch, not `main`. The merge should be a PR.
- [ ] `pnpm dlx shadcn@latest add <name> --dry-run` ran without error.
- [ ] `pnpm dlx shadcn@latest add <name> --diff` reviewed end-to-end.
- [ ] You can name the upstream change (bugfix? new prop? rename?).
  If you cannot, the update is not actionable yet.
- [ ] Custom variants live in a `*-extensions.tsx` file, NOT inline.
  If they are inline, refactor BEFORE the overwrite.

## Quick Reference : Detection : Which Files Are CLI-Touched vs Custom

If you inherit a project and need to know which files the CLI wrote :

```bash
# All shadcn-managed component files (under components/ui/).
git log --diff-filter=A --name-only -- 'components/ui/*' \
  | grep -E '^components/ui/[a-z-]+\.tsx$' | sort -u
```

For deeper attribution, compare each file to the current registry
version (cheap heuristic : if `git log <file>` shows a single commit
with a "chore: add <name>" message, it was likely never edited).

A custom file outside `components/ui/` (e.g. `lib/components/...`) is
unambiguously yours. A file inside `components/ui/` with multiple
commits and non-shadcn-shaped diffs has been customized.

## Quick Reference : Migration Aids

shadcn ships migration commands that perform sweeping rewrites
across all `components/ui/` files. Two are relevant here because they
explicitly touch every component file and therefore interact with the
same overwrite hazard :

- `pnpm dlx shadcn@latest migrate icons` : swaps the project icon
  library (e.g. `lucide-react` -> `@radix-ui/react-icons`). Updates
  every component's icon imports. Run with clean tree. Set the new
  default in `components.json#iconLibrary` after.
- `pnpm dlx shadcn@latest migrate radix` : moves from per-primitive
  `@radix-ui/react-*` packages to the unified `radix-ui` package
  (Feb 2026 change). Rewrites all imports.
- `pnpm dlx shadcn@latest migrate rtl` : adds right-to-left support.

Pass `-y` to skip the per-file confirmation prompt only after a
`--dry-run` review. Both `migrate icons` and `migrate radix` are
SAFE for vendored components in the sense that they only touch
imports, not body logic, BUT they still rewrite the file, so commit
first.

See `references/methods.md` for command signatures and
`references/examples.md` for full workflows.

## Decision Tree : Should I Re-Run `shadcn add <name>`?

```
Is there an upstream change you NEED (bugfix, security, breaking-deps fix)?
├── No  -> Do nothing. The current file is fine.
└── Yes -> Continue.
        Is the local file edited beyond what `add` originally wrote?
        ├── No  -> `add <name> --overwrite`. Done.
        └── Yes -> Are custom variants in a `*-extensions.tsx` sibling file?
                ├── Yes -> `add <name> --overwrite` is safe. The
                │          extensions file is untouched.
                └── No  -> STOP. Refactor inline variants into
                           `*-extensions.tsx` FIRST. Then overwrite.
                           OR use diff-merge (manual three-way merge).
                           OR fork to `lib/components/` if customization
                           is heavy enough.
```

## Patterns : What to Document Per Component

For every component in `components/ui/`, the project SHOULD record one
of these states. A single comment line at the top of the file or in
`components/ui/README.md` is enough :

- `VENDORED YYYY-MM-DD` : never re-add. Living snapshot.
- `TRACKED` : will run `add --diff` before any update.
- `FORKED-TO <path>` : original kept as upstream reference, real code
  lives elsewhere.
- `MIGRATED-AWAY` : replaced with a non-shadcn alternative.

Without this state, the next developer cannot tell whether overwriting
is safe.

## Reference Links

- Companion : `shadcn-core-cli` (full CLI reference including all `add` flags)
- Companion : `shadcn-impl-component-install` (initial install workflow)
- Companion : `shadcn-errors-styling-conflicts` (post-overwrite Tailwind class-order issues)
- Detailed methods : `references/methods.md`
- Worked examples : `references/examples.md`
- Anti-patterns to avoid : `references/anti-patterns.md`
- Official CLI docs : https://ui.shadcn.com/docs/cli
- Official components.json schema : https://ui.shadcn.com/docs/components-json
