# shadcn-core-blocks : Anti-Patterns

Five canonical anti-patterns that produce real, recurring failures with shadcn blocks. Each entry documents WHY the anti-pattern fails and the deterministic fix. Verified against `SOURCES.md` primary URLs on 2026-05-19.

## Anti-Pattern 1 : Treating blocks as immutable starter templates

### Symptom

The team installs `dashboard-01`, sees a complete page, and treats the copied files as a sealed template. They build a wrapper component (`DashboardWrapper`) that imports the block page and tries to override sections via props or context. Customisation requirements pile up into a multi-layer indirection of wrappers, slots, and conditionals.

### Why this fails

1. **Defeats the Open Code pillar.** The official shadcn doctrine (verified verbatim at https://ui.shadcn.com/docs) is : "The top layer of your component code is open for modification." A block is owned code from the moment `shadcn add` finishes ; treating it as immutable inverts the design intent.
2. **Wrapper indirection costs more than direct editing.** Each wrapper layer adds prop-drilling, conditional logic, and a place where a future engineer must understand "what is this wrapper actually doing". Editing the block file directly is a single change.
3. **AI assistants lose effectiveness.** The AI-Ready pillar is predicated on Claude reading the local source. When the source is hidden behind a wrapper that conditionally renders sections, Claude cannot reason over the actual UI surface.
4. **Type safety degrades.** Wrappers that expose every block section as an optional prop produce TypeScript signatures with twenty optional properties, no two of which are ever set together.

### Fix

ALWAYS edit the block files in place. Delete unused sub-components. Rename props that read awkwardly in your domain. Replace data sources with your own server queries. The block IS your code.

NEVER build a `BlockWrapper` component that delegates to the original. If you find yourself writing one, stop and edit the block file instead.

## Anti-Pattern 2 : Running `shadcn add <block-id> --overwrite` on a customised block

### Symptom

Six months after installing `dashboard-01` and customising it extensively, the team wants the latest shadcn block updates. They run :

```bash
pnpm dlx shadcn@latest add dashboard-01 --overwrite
```

All customisations vanish. The files are restored to the canonical upstream version. Hours or weeks of work are lost.

### Why this fails

1. **`--overwrite` is binary and silent.** The flag writes the upstream version to every file the block touches. There is no per-file confirmation, no merge, no preview. The CLI has done exactly what was requested.
2. **The block is a multi-file scaffold.** A component overwrite touches one file ; a block overwrite touches many. The blast radius is correspondingly larger.
3. **Underlying primitive auto-install can compound the damage.** If the block depends on `sidebar` and `data-table`, those primitives may also be overwritten. Customisations to `components/ui/sidebar.tsx` disappear too.
4. **Git history is the only recovery path.** If the customisations were never committed (e.g. on a feature branch waiting for review), recovery is impossible.

### Fix

ALWAYS preview the upstream changes first :

```bash
pnpm dlx shadcn@latest add dashboard-01 --diff
```

Decide per file :
- Sub-component never touched locally → `--overwrite --path <file>` is safe.
- Sub-component customised locally → SKIP the overwrite ; apply the upstream diff manually using `git apply` or hand-edit.

ALWAYS commit the customisations to git BEFORE any rerun, so recovery via `git reset` is always available. NEVER trust `--overwrite` without a clean working tree.

## Anti-Pattern 3 : Mixing multiple `style` enum values in one project

### Symptom

The team initialised the project with `default` years ago. They install a new dashboard block today and decide they prefer `new-york`. They edit `components.json` :

```json
{ "style": "new-york" }
```

They then run `pnpm dlx shadcn@latest add dashboard-01`. The new block uses `new-york` defaults (tight radii, geometric character). The pre-existing buttons, dialogs, and cards still use `default` (softer radii, original character). The visual language fragments. Spacing, radii, and typography no longer compose into a coherent system.

### Why this fails

1. **`style` controls baked-in classes, not just future installs.** Each `shadcn add` reads the current `style` value and copies the components OR blocks with that style's classes and variants baked into the file. Editing `style` post-init does NOT retroactively re-style already-installed code ; it only changes what FUTURE installs produce.
2. **Radii, spacing, and typography differ per style.** `default` uses a softer radius and looser layout. `new-york` uses tighter radii and a more geometric layout. `sera` uses underline controls and uppercase headings. `luma` uses softer ambient gradients. Mixing them produces a UI where buttons look like one design system and cards look like another.
3. **No automatic migration exists.** The CLI provides no `migrate style` command. The documentation does not list one. (Verified at https://ui.shadcn.com/docs/cli, 2026-05-19.)
4. **Block dependencies amplify the mismatch.** A `dashboard-01` block under `new-york` depends on `sidebar`, `data-table`, `card`, etc. If those primitives were installed under `default` and remain in place, the dashboard inherits a half-`new-york`, half-`default` look.

### Fix

ALWAYS choose ONE style at `init` time and keep it. NEVER edit `components.json#style` casually.

If a style migration is genuinely required, do it as a project-wide refactor (see `references/examples.md`, Example 4) :

1. Snapshot the inventory of installed components and blocks.
2. Delete `components.json` and `components/ui/`.
3. Re-init with the new style.
4. Re-add every previously-installed component and block.
5. Port customisations forward from git history.
6. Run a full visual review.

NEVER attempt a partial migration ; it always produces fragmentation.

## Anti-Pattern 4 : Importing a block sub-component from a non-local path

### Symptom

After installing `sidebar-07` (which produces `components/app-sidebar.tsx` and several sub-components), a developer writes :

```tsx
// WRONG : invented package path
import { AppSidebar } from "shadcn-ui/blocks/sidebar-07/app-sidebar"

// WRONG : invented scoped package
import { AppSidebar } from "@shadcn/blocks"

// WRONG : copy-pasted from outdated tutorial
import { AppSidebar } from "@/components/ui/app-sidebar"   // wrong subfolder
```

The build fails with `Module not found` errors, or the import resolves to nothing and the page renders blank.

### Why this fails

1. **There is no runtime shadcn package.** As documented in [shadcn-core-architecture](../../shadcn-core-architecture/SKILL.md), no `shadcn-ui`, `@shadcn/ui`, or `@shadcn/blocks` runtime npm package exists. The shadcn CLI is invoked via `dlx`/`npx` at install time and copies SOURCE into the consumer project. There is nothing to import from.
2. **Block sub-components are local files at known paths.** When `shadcn add sidebar-07` copies `app-sidebar.tsx`, it places the file at the path determined by `components.json` aliases. With default aliases that is `@/components/app-sidebar` (NOT `@/components/ui/app-sidebar` ; the `ui` subfolder is reserved for primitives like `button`, `card`, the `sidebar` PRIMITIVE itself).
3. **AI training data is often outdated.** LLMs frequently invent imports from `shadcn-ui` or `@shadcn/ui` because old tutorials and AI-generated blog posts cited those paths. Verifying against the actual local file tree is the only reliable check.
4. **The same block-id can resolve to different paths in different projects.** A project that customised `components.json` aliases (e.g. `"components": "@/src/components"`) will have the block sub-components at `@/src/components/app-sidebar`, not `@/components/app-sidebar`.

### Fix

ALWAYS run `ls` or use the editor's file tree to confirm WHERE the CLI actually wrote the block files, then import from that exact local path :

```bash
# verify the actual file location
ls components/
# components/app-sidebar.tsx
# components/nav-projects.tsx
# components/nav-secondary.tsx
# components/nav-user.tsx
# components/ui/sidebar.tsx                       <-- primitive, different file
```

```tsx
// CORRECT : import from the actual local alias path
import { AppSidebar } from "@/components/app-sidebar"
import { SidebarProvider } from "@/components/ui/sidebar"
```

NEVER import from a package path that does not exist. If a path-resolution fails, the FIRST debug step is to verify the file's actual location on disk.

## Anti-Pattern 5 : Expecting blocks to auto-update when shadcn ships new revisions

### Symptom

The team installed `dashboard-01` in January, customised it, committed it, and moved on. In June, they read that shadcn shipped an improved version of `dashboard-01` with a new chart variant. They run `pnpm update` (or `pnpm dlx shadcn@latest`) and expect the dashboard to pick up the new chart. Nothing changes.

### Why this fails

1. **There is no library to update.** The `pnpm update` command updates npm packages. The shadcn CLI is one such package, but its update only refreshes the CLI binary, not the components or blocks already copied into the project. The components and blocks ARE the project's source code, not runtime dependencies.
2. **Same ownership doctrine as components.** As documented in [shadcn-core-architecture](../../shadcn-core-architecture/SKILL.md), shadcn explicitly inverts the library model. The maintainer publishes recipes ; the consumer owns the kitchen. Upgrades are explicit, manual, and per-file.
3. **There is no semver-driven rollout for blocks.** A new block revision does not produce a semver bump that triggers an `npm update` cascade. The block is updated only when the consumer chooses to re-run `shadcn add <block-id>` with `--overwrite` or `--diff` and a manual merge.
4. **Block dependencies on underlying components compound the issue.** A "new" block revision may also depend on new component revisions ; re-running `add` for the block may also touch the underlying primitives.

### Fix

ALWAYS treat upgrades as explicit, manual operations :

```bash
# 1. preview the new upstream version
pnpm dlx shadcn@latest add dashboard-01 --diff

# 2. compare against the local customised version
git diff <commit-when-block-was-added> -- app/dashboard/ components/dashboard/

# 3. decide per file and per change : overwrite, skip, or hand-merge

# 4. commit the merge separately from any unrelated work
git add app/dashboard/ components/dashboard/
git commit -m "feat(dashboard): merge upstream dashboard-01 chart-variant"
```

ALWAYS pin the working state of block files in git immediately after `add`. ALWAYS verify https://ui.shadcn.com/docs/changelog for breaking changes in underlying primitives BEFORE re-adding a block. NEVER expect block updates to flow in via `npm update` ; the ownership doctrine is non-negotiable.

## Sources

- https://ui.shadcn.com/docs (Open Code, AI-Ready pillars, "not a component library" framing)
- https://ui.shadcn.com/docs/cli (`--diff`, `--overwrite`, `--path` semantics)
- https://ui.shadcn.com/docs/components-json (`style` field, immutability)
- https://ui.shadcn.com/docs/changelog (style additions, breaking-change history)
- https://ui.shadcn.com/blocks (block gallery and file inventory)
- https://github.com/shadcn-ui/ui/issues (real-world recurrence of overwrite, style-mix, import-path failures)

Verified 2026-05-19.
