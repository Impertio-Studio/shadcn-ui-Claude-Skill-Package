---
name: shadcn-core-blocks
description: >
  Use when scaffolding a full page (dashboard, login, signup, sidebar layout,
  calendar app), bootstrapping a starter template from shadcn's gallery,
  evaluating block vs hand-composed components, choosing between the
  default / new-york / sera / luma styles, or recovering after a `shadcn add
  <block>` rerun overwrote local customisations.
  Prevents six common block-system failures : treating blocks as immutable
  starter templates (they are owned code, edit freely), running `shadcn add
  <block>` over a customised copy without `--diff` and losing work, mixing
  multiple style enum values inside one project, importing block
  sub-components from package paths instead of the local copied location,
  expecting blocks to auto-update when shadcn ships new revisions, and
  conflating blocks with components in CLI invocation when they accept
  different mental models for ownership and customisation.
  Covers the blocks distribution surface (a separate gallery at
  /blocks distinct from /components), block category landscape (dashboard,
  sidebar, login, signup, calendar, authentication), the block-add CLI
  flow (same `shadcn add` command, multi-file output), the style enum
  values (default, new-york, sera Apr 2026, luma Mar 2026) and when to
  choose each, the post-add ownership model, and the block-versus-component
  decision logic for whole-page versus building-block scope.
  Keywords: shadcn blocks, scaffold dashboard, starter template, shadcn
  full page, what is a block, block vs component, npx shadcn add dashboard,
  sidebar-07, dashboard-01, login-03, login-04, blocks gallery,
  ui.shadcn.com/blocks, page template, layout starter, multi-file block,
  block customisation, style enum, new-york style, default style,
  sera style, luma style, typographic style, shadcn full app shell,
  scaffold an admin panel, blank dashboard, copy paste page, page section,
  pre-built layout, how do I get a dashboard, how do I add a login page,
  shadcn template, opinionated starter, free open source blocks.
license: MIT
compatibility: "Designed for Claude Code. Requires shadcn ui evergreen-2026."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# shadcn ui : Core Blocks

Blocks are shadcn's whole-page distribution surface. They are NOT a separate technology from components : they are multi-file scaffolds copied into the consumer project via the same `shadcn add` CLI, governed by the same ownership doctrine. ALWAYS treat a block as a starting point for a page, NEVER as an immutable template.

## Quick Reference

### Block in one sentence

ALWAYS describe a block as : a **multi-file composition of shadcn components, copied into the consumer project as owned source code, that delivers a complete page or large UI surface (dashboard, login page, sidebar shell)**. Components are building bricks ; blocks are pre-stacked walls.

The blocks gallery is its own surface : https://ui.shadcn.com/blocks (distinct from https://ui.shadcn.com/components). Verified verbatim from the gallery : "Clean, modern building blocks. Copy and paste into your apps. Works with all React frameworks. Open Source. Free forever."

### Four invariants

1. ALWAYS install a block with `pnpm dlx shadcn@latest add <block-id>` (or `npx`, `bunx`, `yarn dlx`). The CLI command is IDENTICAL to component install ; the block-id (e.g. `dashboard-01`, `sidebar-07`, `login-03`) signals the multi-file scaffold.
2. ALWAYS expect MULTIPLE files to be written when adding a block (page entry + sub-components + data file). NEVER assume a block is a single file like a component.
3. ALWAYS run `shadcn add <block-id> --diff` before re-running on an already-customised block. NEVER `--overwrite` blind ; the block is your code now.
4. ALWAYS pick ONE style enum value per project at `init` time and keep it. NEVER mix `new-york` and `sera` and `luma` blocks in the same project ; visual coherence breaks.

### Block versus component : decision in one line

ALWAYS choose a BLOCK when the unit of work is a whole page or major layout shell. ALWAYS choose individual COMPONENTS when composing a custom layout from scratch or extending an existing page with one new control.

## Decision Tree 1 : Block or hand-composed components?

```
Q1. Do you need a whole-page surface (login page, full dashboard, sidebar shell, calendar app)?
    no  → ALWAYS hand-compose with individual components. See shadcn-core-cli.
    yes → Q2

Q2. Does the shadcn blocks gallery offer a category that matches?
    Categories observed at /blocks (verified 2026-05-19) :
      dashboard, sidebar, login, signup, authentication, calendar
    no  → ALWAYS hand-compose ; do NOT force-fit an unrelated block
    yes → Q3

Q3. Will the team customise the layout heavily (rearrange sections, swap data sources)?
    yes → ALWAYS use the block as a starter, then edit freely (Open Code pillar)
    no  → ALWAYS use the block as-is, but still commit it as project source
```

## Decision Tree 2 : Which style enum value?

The `style` field in `components.json` selects the visual character of every component and block copied into the project. It is set at `init` time and is **immutable** after init.

```
Q1. Is this a NEW project being initialised in 2026?
    no  → ALWAYS keep the existing `style` value. NEVER edit it post-init.
    yes → Q2

Q2. What visual character does the design language need?
    typography-driven, editorial, print-design feel
                                   → ALWAYS choose `sera` (introduced Apr 2026,
                                     CLI >= 4.3.0). Underline controls, uppercase
                                     headings, typographic emphasis.
    soft, ambient, gradient-friendly, modern SaaS feel
                                   → ALWAYS choose `luma` (introduced Mar 2026,
                                     CLI >= 4.1.2). Lighter visual weight.
    geometric, design-system-grade, the modern default
                                   → ALWAYS choose `new-york`. Tight radii,
                                     opinionated defaults. Recommended for
                                     most new projects.
    legacy compatibility with pre-2024 shadcn projects only
                                   → ALWAYS choose `default` (the original
                                     style ; deprecated for new work but
                                     retained for compatibility).
```

ALWAYS install blocks AFTER `init` so they inherit the chosen style. NEVER attempt to "switch style" mid-project by editing `components.json` ; the only correct switch is a full re-init + re-add of every component and block.

## Decision Tree 3 : Adding a block to the project

```
Q1. Has `pnpm dlx shadcn@latest init` been run (does `components.json` exist)?
    no  → ALWAYS run `init` first and choose `style`, `baseColor`,
          `cssVariables`, aliases. These are immutable.
    yes → Q2

Q2. Does the local copy of the block already exist (e.g. `app/dashboard/page.tsx` + `components/dashboard/...`)?
    no  → `pnpm dlx shadcn@latest add <block-id>` copies the full scaffold.
    yes → Q3 (re-install scenario)

Q3. Has the local copy been customised?
    no  → `pnpm dlx shadcn@latest add <block-id> --overwrite` is safe.
    yes → ALWAYS run `--diff` first, decide per file : skip, overwrite,
          or hand-merge. See `references/examples.md`.
```

## Decision Tree 4 : Did the block break after a rerun?

```
Q1. Did you re-run `shadcn add <block-id> --overwrite` and lose customisations?
    yes → ALWAYS recover from git. The local block files ARE your version control.
          See [shadcn-errors-cli-sync-mismatch] for the merge workflow.
    no  → Q2

Q2. Did the visual style suddenly look inconsistent across the page?
    yes → check whether multiple blocks installed at different times used
          different `style` values (highly unlikely if `components.json` is
          stable, but possible if `style` was edited manually). See
          `references/anti-patterns.md` anti-pattern 3.
    no  → Q3

Q3. Did an internal sub-component fail to import?
    yes → ALWAYS verify imports point at the LOCAL alias where the block was
          copied (e.g. `@/components/dashboard/...`), NOT at any external
          package path. See `references/anti-patterns.md` anti-pattern 4.
    no  → unrelated to blocks ; consult the component-specific error skill
```

## Patterns

### Pattern 1 : Block catalog overview

Verified at https://ui.shadcn.com/blocks (2026-05-19). The gallery groups blocks into categories. Headline examples observed :

| Category | Headline blocks (verified IDs) | Typical use |
|----------|-------------------------------|-------------|
| Dashboard | `dashboard-01` | Full admin / SaaS dashboard with sidebar, header, charts, data table |
| Sidebar | `sidebar-03`, `sidebar-07` | Standalone sidebar shells with various navigation patterns |
| Login | `login-03`, `login-04` | Authentication entry forms with branded layout |
| Signup | (gallery entry, verify current IDs at /blocks) | Signup / registration entry forms |
| Authentication | (gallery entry) | Broader auth flows (forgot password, two-factor, etc.) |
| Calendar | (gallery entry) | Calendar-centric layouts |

ALWAYS browse https://ui.shadcn.com/blocks before committing to a block-id ; the gallery is the canonical inventory. NEVER assume a block-id from memory or training data ; verify at install time.

### Pattern 2 : Block-add CLI flow

The CLI command is the SAME as for components ; the block-id is what signals a multi-file scaffold :

```bash
# component (single file output)
pnpm dlx shadcn@latest add button

# block (multi-file scaffold output)
pnpm dlx shadcn@latest add dashboard-01
```

After `add dashboard-01` the CLI typically copies :

- A page entry (e.g. `app/dashboard/page.tsx` on Next.js App Router projects).
- A folder of sub-components (e.g. `components/dashboard/`).
- Optionally a data file (e.g. `app/dashboard/data.json`) when the block demos data-driven UI.
- Any missing primitives the block depends on (e.g. `components/ui/sidebar.tsx`, `components/ui/data-table.tsx`).

The exact file list is documented per-block at the gallery preview. ALWAYS run `shadcn add <block-id> --dry-run` if uncertain about file output ; the dry-run prints the planned writes without touching disk. ALWAYS run `--view` to inspect a single file before installing.

For full CLI flag detail (`--overwrite`, `--diff`, `--dry-run`, `--view`, `--yes`, `--path`), see [shadcn-core-cli](../shadcn-core-cli/SKILL.md).

### Pattern 3 : Style enum (default / new-york / sera / luma)

Verified from https://ui.shadcn.com/docs/changelog (2026-05-19) :

| Style | Introduced | CLI minimum | Character |
|-------|-----------|-------------|-----------|
| `default` | Original (pre-2024) | any | Legacy. Deprecated for new projects. Retained for backward compatibility. |
| `new-york` | 2024 | any | Geometric, tight radii, design-system-grade. The conventional 2025-2026 choice. |
| `luma` | March 2026 | `shadcn@4.1.2` | Soft, ambient, gradient-friendly. SaaS-modern visual weight. |
| `sera` | April 2026 | `shadcn@4.3.0` | Typography-first. Built on print design principles. Underline controls, uppercase headings. |

The style is selected at `init` time :

```bash
pnpm dlx shadcn@latest init    # interactive prompt selects style
# or non-interactively :
pnpm dlx shadcn@latest init --style new-york
```

ALWAYS choose ONE style and keep it. NEVER mix styles within a project ; the visual language fragments because radii, spacing, and typographic defaults differ per style. The `style` field in `components.json` is **immutable** post-init in the operational sense : the CLI does not provide a switch command, and any attempt to edit it manually leaves the already-installed components in the OLD style.

### Pattern 4 : Customisation after add (ownership reinforcement)

Blocks inherit the **Open Code** pillar from `shadcn-core-architecture`. After `pnpm dlx shadcn@latest add dashboard-01` the consumer owns every file copied. Five concrete obligations :

1. ALWAYS commit every block file (`app/dashboard/page.tsx`, `components/dashboard/*`, data files) to source control as first-class application code.
2. ALWAYS lint and type-check block files together with the rest of the codebase.
3. ALWAYS expect re-adding the block to be a MANUAL merge operation, not an automatic upgrade.
4. ALWAYS feel free to delete or refactor any sub-component shipped with the block ; if the dashboard sidebar is irrelevant, remove its file.
5. ALWAYS prefer editing the block files in place over wrapping them in further indirection. Wrapper components defeat the Open Code pillar.

### Pattern 5 : Block and component composition

Blocks and components compose without friction. A common pattern : install a block for the page shell, then drop hand-composed components into the body slot.

```tsx
// app/dashboard/page.tsx (copied from `dashboard-01` block, then customised)
import { SidebarProvider, SidebarInset } from "@/components/ui/sidebar"
import { AppSidebar } from "@/components/dashboard/app-sidebar"   // block sub-component
import { CustomMetricsPanel } from "@/components/metrics-panel"   // hand-built component

export default function DashboardPage() {
  return (
    <SidebarProvider>
      <AppSidebar />
      <SidebarInset>
        <CustomMetricsPanel />
      </SidebarInset>
    </SidebarProvider>
  )
}
```

ALWAYS treat a block as the page skeleton and components as the organs inside it. NEVER feel obliged to keep block sub-components untouched ; they are owned code and can be replaced wholesale.

### Pattern 6 : Block vs custom-registry relationship

Blocks live in the default `@shadcn` registry alongside components. The `registry.json` schema (see [shadcn-core-registry](../shadcn-core-registry/SKILL.md)) accepts a `type` discriminator that identifies the registry item as a block (`registry:block`) or a component (`registry:ui`, `registry:lib`, etc.). The CLI uses this discriminator internally to decide how to copy files. Consumers do NOT supply the type ; the registry does.

Custom registries (private internal design systems) CAN publish their own blocks by setting `type: "registry:block"` on registry items. ALWAYS see [shadcn-core-registry](../shadcn-core-registry/SKILL.md) for the full schema. NEVER hand-edit registry JSON files in the consumer project ; the registry is the publisher's surface, not the consumer's.

### Pattern 7 : Versioning quirks specific to blocks

Blocks depend on specific components (a dashboard block depends on Sidebar, DataTable, Card, etc.). When the shadcn project ships a new version of an underlying component, the block may quietly depend on the new API surface. Implications :

- ALWAYS pin the working state of block files in git immediately after `add`. The committed source IS the version.
- ALWAYS verify breaking changes via https://ui.shadcn.com/docs/changelog BEFORE re-adding a block (especially after a Calendar v9 transition or cmdk drift incident).
- ALWAYS lock underlying runtime dependencies (`radix-ui`, `cva`, `tailwind-merge`, `react-day-picker`, `cmdk`, `lucide-react`) in `package.json` so that re-adding a block does not silently install a new major of an underlying primitive.

NEVER expect blocks to auto-upgrade. The ownership doctrine is non-negotiable.

## Anti-patterns (summary ; full catalogue in references/anti-patterns.md)

NEVER do these :

1. NEVER treat a block as an immutable starter template : a block is owned code and SHOULD be edited freely after `add`.
2. NEVER re-run `shadcn add <block-id> --overwrite` on a customised block without first running `--diff` : the overwrite is irreversible and loses work.
3. NEVER mix multiple `style` enum values in one project : visual coherence breaks because radii, spacing, and typography differ per style.
4. NEVER import a block sub-component from any path other than the local alias where it was copied (e.g. `@/components/dashboard/...`).
5. NEVER expect blocks to auto-update when shadcn ships new block revisions : they are local source ; upgrades are manual re-add + merge.

## Reference Links

- [references/methods.md](references/methods.md) : CLI signatures for `add`, registry-item `type` discriminator, exact style-enum values and minimum CLI versions
- [references/examples.md](references/examples.md) : dashboard block install, file-output expectations, customisation diff, style switch workaround, block + component composition
- [references/anti-patterns.md](references/anti-patterns.md) : five canonical anti-patterns with WHY each fails and the fix

## Companion Skills

- [shadcn-core-architecture](../shadcn-core-architecture/SKILL.md) : ownership doctrine reinforcement (blocks obey the same Open Code pillar)
- [shadcn-core-cli](../shadcn-core-cli/SKILL.md) : full CLI surface (`init`, `add`, `--diff`, `--overwrite`, `--dry-run`, `--view`)
- [shadcn-core-registry](../shadcn-core-registry/SKILL.md) : `registry.json` schema, `type: "registry:block"` discriminator
- [shadcn-core-theming](../shadcn-core-theming/SKILL.md) : how the style enum interacts with CSS variables and dark mode
- [shadcn-errors-cli-sync-mismatch](../../shadcn-errors/shadcn-errors-cli-sync-mismatch/SKILL.md) : recovering from accidental overwrite of customised blocks

## Sources

All claims in this skill trace to URLs in `SOURCES.md` :

- https://ui.shadcn.com/blocks (gallery inventory, categories, headline block IDs, verbatim tagline)
- https://ui.shadcn.com/docs/cli (the `add` command and its flags, identical for blocks and components)
- https://ui.shadcn.com/docs/changelog (style enum additions : `luma` March 2026, `sera` April 2026, CLI version gates)
- https://ui.shadcn.com/docs/components-json (style field semantics, immutability after init)
- https://ui.shadcn.com/docs/registry (registry item type discriminator)

Verified 2026-05-19.
