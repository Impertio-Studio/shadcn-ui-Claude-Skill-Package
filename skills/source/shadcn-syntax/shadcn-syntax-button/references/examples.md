# Button : Examples

Ten canonical recipes. Each example annotates the Tailwind version (v3 or v4) where the choice matters. Every imported symbol traces to a real export documented at `https://ui.shadcn.com/docs/components/radix/button` or in the project's own `components/ui/button.tsx` after `pnpm dlx shadcn@latest add button`.

## 1. Basic button (default variant, default size)

```tsx
// Works identically on Tailwind v3 and v4.
import { Button } from "@/components/ui/button"

export function SaveAction() {
  return <Button>Save</Button>
}
```

`variant` and `size` default to `"default"` via cva's `defaultVariants` AND via explicit JS defaults on the function signature. The rendered HTML is `<button data-slot="button" data-variant="default" data-size="default" class="...">Save</button>` (on v4 ; v3 omits the data-attributes).

## 2. Destructive button with a confirmation dialog

```tsx
// Works on v3 and v4. Shows the canonical "destructive-action-needs-confirmation" pattern.
import {
  AlertDialog,
  AlertDialogAction,
  AlertDialogCancel,
  AlertDialogContent,
  AlertDialogDescription,
  AlertDialogFooter,
  AlertDialogHeader,
  AlertDialogTitle,
  AlertDialogTrigger,
} from "@/components/ui/alert-dialog"
import { Button } from "@/components/ui/button"

export function DeleteAccountButton({ onConfirm }: { onConfirm: () => void }) {
  return (
    <AlertDialog>
      <AlertDialogTrigger asChild>
        <Button variant="destructive">Delete account</Button>
      </AlertDialogTrigger>
      <AlertDialogContent>
        <AlertDialogHeader>
          <AlertDialogTitle>Are you absolutely sure?</AlertDialogTitle>
          <AlertDialogDescription>
            This action cannot be undone. This will permanently delete your account.
          </AlertDialogDescription>
        </AlertDialogHeader>
        <AlertDialogFooter>
          <AlertDialogCancel>Cancel</AlertDialogCancel>
          <AlertDialogAction onClick={onConfirm}>Delete</AlertDialogAction>
        </AlertDialogFooter>
      </AlertDialogContent>
    </AlertDialog>
  )
}
```

ALWAYS pair `variant="destructive"` with an AlertDialog confirmation when the action is irreversible. NEVER trigger a destructive operation on the first click.

## 3. Variant catalogue : outline, ghost, link, secondary

```tsx
// Works on v3 and v4. Demonstrates each non-primary variant in context.
import { Button } from "@/components/ui/button"

export function VariantShowcase() {
  return (
    <div className="flex gap-2">
      <Button variant="outline">Cancel</Button>
      <Button variant="secondary">Save draft</Button>
      <Button variant="ghost">Dismiss</Button>
      <Button variant="link">Forgot password?</Button>
    </div>
  )
}
```

| Variant | Reads as | Pairs with |
|---------|----------|------------|
| `outline` | Alternate path | A primary `default` button to its right |
| `secondary` | Muted primary | A primary `default` button when both are "real" actions |
| `ghost` | Low chrome action | Toolbars, sidebars, table rows |
| `link` | Inline navigation | An `asChild + <Link>` wrap (see Recipe 5) |

## 4. Size variants : sm, lg, icon

```tsx
// v4 ONLY for `xs`, `icon-xs`, `icon-sm`, `icon-lg`. v3 has only `default / sm / lg / icon`.
import { Button } from "@/components/ui/button"
import { Plus } from "lucide-react"

export function SizeShowcase() {
  return (
    <div className="flex items-center gap-2">
      <Button size="sm">Small</Button>
      <Button>Default</Button>
      <Button size="lg">Large</Button>
      <Button size="icon" aria-label="Add item">
        <Plus />
      </Button>
    </div>
  )
}
```

ALWAYS provide `aria-label` on an icon-only button. The visible content is empty for assistive tech otherwise.

## 5. asChild with Next.js Link (App Router, Next.js 15+)

```tsx
// Works on v3 and v4. Uses next/link to keep prefetch + middle-click semantics.
import Link from "next/link"
import { Button } from "@/components/ui/button"

export function LoginNav() {
  return (
    <Button asChild>
      <Link href="/login">Login</Link>
    </Button>
  )
}
```

The rendered HTML is an `<a>` element with the Button's classes, data-attributes, and event handlers merged onto it. Next.js Link's prefetch behaviour, scroll restoration, and `replace` prop continue to work because the underlying element IS the Link.

NEVER replace this with `<Button onClick={() => router.push("/login")}>`. See `references/anti-patterns.md` anti-pattern 5.

## 6. asChild with react-router-dom Link

```tsx
// Works on react-router-dom v6 and react-router v7 (Remix rebrand).
import { Link } from "react-router-dom"
import { Button } from "@/components/ui/button"

export function ForgotPasswordLink() {
  return (
    <Button asChild variant="link">
      <Link to="/forgot-password">Forgot password?</Link>
    </Button>
  )
}
```

The `to` prop is react-router-specific ; the Button merges its own props (className, data-*) onto the Link without disturbing the navigation API. ALWAYS combine `variant="link"` with `asChild + <Link>` when the action is semantically a link AND visually a link.

## 7. Icon-only button with lucide-react

```tsx
// Works on v3 and v4. The cva base layer auto-sizes the icon to 16px (size-4).
import { Button } from "@/components/ui/button"
import { Settings } from "lucide-react"

export function SettingsTrigger({ onOpen }: { onOpen: () => void }) {
  return (
    <Button size="icon" variant="ghost" aria-label="Open settings" onClick={onOpen}>
      <Settings />
    </Button>
  )
}
```

`size="icon"` produces a square button (`size-9` on v4 = 36px ; `h-10 w-10` on v3 = 40px). The lucide icon needs no explicit `className="size-4"` because the cva base layer includes `[&_svg:not([class*='size-'])]:size-4` (v4) or `[&_svg]:size-4` (v3).

## 8. Loading button (spinner + disabled state)

```tsx
// Works on v3 and v4. Uses `useTransition` for the canonical Next.js Server Action pattern.
"use client"

import * as React from "react"
import { Loader2 } from "lucide-react"
import { Button } from "@/components/ui/button"

export function SubmitButton({ action }: { action: () => Promise<void> }) {
  const [isPending, startTransition] = React.useTransition()

  return (
    <Button
      disabled={isPending}
      onClick={() => startTransition(() => action())}
    >
      {isPending && <Loader2 className="animate-spin" />}
      {isPending ? "Saving..." : "Save"}
    </Button>
  )
}
```

`disabled:pointer-events-none disabled:opacity-50` from the cva base layer handles the disabled visual state. The `Loader2` icon auto-sizes via the same `[&_svg]` rule that handles a leading icon. ALWAYS combine `disabled={isPending}` with the spinner ; NEVER show the spinner without disabling, otherwise the user can submit twice.

## 9. Custom variant : extend buttonVariants with `warning`

```tsx
// Editing the LOCAL `components/ui/button.tsx` directly. This is the
// shadcn "Open Code" pillar in practice. Works on v3 and v4 (only the
// base layer differs ; the variant addition pattern is identical).

// components/ui/button.tsx
const buttonVariants = cva(
  /* ...existing base layer... */,
  {
    variants: {
      variant: {
        default: "bg-primary text-primary-foreground hover:bg-primary/90",
        destructive: /* ... */,
        outline: /* ... */,
        secondary: /* ... */,
        ghost: /* ... */,
        link: /* ... */,
        // NEW : project-specific warning variant.
        warning:
          "bg-amber-500 text-amber-950 hover:bg-amber-500/90 focus-visible:ring-amber-500/30",
      },
      size: { /* unchanged */ },
    },
    defaultVariants: { variant: "default", size: "default" },
  }
)
```

After the edit, `<Button variant="warning">` type-checks automatically because `VariantProps<typeof buttonVariants>` re-derives the union. NEVER hand-edit the consumer's TypeScript type after extending the cva ; let inference do the work.

ALWAYS pick colour tokens that exist in the project's Tailwind theme. NEVER hard-code raw hex values in the variant string ; route them through `globals.css` design tokens so dark mode still works.

## 10. Tailwind v3 ring-2 vs Tailwind v4 ring-[3px] (annotated)

```tsx
// v3 form (registry "default" style, ships when Tailwind v3 is detected at init):
const buttonVariants = cva(
  "inline-flex items-center justify-center gap-2 whitespace-nowrap rounded-md text-sm font-medium ring-offset-background transition-colors " +
  // v3 focus ring : 2px ring, 2px offset, named ring-ring token.
  "focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2 " +
  "disabled:pointer-events-none disabled:opacity-50 [&_svg]:pointer-events-none [&_svg]:size-4 [&_svg]:shrink-0",
  /* variants ... */
)

// v4 form (registry "new-york-v4" style, ships when Tailwind v4 is detected at init):
const buttonVariants = cva(
  "inline-flex shrink-0 items-center justify-center gap-2 rounded-md text-sm font-medium whitespace-nowrap transition-all " +
  // v4 focus ring : 3px ring via arbitrary-value, no ring-offset, border-colour shift on focus.
  "outline-none focus-visible:border-ring focus-visible:ring-[3px] focus-visible:ring-ring/50 " +
  "disabled:pointer-events-none disabled:opacity-50 " +
  "aria-invalid:border-destructive aria-invalid:ring-destructive/20 dark:aria-invalid:ring-destructive/40 " +
  "[&_svg]:pointer-events-none [&_svg]:shrink-0 [&_svg:not([class*='size-'])]:size-4",
  /* variants ... */
)
```

Detection rules :

1. Check `package.json` : `tailwindcss` major version `3.x` => emit v3 form ; major version `4.x` => emit v4 form.
2. Or check `globals.css` : `@tailwind base ;` directives => v3 ; `@import "tailwindcss" ;` => v4.
3. Or check `components.json` (rare) : the CLI persists the registry style used at init.

ALWAYS keep the focus-ring incantation consistent with the rest of the project's components. NEVER mix v3's `ring-offset-background` with v4's `ring-[3px]` in one button file ; they layer poorly and the offset reads as a double-border under v4 semantics.

## Cross-references

- See `references/methods.md` for the verbatim cva instance and the v3 vs v4 prop-signature diff.
- See `references/anti-patterns.md` for the six canonical Button-level failures.
- See `shadcn-syntax-variant-cva` for the deeper variant API (compoundVariants, conditional variants, custom variant typing).
- See `shadcn-core-architecture` for the "Open Code" pillar that justifies editing `components/ui/button.tsx` directly (Recipe 9).

Verified against the canonical sources on 2026-05-19.
