# Anti-Patterns : Installation and Customisation Mistakes

Each entry : symptom, root cause, fix, prevention.

## 1. Running `add` Before `init` (components.json Missing)

### Symptom

```
$ pnpm dlx shadcn@latest add button
✖ Cannot find components.json at /home/me/my-app
```

Or worse : the CLI proceeds with prompts but the project ends up half-
configured. Tailwind tokens are missing, `cn()` does not exist at
`@/lib/utils`, and the newly written Button file ships with broken
imports.

### Root cause

`add` reads `components.json` to know aliases, base library, style, and
icon set. Without it, alias resolution fails silently OR the CLI prompts
for every field every time you add a component. The expected entry point
is ALWAYS `init` first.

### Fix

```bash
# Run init properly :
pnpm dlx shadcn@latest init

# THEN add components :
pnpm dlx shadcn@latest add button
```

If you have already added components into a project without `init`,
re-run init with `-f` to back-fill components.json, then `add --overwrite`
every component to fix the broken imports :

```bash
pnpm dlx shadcn@latest init -f
pnpm dlx shadcn@latest add button --overwrite
```

### Prevention

ALWAYS verify `components.json` exists at the project root before any
`add` invocation. The first action in any new shadcn project is `init`,
not `add`. Treat `components.json` as a project-level prerequisite the
same way `tsconfig.json` and `package.json` are.

## 2. Running `add --overwrite` Without `--diff` First

### Symptom

You re-run `pnpm dlx shadcn@latest add button --overwrite` to pull a bug
fix from upstream. Your custom `brand` variant, your custom `xl` size,
and your project-specific Tailwind tokens are silently gone. The git
history is the only record of what was lost.

### Root cause

`--overwrite` replaces the file wholesale. There is no merge step, no
prompt, no backup file. The CLI fetches the registry's current shape
and writes it on top of yours.

### Fix

Recovery is git-driven :

```bash
# 1. Find the commit before the overwrite.
git log --oneline components/ui/button.tsx

# 2. Diff the lost edits against the post-overwrite shape.
git diff <pre-overwrite-sha>..HEAD -- components/ui/button.tsx

# 3. Hand-port the custom variants onto the new shape.
# 4. Commit.
```

If you ran `--overwrite` without committing first, your edits are gone.
There is no `shadcn undo`.

### Prevention

ALWAYS run the diff-then-overwrite pair, never overwrite blind :

```bash
# Step 1 : commit current state.
git add . && git commit -m "Snapshot before shadcn sync"

# Step 2 : preview.
pnpm dlx shadcn@latest add button --diff

# Step 3 : decide. If accepting, overwrite.
pnpm dlx shadcn@latest add button --overwrite

# Step 4 : re-apply custom changes from the snapshot commit.
```

ALWAYS commit BEFORE `--overwrite`. The snapshot commit IS the only
recovery mechanism.

## 3. Alias Mismatch Between tsconfig.json and components.json

### Symptom

```
$ pnpm dlx shadcn@latest add button
# (writes components/ui/button.tsx)

$ pnpm dev
Error: Cannot find module '@/lib/utils' from
  '/home/me/my-app/components/ui/button.tsx'
```

OR : the dev server starts but the editor (VS Code) shows red squiggles
under every `@/...` import. OR : on a Vite project, the editor is fine
but `pnpm build` fails on every import.

### Root cause

shadcn rewrites imports inside the file based on `components.json#aliases`,
but the actual import resolver (TypeScript / Vite / Next.js / Astro) uses
its own config. When `components.json#aliases.utils` is `@/lib/utils` but
`tsconfig.json#compilerOptions.paths."@/*"` points at `["./packages/ui/*"]`
(or is missing entirely), the rewrite produces a path the resolver cannot
follow.

Vite specifically requires the alias in THREE places : `tsconfig.json`,
`tsconfig.app.json`, AND `vite.config.ts`. Missing one of them is the most
common Vite-specific variant of this bug.

### Fix

1. Open `components.json` and note `aliases.utils`, `aliases.components`,
   `aliases.ui`, `aliases.lib`, `aliases.hooks`.
2. Open `tsconfig.json` and verify the prefix in `compilerOptions.paths`
   matches. Example : if `aliases.utils` is `@/lib/utils`, then
   `paths."@/*"` must point to wherever `lib/utils.ts` actually lives.
3. For Vite : repeat in `tsconfig.app.json` AND `vite.config.ts`.
4. Re-run `add --overwrite` on every affected component file to refresh
   the rewritten imports.

### Prevention

ALWAYS run `pnpm dlx shadcn@latest info` after init and after any tsconfig
change. `info` dumps the resolved config the CLI sees ; if it disagrees
with what you expect, the mismatch is visible.

NEVER hand-edit `aliases` in components.json without also updating tsconfig
(and vite.config.ts for Vite). The two MUST stay in lockstep. If you must
change one, change both in the same commit.

NEVER mix tsconfig paths AND package.json#imports for the same project.
The resolution order is implementation-defined and silent.

## 4. Hand-Installing `@radix-ui/react-*` Packages Without the CLI

### Symptom

A developer reads the Dialog source code at `components/ui/dialog.tsx`,
sees it imports from `@radix-ui/react-dialog`, and decides to install
that package by hand : `pnpm add @radix-ui/react-dialog`. Then they
write or copy the Dialog file by hand from the docs site. The
installation appears to work but :

- Versions drift from what the registry has verified together
- Other shadcn components that depend on Dialog (Sheet, AlertDialog,
  Drawer's responsive variant) end up with different Radix peer-dep
  versions
- The Feb 2026 migration to unified `radix-ui` cannot run cleanly because
  the project has BOTH `radix-ui` and stale `@radix-ui/react-*` packages
- ESLint rules tied to component-set assumptions throw warnings

### Root cause

The CLI knows the exact dependency set per registry item and pins versions
that are tested together. Hand-installing `@radix-ui/react-*` skips this
guarantee. The same applies to `cmdk`, `vaul`, `react-day-picker`,
`input-otp`, `sonner`, `@tanstack/react-table`, `embla-carousel-react`,
`react-resizable-panels`.

### Fix

Remove the hand-installed packages and rerun `add` :

```bash
pnpm rm @radix-ui/react-dialog @radix-ui/react-slot @radix-ui/react-portal
# (or whatever you hand-installed)

pnpm dlx shadcn@latest add dialog --overwrite
# CLI installs the right Radix versions automatically.
```

### Prevention

ALWAYS let the CLI install the runtime deps. NEVER hand-install
`@radix-ui/react-*`, `cmdk`, `vaul`, `react-day-picker`, `input-otp`,
`sonner`, `@tanstack/react-table`, `embla-carousel-react`,
`react-resizable-panels`, `class-variance-authority`, `tailwind-merge`,
`clsx`, or `lucide-react` outside the CLI. The only sanctioned manual
install is when YOU are publishing a component to a registry and need
the deps in your dev tree to author it.

For migrating away from individual `@radix-ui/react-*` packages, use
`pnpm dlx shadcn@latest migrate radix` (introduced Feb 2026, see
`shadcn-core-cli`).

## 5. Never Re-Running the CLI : Components Stuck Six Months Behind

### Symptom

A project initialised in late 2025 has never run `shadcn add --diff`
since. Button is on the pre-`cursor-pointer` version, Sidebar is missing
the v2 `useSidebar` hook, Calendar still wraps `react-day-picker` v8 even
though everyone else moved to v9, and a console warning about a missing
`DialogTitle` on mobile sidebars has been ignored for months.

### Root cause

The "components are owned code" doctrine is true but is OFTEN misread as
"never touch them again". The correct reading is : you own the right to
diverge, you do not have an obligation to. Upstream ships bug fixes,
accessibility improvements, new variants, performance tweaks, and
breaking-change accommodations (e.g., Tailwind v3 -> v4). Refusing to
sync means accepting indefinite drift.

### Fix

Treat shadcn sync as a quarterly maintenance task :

```bash
# Quarterly maintenance script (run every ~3 months) :

# 1. Snapshot commit.
git add . && git commit -m "Snapshot before quarterly shadcn sync"

# 2. Wide diff.
pnpm dlx shadcn@latest add -a --diff > /tmp/shadcn-diff-$(date +%F).txt

# 3. Review the diff file. Per component : accept / reject / customise.
# 4. Accept :
pnpm dlx shadcn@latest add button card dialog form input label --overwrite

# 5. Reapply local customisations using git diff against the snapshot.
git diff HEAD~1 -- components/ui/

# 6. Commit the merged result.
git add . && git commit -m "Quarterly shadcn sync : v4.7.0 baseline"
```

### Prevention

ALWAYS schedule a recurring shadcn-sync task (quarterly or after every
major shadcn CLI release). The shadcn CLI semver-versions itself ; pin
yourself to `pnpm dlx shadcn@latest` so the next sync uses the latest
registry shape.

ALWAYS subscribe to https://ui.shadcn.com/docs/changelog so you know when
non-trivial component changes ship (cursor-pointer flag, sera/luma styles,
package.json#imports support, radix unified package). When those land,
sync within the next two weeks rather than waiting for a quarterly.

NEVER assume "we copied it once, we are done". The ownership model means
you own the right to merge upstream, not the obligation to drift forever.

## Verified Sources

- https://ui.shadcn.com/docs/cli (verified 2026-05-19)
- https://ui.shadcn.com/docs/installation/vite (verified 2026-05-19)
- https://ui.shadcn.com/docs/changelog (verified 2026-05-19)
- https://ui.shadcn.com/docs/registry (verified 2026-05-19)
- https://github.com/shadcn-ui/ui/issues (anti-patterns cross-referenced ;
  verified 2026-05-19)
