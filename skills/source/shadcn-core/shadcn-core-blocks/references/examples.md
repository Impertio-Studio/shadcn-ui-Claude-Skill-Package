# shadcn-core-blocks : Examples

Working code samples and CLI sessions for the block distribution surface. All examples verified against `SOURCES.md` primary URLs on 2026-05-19. ALWAYS treat copied block files as project source ; NEVER as vendored dependencies.

## Example 1 : Add a dashboard block

Add `dashboard-01` from the default `@shadcn` registry to a Next.js App Router project that has been initialised with `pnpm dlx shadcn@latest init` and the `new-york` style.

### CLI invocation

```bash
# preview first (no disk writes)
pnpm dlx shadcn@latest add dashboard-01 --dry-run

# inspect a single file before install
pnpm dlx shadcn@latest add dashboard-01 --view app/dashboard/page.tsx

# perform the install
pnpm dlx shadcn@latest add dashboard-01
```

### Expected file output

The exact tree is documented at the per-block preview on https://ui.shadcn.com/blocks. A typical `dashboard-01` install produces :

```
app/
  dashboard/
    page.tsx                          # block page entry (registry:page)
    data.json                         # demo data (registry:file)
components/
  dashboard/
    nav-main.tsx                      # block sub-component
    nav-user.tsx                      # block sub-component
    app-sidebar.tsx                   # block sub-component
    data-table.tsx                    # block sub-component
    chart-area-interactive.tsx        # block sub-component
    section-cards.tsx                 # block sub-component
    site-header.tsx                   # block sub-component
  ui/
    sidebar.tsx                       # auto-installed primitive (registry:ui)
    chart.tsx                         # auto-installed primitive
    data-table.tsx                    # auto-installed primitive
    card.tsx                          # auto-installed primitive
    ... (any other missing primitives)
lib/
  utils.ts                            # auto-installed if missing
```

ALWAYS commit ALL of these files in the SAME commit. The block is now project source.

### Verify post-install

```bash
# confirm the page route serves the block
pnpm dev
# visit http://localhost:3000/dashboard

# type-check
pnpm tsc --noEmit

# lint
pnpm lint
```

NEVER move the copied files to a vendored or `node_modules`-shaped path. They live in `app/`, `components/`, `lib/` as first-class application code.

## Example 2 : Customise a block after add

Scenario : the team installed `dashboard-01` and wants to replace the demo data source with a Supabase query.

### Before (as shipped by the block)

```tsx
// app/dashboard/page.tsx (as copied by `shadcn add dashboard-01`)
import data from "./data.json"
import { DataTable } from "@/components/dashboard/data-table"

export default function Page() {
  return <DataTable data={data} />
}
```

### After (customised)

```tsx
// app/dashboard/page.tsx (customised by the team)
import { createServerClient } from "@/lib/supabase/server"
import { DataTable } from "@/components/dashboard/data-table"

export default async function Page() {
  const supabase = createServerClient()
  const { data, error } = await supabase.from("transactions").select("*")
  if (error) throw error
  return <DataTable data={data} />
}
```

### What this demonstrates

- The block file is owned code ; editing it is the intended workflow.
- The page becomes an async Server Component because Next.js App Router supports it ; the block was a starter, not an immutable contract.
- The local sub-component `@/components/dashboard/data-table` continues to work because its imports were never package-shaped.

ALWAYS commit customisations on the SAME branch where the block was initially added. NEVER mix the `add` commit with the customisation commit ; keeping them separate makes future re-add merges tractable.

## Example 3 : Re-adding a customised block safely

Scenario : six months later the team wants to pull updates from `dashboard-01`. The local `app/dashboard/page.tsx` has been heavily customised.

### Wrong workflow

```bash
# DESTRUCTIVE : overwrites customisations without warning
pnpm dlx shadcn@latest add dashboard-01 --overwrite
```

This is the single most common block-system disaster ; the customisation work disappears.

### Correct workflow

```bash
# 1. preview the diff for each file the block would write
pnpm dlx shadcn@latest add dashboard-01 --diff

# example diff output for one file :
# app/dashboard/page.tsx
#   - import data from "./data.json"
#   + import data from "./data.json"
#   + import { revalidatePath } from "next/cache"
#   ... 12 more changed lines ...

# 2. decide per file :
#    - block sub-component you never touched   → overwrite is safe
#    - file you customised                     → SKIP or hand-merge
#
# 3. for individual files you want to overwrite :
pnpm dlx shadcn@latest add dashboard-01 --overwrite --path components/dashboard/section-cards.tsx

# 4. for files you customised, do NOT overwrite ; apply the diff by hand using git :
git diff HEAD~1 -- app/dashboard/page.tsx
# manually merge the upstream changes into your customised file
```

ALWAYS preview with `--diff` before any rerun. NEVER use `--overwrite` blanket on a customised block.

## Example 4 : Switching style enum mid-project (the only correct path)

Scenario : the team initialised the project with `default` and wants to migrate to `new-york`.

### Why the obvious approach fails

```bash
# WRONG : editing components.json manually does NOT re-style existing components
sed -i 's/"style": "default"/"style": "new-york"/' components.json
```

The already-installed components have the `default` style's classes and variant compositions baked in. Editing `components.json` only changes what FUTURE `shadcn add` runs produce. The project ends up half in `default`, half in `new-york`. Visually broken.

### The only correct workflow

```bash
# 1. snapshot the project on a branch
git checkout -b chore/style-migration

# 2. record the list of installed components and blocks
pnpm dlx shadcn@latest info > installed-before.txt

# 3. delete the existing components.json and the components/ui/ folder
rm components.json
rm -rf components/ui/

# 4. re-init with the new style
pnpm dlx shadcn@latest init --style new-york

# 5. re-add every component and block that was previously installed
pnpm dlx shadcn@latest add button card dialog sidebar data-table
pnpm dlx shadcn@latest add dashboard-01 sidebar-07 login-03
# (etc. for the full inventory recorded in step 2)

# 6. manually reconcile any customisations from the deleted files using git history
git log --all -- components/ui/button.tsx
git show <commit>:components/ui/button.tsx     # retrieve the customised version
# port the customisations into the new new-york version of button.tsx

# 7. run type-check, lint, and visual review before merging
pnpm tsc --noEmit
pnpm lint
pnpm test
```

ALWAYS treat a style migration as a project-wide refactor, not a config tweak. NEVER edit the `style` field in isolation ; the result is silent visual fragmentation.

## Example 5 : Block + component composition

Use `sidebar-07` as the layout shell, then drop a hand-built panel into the inset.

### Step 1 : install the sidebar block

```bash
pnpm dlx shadcn@latest add sidebar-07
```

This produces (typical) :

```
app/
  layout.tsx (or page.tsx)             # block layout entry
components/
  app-sidebar.tsx                      # block sidebar component
  nav-projects.tsx                     # block sub-component
  nav-secondary.tsx                    # block sub-component
  nav-user.tsx                         # block sub-component
  ui/
    sidebar.tsx                        # primitive (auto-installed)
```

### Step 2 : write the hand-built panel

```tsx
// components/projects-panel.tsx (NOT shipped by any block ; the team's own code)
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card"
import { Button } from "@/components/ui/button"
import { PlusIcon } from "lucide-react"

export function ProjectsPanel({ projects }: { projects: { id: string; name: string }[] }) {
  return (
    <Card>
      <CardHeader>
        <CardTitle>Projects</CardTitle>
        <Button size="sm">
          <PlusIcon className="mr-1 size-4" />
          New
        </Button>
      </CardHeader>
      <CardContent>
        <ul>
          {projects.map((p) => (
            <li key={p.id}>{p.name}</li>
          ))}
        </ul>
      </CardContent>
    </Card>
  )
}
```

### Step 3 : compose them in the page

```tsx
// app/page.tsx (edited block output)
import { SidebarProvider, SidebarInset } from "@/components/ui/sidebar"
import { AppSidebar } from "@/components/app-sidebar"        // from sidebar-07 block
import { ProjectsPanel } from "@/components/projects-panel"  // hand-built

export default function HomePage() {
  return (
    <SidebarProvider>
      <AppSidebar />
      <SidebarInset className="p-6">
        <ProjectsPanel projects={[{ id: "1", name: "Alpha" }, { id: "2", name: "Beta" }]} />
      </SidebarInset>
    </SidebarProvider>
  )
}
```

ALWAYS treat blocks as page skeletons and components as organs inside them. NEVER feel obliged to keep block sub-components untouched ; replace, refactor, or delete them as the project demands.

## Sources

- https://ui.shadcn.com/blocks (per-block file inventory, gallery preview)
- https://ui.shadcn.com/docs/cli (`add`, `--dry-run`, `--diff`, `--view`, `--overwrite`, `--path`)
- https://ui.shadcn.com/docs/components-json (`style` field, immutability)
- https://ui.shadcn.com/docs/changelog (style enum dates, CLI version gates)

Verified 2026-05-19.
