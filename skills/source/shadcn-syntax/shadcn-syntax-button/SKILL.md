---
name: shadcn-syntax-button
description: >
  Use when adding, customising, or debugging a shadcn ui Button, choosing
  between the six built-in variants (default / destructive / outline /
  secondary / ghost / link), choosing between the size variants (default /
  sm / lg / icon plus the v4-only xs / icon-xs / icon-sm / icon-lg),
  using the `asChild` Slot pattern to wrap a Next.js Link, a react-router
  Link, or any other custom anchor as the rendered button element, typing
  a Button-derived component with `VariantProps<typeof buttonVariants>`,
  building an icon-only button, building a loading button with a spinner,
  understanding the focus-ring difference between Tailwind v3 and v4
  output, or diagnosing why a Button renders unstyled, ignores `className`,
  or crashes the page after `asChild` is added.
  Prevents the canonical Button-level failures : passing multiple
  children inside `<Button asChild>` (Slot.Root only accepts one), using
  `asChild` with a child that is not a forwardRef-compatible component,
  dropping an icon into `size="default"` instead of `size="icon"` and
  ending up with an over-padded square, hand-concatenating className
  strings instead of routing through `cn()` so that the caller's classes
  silently lose the twMerge conflict resolution, and replacing
  `<Button asChild><Link/></Button>` with
  `<button onClick={() => router.push()}>` which loses Next.js prefetch
  and breaks middle-click / keyboard accessibility.
  Covers the full buttonVariants cva() catalogue, the VariantProps typing
  pattern, the Slot.Root vs `"button"` swap that powers `asChild`, the
  Tailwind v3 (`focus-visible:ring-2 ring-ring ring-offset-2`) versus
  Tailwind v4 (`focus-visible:ring-[3px] ring-ring/50`) focus-ring shift,
  the icon-button + loading-button + linked-button recipes, RSC safety
  (Button is pure presentational, no `"use client"` directive needed),
  and how to extend buttonVariants with a project-specific variant such
  as `warning`.
  Keywords: shadcn button, Button component, button variant, button size,
  destructive button, outline button, ghost button, link variant,
  secondary button, icon button, size icon, asChild, Slot, Slot.Root,
  Button as Link, Button as anchor, Next.js Link button, react-router
  Link button, VariantProps, buttonVariants, cn helper, twMerge,
  Tailwind v3 ring, Tailwind v4 ring, focus-visible ring,
  how do I make a button, button not styled, button has wrong cursor,
  Button.Children Slot error, multiple children Slot crash,
  loading button, button with spinner, button disabled state,
  extend buttonVariants, custom variant warning, button data-slot,
  button data-variant.
license: MIT
compatibility: "Designed for Claude Code. Requires shadcn ui evergreen-2026."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# shadcn ui : Button Syntax

The Button component is shadcn's reference implementation of the cva-variant pattern. Every other variant-driven primitive (Badge, Alert, Toggle, ToggleGroup) follows the same shape. Mastering Button means mastering the shadcn variant API.

## Quick Reference

### Canonical call

```tsx
import { Button } from "@/components/ui/button"

<Button variant="default" size="default">Save</Button>
```

`variant` and `size` are independent props. Both default to `"default"`. Both are typed by `VariantProps<typeof buttonVariants>` so autocomplete reflects exactly the keys in the local `buttonVariants` cva definition.

### Variant decision table

| User intent | Variant | Reason |
|-------------|---------|--------|
| Primary call-to-action ("Save", "Submit", "Continue") | `default` | Filled primary background ; highest visual weight |
| Irreversible / destructive action ("Delete", "Discard", "Reset") | `destructive` | Red fill signals risk ; pairs with confirmation dialog |
| Secondary action next to a primary ("Cancel", "Skip", "Back") | `outline` or `secondary` | Lower weight than `default` ; `outline` reads as "alternate", `secondary` reads as "muted-primary" |
| Tertiary action inside a busy UI (toolbar, table row) | `ghost` | No background until hover ; minimal chrome |
| Action that semantically IS a link (navigation, "Forgot password?") | `link` | Reads as text-link ; pairs with `asChild` + `<Link>` |

ALWAYS pick variant by user intent, NEVER by visual mood. If the action is destructive, `variant="destructive"` is non-optional, even if the surrounding design feels muted.

### Size decision table

| User intent | Size | Notes |
|-------------|------|-------|
| Default form / dialog button | `default` | 36px height (v4) ; pairs with `Input` of identical height |
| Compact toolbar / table action | `sm` | 32px height ; gap-1.5 ; rounded-md |
| Hero / marketing CTA | `lg` | 40px height ; px-6 |
| Icon-only button (toolbar, dialog close, table action) | `icon` | square 36px (v4) ; mandates a single icon child |
| Tailwind-v4-only : xs / icon-xs / icon-sm / icon-lg | `xs`, `icon-xs`, `icon-sm`, `icon-lg` | Extra granularity introduced in v4 ; NEVER assume these exist in a v3 project |

ALWAYS use `size="icon"` when the visible content is a single icon. NEVER drop an icon into `size="default"` : padding will dominate and accessibility-target rules become awkward.

### Four invariants

1. ALWAYS import from the local alias `@/components/ui/button`. NEVER from `shadcn-ui` or `@shadcn/ui`. (See `shadcn-core-architecture` for the copy-not-install rationale.)
2. ALWAYS pass exactly ONE child element when `asChild` is true. NEVER two siblings ; Slot.Root crashes.
3. ALWAYS let `className` pass through unchanged. NEVER concatenate by hand : the Button source already calls `cn(buttonVariants({ variant, size, className }))`, which routes through twMerge so caller classes win conflicts.
4. ALWAYS treat Button as RSC-safe : the file has no `"use client"` directive in either v3 or v4 form. NEVER add one just because the file uses an `asChild` boolean.

## Decision Tree 1 : Which variant?

```
Q1. Is this the primary action of the surface (form, dialog, page hero)?
    yes -> variant="default"
    no  -> Q2

Q2. Is this action destructive, irreversible, or data-losing?
    yes -> variant="destructive"  (ALWAYS pair with a confirmation Dialog)
    no  -> Q3

Q3. Is this action a secondary peer of a primary button (Cancel / Back / Skip)?
    yes -> Q3a
    no  -> Q4

Q3a. Should the secondary read as "alternate path"  -> variant="outline"
     Should the secondary read as "muted primary"   -> variant="secondary"

Q4. Is this a tertiary action embedded inside a busy region
    (table row, toolbar, command palette, sidebar item)?
    yes -> variant="ghost"
    no  -> Q5

Q5. Is the action SEMANTICALLY a link (navigation, "Forgot password?",
    "Privacy policy", external URL)?
    yes -> variant="link" + asChild + <Link>/<a>
    no  -> reconsider : default or outline is almost certainly right
```

## Decision Tree 2 : `asChild` or no `asChild`?

```
Q1. Does the consumer need a NON-button element to be rendered
    (an <a>, a Next.js <Link>, a react-router <Link>, a custom
    forwardRef component)?
    no  -> NEVER use asChild ; just render <Button onClick={...}>
    yes -> Q2

Q2. Is the child a single element that supports ref-forwarding
    (Next.js <Link>, react-router <Link>, a native <a>, or a
    React.forwardRef'd custom component)?
    no  -> ALWAYS wrap the non-forwardRef child first ; raw <div onClick>
           swallows merged props. Or use <Button onClick={...}>.
    yes -> Q3

Q3. Is there EXACTLY one child element (no siblings, no text node
    next to an element)?
    no  -> ALWAYS collapse to one child ; Slot.Root throws on
           multiple children. Compose icon + label INSIDE the child.
    yes -> ALWAYS use <Button asChild><ChildElement>...</ChildElement></Button>
```

## Decision Tree 3 : Icon-button or labelled button with an icon?

```
Q1. Is the visible content ONLY an icon (no text)?
    yes -> size="icon" + lucide icon child + ALWAYS provide an
           accessible name via aria-label
    no  -> Q2

Q2. Is the button text-with-leading-or-trailing icon?
    yes -> size="default" (or sm/lg) ; drop the icon AND the label
           as siblings inside the Button ; the cva base layer
           includes `gap-2` and `[&_svg]:size-4` so the icon spaces
           and sizes automatically
    no  -> reconsider : you have neither an icon nor a label
```

## Patterns

### Pattern 1 : Variant catalogue (verbatim from the canonical source)

The six variant values, with verbatim classes from `apps/v4/registry/new-york-v4/ui/button.tsx` :

| Variant | Verbatim classes |
|---------|------------------|
| `default` | `bg-primary text-primary-foreground hover:bg-primary/90` |
| `destructive` | `bg-destructive text-white hover:bg-destructive/90 focus-visible:ring-destructive/20 dark:bg-destructive/60 dark:focus-visible:ring-destructive/40` |
| `outline` | `border bg-background shadow-xs hover:bg-accent hover:text-accent-foreground dark:border-input dark:bg-input/30 dark:hover:bg-input/50` |
| `secondary` | `bg-secondary text-secondary-foreground hover:bg-secondary/80` |
| `ghost` | `hover:bg-accent hover:text-accent-foreground dark:hover:bg-accent/50` |
| `link` | `text-primary underline-offset-4 hover:underline` |

ALWAYS reference these by `variant=` prop. NEVER hand-copy the class strings : if shadcn upgrades them via the registry, your hand-copy diverges silently.

### Pattern 2 : Size catalogue (verbatim, v4 only)

| Size | Verbatim classes | Use |
|------|------------------|-----|
| `default` | `h-9 px-4 py-2 has-[>svg]:px-3` | Default form / dialog button |
| `xs` | `h-6 gap-1 rounded-md px-2 text-xs has-[>svg]:px-1.5 [&_svg:not([class*='size-'])]:size-3` | Tiny inline action (v4 only) |
| `sm` | `h-8 gap-1.5 rounded-md px-3 has-[>svg]:px-2.5` | Compact toolbar / table action |
| `lg` | `h-10 rounded-md px-6 has-[>svg]:px-4` | Hero / marketing CTA |
| `icon` | `size-9` | Icon-only square button |
| `icon-xs` | `size-6 rounded-md [&_svg:not([class*='size-'])]:size-3` | Tiny icon-only (v4 only) |
| `icon-sm` | `size-8` | Small icon-only (v4 only) |
| `icon-lg` | `size-10` | Large icon-only (v4 only) |

For the legacy v3 size set, only `default / sm / lg / icon` exist, with different exact pixel values (see Pattern 5).

### Pattern 3 : `asChild` Slot pattern

The canonical (v4) Button source contains exactly this pivot :

```tsx
import { Slot } from "radix-ui"
// ...
function Button({ asChild = false, ...props }) {
  const Comp = asChild ? Slot.Root : "button"
  return <Comp data-slot="button" {...props} />
}
```

When `asChild` is true, the component RENDERS THE CHILD ELEMENT and merges its props (className, ref, event handlers, data attributes) onto that child. The result is a fully styled Button with a different underlying tag.

Canonical use : wrap a navigation primitive so the button styling lives on the link.

```tsx
// Next.js (App Router, v15+)
import Link from "next/link"

<Button asChild>
  <Link href="/login">Login</Link>
</Button>
```

```tsx
// react-router-dom (v6 or v7 / React Router v7 rebrand)
import { Link } from "react-router-dom"

<Button asChild variant="link">
  <Link to="/forgot-password">Forgot password?</Link>
</Button>
```

ALWAYS exactly ONE child element inside `<Button asChild>`. NEVER siblings ; Slot.Root throws. Composition of icon + label happens INSIDE the child :

```tsx
<Button asChild>
  <Link href="/docs">
    <BookOpen />
    Docs
  </Link>
</Button>
```

## Section : Tailwind v3 vs Tailwind v4 ring utility

shadcn's Button ships two different focus-ring incantations depending on the registry style and the Tailwind version of the consumer project. Both are taken verbatim from the official registry source ; treat them as a single divergence point and emit the correct one.

### v3 (default registry style, `apps/v4/public/r/styles/default/button.json`)

```
focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2
```

Uses the v3 ring API : `ring-2` (a 2px ring) + `ring-offset-2` (a 2px offset) + named `ring-ring` token. The Button component is a `React.forwardRef<HTMLButtonElement, ButtonProps>` and exports a `ButtonProps` interface.

### v4 (new-york-v4 registry style, `apps/v4/registry/new-york-v4/ui/button.tsx`)

```
outline-none focus-visible:border-ring focus-visible:ring-[3px] focus-visible:ring-ring/50
```

Uses the v4 ring API : `ring-[3px]` (a 3px ring via the arbitrary-value syntax), no `ring-offset`, a `border-ring` border-colour-shift on focus, and `ring-ring/50` (the ring colour at 50% opacity). The Button is now a plain function component (no `React.forwardRef`, React 19 forwards refs implicitly) typed as `React.ComponentProps<"button">`.

ALWAYS emit the variant that matches the consumer project's Tailwind version. To check : read `package.json` for `tailwindcss` (the major version), or read the consumer's `globals.css` for `@import "tailwindcss"` (v4) versus `@tailwind base / components / utilities` (v3). NEVER mix the two ring incantations in one file ; the v3 ring uses ring-offset, the v4 ring uses a border shift, and they layer poorly when combined.

## Section : Icon button + Loading button patterns

### Icon-only button (lucide-react icon)

```tsx
import { Plus } from "lucide-react"

<Button size="icon" aria-label="Add item">
  <Plus />
</Button>
```

The cva base layer includes `[&_svg]:pointer-events-none [&_svg]:shrink-0 [&_svg:not([class*='size-'])]:size-4`, so a lucide icon auto-sizes to 16px (size-4) without any explicit size prop. ALWAYS provide `aria-label` when the visible text is empty ; screen readers have no label otherwise.

### Loading button (spinner + disabled state)

```tsx
import { Loader2 } from "lucide-react"

function SubmitButton({ isPending }: { isPending: boolean }) {
  return (
    <Button disabled={isPending}>
      {isPending && <Loader2 className="animate-spin" />}
      {isPending ? "Saving..." : "Save"}
    </Button>
  )
}
```

The cva base layer also includes `disabled:pointer-events-none disabled:opacity-50`, so the visual disabled state is automatic. NEVER add a manual `cursor-not-allowed` class on top : the existing `disabled:pointer-events-none` removes hover and pointer events entirely, and Tailwind v4 sets `cursor: default` on buttons by default anyway (see `shadcn-errors-tailwind-v3-v4-migration` for cursor-pointer-on-button workarounds).

## Section : RSC compatibility

Button is RSC-safe in both v3 and v4 form. The component file does NOT carry a `"use client"` directive, and it has no client-only React features (no `useState`, no `useEffect`, no DOM refs that the consumer's caller would need to use during render). ALWAYS import `Button` directly into a Server Component without wrapping it in a `"use client"` boundary. The button receives event handlers as props from the calling component (so the CALLER may need `"use client"` if the handler closes over client state), but Button itself does not.

NEVER add `"use client"` to `components/ui/button.tsx`. Doing so unnecessarily marks every importing module as a client module and breaks the RSC bundle split documented in `shadcn-impl-rsc-vs-client-boundaries`.

## Anti-patterns (summary ; full catalogue in references/anti-patterns.md)

NEVER do these :

1. NEVER pass multiple children to `<Button asChild>` : Slot.Root accepts exactly one child and throws "React.Children.only expected to receive a single React element child" otherwise. Move composition INSIDE the child.
2. NEVER use `asChild` with a child that ignores forwarded refs and props (raw `<input type="button">`, an unwrapped function component without forwardRef on v3, or a component that forgets to spread `...props`). The styling silently fails to attach.
3. NEVER drop an icon into `size="default"` and expect it to look right. ALWAYS use `size="icon"` for icon-only buttons.
4. NEVER override `className` by string concatenation : the cva machinery routes through `cn(buttonVariants({ variant, size, className }))`, which uses twMerge to resolve conflicts. Hand-concatenating defeats conflict resolution and your overrides lose silently.
5. NEVER replace `<Button asChild><Link href="/x"/></Button>` with `<Button onClick={() => router.push("/x")}>`. You lose Next.js link prefetch, middle-click open-in-new-tab, keyboard activation semantics, and screen-reader role.
6. NEVER toggle both `disabled` and a manual hover variant class on the same render : the cva base layer already disables pointer events when `disabled` is true. Manual overrides collide with the base disabled state.

See `references/anti-patterns.md` for the full WHY and FIX of each.

## Companion Skills

- [shadcn-syntax-variant-cva](../shadcn-syntax-variant-cva/SKILL.md) : the cva variant API, defaultVariants, compoundVariants, VariantProps in depth. Read BEFORE extending buttonVariants with a custom variant.
- [shadcn-core-architecture](../../shadcn-core/shadcn-core-architecture/SKILL.md) : the copy-not-install paradigm and the "you own the source" doctrine that lets you extend `button.tsx` freely.
- [shadcn-core-stack](../../shadcn-core/shadcn-core-stack/SKILL.md) : Radix Slot, cva, tailwind-merge, clsx composition map ; explains exactly what Slot.Root does under the hood.
- [shadcn-errors-radix-controlled](../../shadcn-errors/shadcn-errors-radix-controlled/SKILL.md) : controlled-state failures across Radix primitives ; includes the `asChild` multi-children crash and the non-forwardRef-child swallowed-props pattern.
- [shadcn-impl-rsc-vs-client-boundaries](../../shadcn-impl/shadcn-impl-rsc-vs-client-boundaries/SKILL.md) : why Button stays server-safe and what triggers the `"use client"` requirement elsewhere.
- [shadcn-errors-tailwind-v3-v4-migration](../../shadcn-errors/shadcn-errors-tailwind-v3-v4-migration/SKILL.md) : the ring-2 -> ring-[3px] shift and the cursor-default-on-buttons gotcha.

## Reference Links

- [references/methods.md](references/methods.md) : Button component signature, buttonVariants cva instance, VariantProps typing, full variant + size enumeration verbatim.
- [references/examples.md](references/examples.md) : ten canonical recipes including Next.js Link, react-router Link, icon-only, loading, custom warning variant, v3 vs v4 ring annotations.
- [references/anti-patterns.md](references/anti-patterns.md) : six anti-patterns with WHY and FIX.

## Sources

All claims in this skill trace to URLs in `SOURCES.md` :

- https://ui.shadcn.com/docs/components/radix/button (canonical Button docs page, variant + size prop tables)
- https://github.com/shadcn-ui/ui (apps/v4/registry/new-york-v4/ui/button.tsx, verbatim v4 source ; apps/v4/public/r/styles/default/button.json, verbatim v3 source)
- https://www.radix-ui.com/primitives/docs/utilities/slot (Slot.Root API + Slottable for multi-child cases)
- https://cva.style/docs (cva function signature, defaultVariants, VariantProps helper)
- https://github.com/dcastil/tailwind-merge (twMerge conflict resolution that powers `cn()`)
- https://github.com/shadcn-ui/ui/issues/6843 (Tailwind v4 cursor-default on buttons, 126 reactions, open)

Verified 2026-05-19.
