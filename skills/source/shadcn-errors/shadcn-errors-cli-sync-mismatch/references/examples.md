# shadcn CLI Sync Mismatch : Worked Examples

Every example assumes a project initialized via `pnpm dlx shadcn@latest
init` with `components.json` and at least one shadcn component already
installed.

## Example 1 : Diff Workflow Before Re-Adding

Scenario : you want to pull a recent upstream fix for `Calendar`
(react-day-picker v9 break, see vooronderzoek §10 issue #4366). You
have a small local edit : you added a `showWeekNumber` default prop.

Step 1 : clean tree, branch out.

```bash
git status
# On branch main
# nothing to commit, working tree clean

git switch -c chore/sync-calendar
```

Step 2 : probe the change.

```bash
pnpm dlx shadcn@latest add calendar --dry-run
# - components/ui/calendar.tsx          (would overwrite)
# - 1 file would be written.
```

Step 3 : view the diff.

```bash
pnpm dlx shadcn@latest add calendar --diff
# (prints unified diff against components/ui/calendar.tsx)
# ...
# -  selected,
# +  selected: selectedDate,
# +  onSelect,
# -  mode = "single",
# +  mode = "single" as const,
# ...
```

The diff shows : props renamed for v9 compatibility (`selected` ->
`selectedDate`), `mode` typed as a const. Your `showWeekNumber`
default is on a different line. Safe to overwrite.

Step 4 : apply.

```bash
pnpm dlx shadcn@latest add calendar --overwrite
# - components/ui/calendar.tsx          (overwritten)
```

Step 5 : re-apply local change.

```bash
git diff HEAD -- components/ui/calendar.tsx
# (the file is now upstream-clean ; your showWeekNumber default is gone)
```

Edit the file to re-introduce the local change :

```tsx
// components/ui/calendar.tsx (after re-add, with local edit re-applied)
function Calendar({
  className,
  classNames,
  showOutsideDays = true,
  showWeekNumber = true,           // RE-APPLIED 2026-05-19 (local default)
  ...props
}: React.ComponentProps<typeof DayPicker>) {
```

Step 6 : test and commit.

```bash
pnpm test
git add components/ui/calendar.tsx
git commit -m "chore: sync Calendar with shadcn@latest for react-day-picker v9 fix"
```

Lesson : because the local edit was a single default value on an
unaffected line, the manual re-apply took 30 seconds. If your edits
had been heavier, the next example (custom variants in a separate
file) shows the better long-term shape.

## Example 2 : Custom Warning Variant in a Separate File

Scenario : your design system needs a "warning" Button variant (amber
background). You want it to survive any future `shadcn add button
--overwrite`.

WRONG (inline, fragile) :

```tsx
// components/ui/button.tsx   - DO NOT do this
const buttonVariants = cva(
  "inline-flex items-center justify-center ...",
  {
    variants: {
      variant: {
        default: "...",
        destructive: "...",
        outline: "...",
        secondary: "...",
        ghost: "...",
        link: "...",
        warning: "bg-amber-500 text-amber-50 hover:bg-amber-600", // FRAGILE
      },
      size: { ... },
    },
    defaultVariants: { ... },
  }
)
```

The next `shadcn add button --overwrite` deletes that line silently.

RIGHT (separate file, durable) :

```tsx
// components/ui/button-extensions.tsx
import * as React from "react"
import { cva, type VariantProps } from "class-variance-authority"
import { Button as BaseButton } from "@/components/ui/button"
import { cn } from "@/lib/utils"

const extButtonVariants = cva("", {
  variants: {
    intent: {
      warning: "bg-amber-500 text-amber-50 hover:bg-amber-600 focus-visible:ring-amber-500",
      success: "bg-emerald-600 text-emerald-50 hover:bg-emerald-700 focus-visible:ring-emerald-600",
      info:    "bg-sky-600    text-sky-50    hover:bg-sky-700    focus-visible:ring-sky-600",
    },
  },
})

type ExtButtonProps = React.ComponentProps<typeof BaseButton> &
  VariantProps<typeof extButtonVariants>

const ExtButton = React.forwardRef<HTMLButtonElement, ExtButtonProps>(
  ({ className, intent, ...props }, ref) => {
    return (
      <BaseButton
        ref={ref}
        className={cn(extButtonVariants({ intent }), className)}
        {...props}
      />
    )
  }
)
ExtButton.displayName = "ExtButton"

export { ExtButton, extButtonVariants }
```

Usage :

```tsx
import { ExtButton } from "@/components/ui/button-extensions"

<ExtButton intent="warning">Confirm deletion</ExtButton>
<ExtButton intent="success" size="lg">Saved</ExtButton>
```

Now `pnpm dlx shadcn@latest add button --overwrite` is safe. The
`button.tsx` file is restored to the registry version ; the
`button-extensions.tsx` file is untouched ; every call site keeps
working.

The same pattern applies to Card (`card-extensions.tsx`), Input,
Badge, etc. ONE rule : NEVER edit the registry file ; ALWAYS extend
in a sibling file.

## Example 3 : Vendor Pattern (Locked Snapshot)

Scenario : you have heavily edited `components/ui/badge.tsx` to match
a brand color palette and add three custom variants. You do not need
upstream changes ; Badge has been stable for two years.

Step 1 : mark the file as vendored at the top.

```tsx
// components/ui/badge.tsx
//
// VENDORED 2026-05-19 from shadcn@latest.
// Reason : brand-color palette + extra variants do not survive re-add.
// Action : DO NOT run `shadcn add badge` for this project.
//          If upstream has a security fix worth pulling, switch to
//          diff-merge strategy : refactor variants into
//          badge-extensions.tsx FIRST, then accept the overwrite.

import * as React from "react"
import { Slot } from "radix-ui/slot"
import { cva, type VariantProps } from "class-variance-authority"
import { cn } from "@/lib/utils"

const badgeVariants = cva(
  "inline-flex items-center rounded-md px-2 py-0.5 text-xs font-medium ...",
  {
    variants: {
      variant: {
        default:     "bg-brand text-brand-foreground",
        destructive: "bg-destructive text-destructive-foreground",
        outline:     "border text-foreground",
        warning:     "bg-amber-500 text-amber-50",
        success:     "bg-emerald-600 text-emerald-50",
        info:        "bg-sky-600 text-sky-50",
      },
    },
    defaultVariants: { variant: "default" },
  }
)
```

Step 2 : record in a project-level `components/ui/README.md`.

```md
# Component Sync Policy

| Component | Strategy | Last action | Note |
|-----------|----------|-------------|------|
| badge     | VENDORED | 2026-05-19  | brand colors + 3 extra variants |
| button    | TRACKED  | 2026-04-12  | extensions in button-extensions.tsx |
| calendar  | TRACKED  | 2026-05-19  | react-day-picker v9 sync done |
| sidebar   | FORKED   | 2026-03-01  | real code in lib/components/sidebar |
```

Step 3 : never re-run `shadcn add badge`. A teammate who tries gets
slapped by the top-of-file VENDORED comment in code review.

## Example 4 : Fork Pattern (Copy to lib/components/)

Scenario : your Sidebar has divergent navigation logic, a custom
collapsible behavior, and a different mobile layout. Continuing to
edit `components/ui/sidebar.tsx` makes future `shadcn add` runs
hazardous, even with the diff workflow.

Step 1 : add (or re-add) the upstream version cleanly.

```bash
pnpm dlx shadcn@latest add sidebar --overwrite
```

Step 2 : copy to a new home.

```bash
mkdir -p lib/components/sidebar
cp components/ui/sidebar.tsx lib/components/sidebar/sidebar.tsx
```

Step 3 : rewrite imports in the fork to point at the same primitives.

```tsx
// lib/components/sidebar/sidebar.tsx
import { Slot } from "radix-ui/slot"
import { Sheet, SheetContent } from "@/components/ui/sheet"
import { cn } from "@/lib/utils"
// ...your custom logic here...
```

Step 4 : update consumer imports.

```diff
- import { Sidebar, SidebarProvider } from "@/components/ui/sidebar"
+ import { Sidebar, SidebarProvider } from "@/lib/components/sidebar/sidebar"
```

Step 5 : keep `components/ui/sidebar.tsx` ONLY as the upstream
reference. To pull upstream changes :

```bash
pnpm dlx shadcn@latest add sidebar --diff       # see what upstream changed
pnpm dlx shadcn@latest add sidebar --overwrite  # update the reference copy
diff -u components/ui/sidebar.tsx lib/components/sidebar/sidebar.tsx \
  | less                                        # see what to port to the fork
```

Step 6 : record in `components/ui/README.md` :

```md
| sidebar | FORKED-TO lib/components/sidebar/ | reference kept in components/ui/ |
```

This is the most disruptive of the three strategies, but it is the
ONLY one safe for components where customization volume exceeds
upstream churn rate.

## Example 5 : Migrate Icons (lucide -> radix)

Scenario : your team decides to standardize on `@radix-ui/react-icons`
instead of `lucide-react`. Every `components/ui/<name>.tsx` file
imports icons.

Step 1 : clean tree, branch.

```bash
git status                            # clean
git switch -c chore/migrate-icons
```

Step 2 : list available targets.

```bash
pnpm dlx shadcn@latest migrate icons --list
# Available icon libraries :
#   lucide      lucide-react
#   radix       @radix-ui/react-icons
#   tabler      @tabler/icons-react
```

Step 3 : run the migration (interactive picks the target).

```bash
pnpm dlx shadcn@latest migrate icons
# ? Which icon library would you like to use? radix
# Updated: components/ui/dialog.tsx
# Updated: components/ui/select.tsx
# Updated: components/ui/dropdown-menu.tsx
# ...
```

Step 4 : review.

```bash
git diff HEAD
# (every changed import is from lucide-react -> @radix-ui/react-icons,
#  plus icon JSX symbol-name rewrites)
```

Step 5 : update `components.json` to record the new default.

```diff
- "iconLibrary": "lucide"
+ "iconLibrary": "radix"
```

Step 6 : install the new dep, remove the old.

```bash
pnpm add @radix-ui/react-icons
pnpm remove lucide-react
pnpm test
git add -A
git commit -m "chore: migrate icon library to @radix-ui/react-icons"
```

Caveat : if your `*-extensions.tsx` files also import lucide icons,
the migrate command will NOT update them (it only touches
`components/ui/<name>.tsx` files matching the registry shape). You
must update those by hand AFTER the migrate run. Search for stragglers :

```bash
grep -r "from \"lucide-react\"" src/ lib/ components/ | grep -v "components/ui/"
```

## Example 6 : Refactor Inline Customizations Before First Overwrite

Scenario : you inherit a project with inline custom variants in
`components/ui/button.tsx`. Upstream has a security fix and you want
to overwrite, but the inline edits will be lost.

WRONG : just run `add button --overwrite` and re-edit by memory.

RIGHT : refactor FIRST, overwrite SECOND.

Step 1 : extract the custom variants to `button-extensions.tsx`
(see Example 2 above). Commit this refactor on its own branch.

```bash
git switch -c refactor/button-extract-variants
# (write components/ui/button-extensions.tsx as in Example 2,
#  remove the warning/success/info entries from components/ui/button.tsx)
pnpm test
git add components/ui/button.tsx components/ui/button-extensions.tsx
git commit -m "refactor: extract custom Button variants to button-extensions.tsx"
```

Step 2 : merge the refactor (PR review confirms nothing visual
changed). NOW the base file is structurally identical to the registry
version except for tiny incidental edits.

Step 3 : sync.

```bash
git switch -c chore/sync-button
pnpm dlx shadcn@latest add button --diff
# Diff is now manageable : only the upstream security fix lines change.
pnpm dlx shadcn@latest add button --overwrite
pnpm test
git add components/ui/button.tsx
git commit -m "chore: sync Button with shadcn@latest for security fix"
```

The customizations in `button-extensions.tsx` are untouched. Every
call site that uses `intent="warning"` keeps working.
