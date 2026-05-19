# Button : Anti-Patterns

Six canonical Button-level failures, each with WHY it fails and the exact FIX. All six are observed in real shadcn ui issues, Radix Slot issues, or repeated AI-generated code patterns.

---

## AP-1 : `asChild` with multiple children

### Wrong

```tsx
import Link from "next/link"
import { ArrowRight } from "lucide-react"
import { Button } from "@/components/ui/button"

<Button asChild>
  <Link href="/next">Continue</Link>
  <ArrowRight />
</Button>
```

### Why it fails

`Slot.Root` (the component that powers `asChild`) calls `React.Children.only(children)` internally. With two children (the Link and the ArrowRight), React throws :

```
Error: React.Children.only expected to receive a single React element child.
```

The error halts rendering of the entire subtree. In dev mode it produces a red error overlay ; in production it crashes the closest error boundary.

### Fix

Move composition INSIDE the single child element :

```tsx
<Button asChild>
  <Link href="/next">
    Continue
    <ArrowRight />
  </Link>
</Button>
```

Now Slot.Root receives exactly one child (the Link). The Link element wraps both the text and the icon, and the Button's cva base layer applies its `[&_svg]` rules to the nested SVG correctly. ALWAYS collapse to one child when `asChild` is true ; NEVER pass siblings.

If the API truly needs multiple slots, switch to `Slot.Slottable` (see `https://www.radix-ui.com/primitives/docs/utilities/slot`) and rework the Button to support multi-slot composition. That is out of scope for the default shadcn Button.

---

## AP-2 : `asChild` with a non-ref-forwarding child

### Wrong

```tsx
import { Button } from "@/components/ui/button"

// Custom component that ignores forwarded props.
function MyClickArea({ children }: { children: React.ReactNode }) {
  return <div>{children}</div>
}

<Button asChild>
  <MyClickArea>Click me</MyClickArea>
</Button>
```

```tsx
// Or : asChild on a raw <input type="button"> ; while <input> IS a native
// intrinsic and CAN accept refs, it is a SELF-CLOSING element and cannot
// have children, so the Button's child text node has nowhere to land.
<Button asChild>
  <input type="button" value="Click" />
</Button>
```

### Why it fails

Two distinct sub-failures, both observed :

1. `MyClickArea` does not spread `...props` onto the underlying `<div>`. Slot.Root clones the child and merges Button's className, data-attributes, and event handlers onto it, but `MyClickArea` discards every prop it does not name. Result : the Button is invisible (no classes), unstyled, and inert (no onClick).
2. `<input type="button">` is self-closing. The Button's "Click me" children become Slot's children, which Slot then tries to render through the input element, but `<input>` cannot have children. React emits the "input is a void element tag and must neither have `children` nor use `dangerouslySetInnerHTML`" warning.

### Fix

Either :

- ALWAYS use a forwardRef-compatible child that spreads `...props` (native intrinsics `a`, `button`, `span`, `div` ; Next.js `Link` ; react-router `Link` ; any function component under React 19 ; any `React.forwardRef`'d v3 component).
- Or do NOT use `asChild` and render a real `<Button onClick={...}>` instead :

```tsx
<Button onClick={() => doThing()}>Click me</Button>
```

If the requirement is a button-shaped form input, use `<Button type="submit">` ; the shadcn Button is already a `<button>` element and supports `type="submit"` natively. NEVER reach for `<input type="button">` inside `asChild`.

---

## AP-3 : Icon dropped into `size="default"` without `size="icon"`

### Wrong

```tsx
import { Trash2 } from "lucide-react"
import { Button } from "@/components/ui/button"

// Intended to be an icon-only delete button.
<Button variant="ghost">
  <Trash2 />
</Button>
```

### Why it fails

`size="default"` produces a rectangular button with `h-9 px-4 py-2` (v4) or `h-10 px-4 py-2` (v3). With only a 16px icon inside, the padding dominates : the rendered button is roughly 60px wide for a 16px icon, leaving a large empty area on both sides of the glyph. Visually it reads as "broken button" or "the label is missing". Accessibility tooling also flags the empty accessible name unless an `aria-label` is set, which AP-3's pattern usually forgets too.

### Fix

ALWAYS pair an icon-only Button with `size="icon"` AND an `aria-label` :

```tsx
<Button size="icon" variant="ghost" aria-label="Delete item">
  <Trash2 />
</Button>
```

`size="icon"` resolves to `size-9` (v4) or `h-10 w-10` (v3), producing a square button sized to the icon plus consistent padding. The icon size is handled automatically by the base-layer `[&_svg]` selector ; NEVER add `className="size-4"` to the icon manually. NEVER omit `aria-label` ; the visible content is empty for screen readers.

For an icon-WITH-label button, use `size="default"` and drop both the icon and the text as siblings inside the Button :

```tsx
<Button variant="ghost">
  <Trash2 />
  Delete
</Button>
```

The cva base layer's `gap-2` produces the correct icon-to-text spacing automatically.

---

## AP-4 : Manually concatenating `className` instead of routing through `cn()`

### Wrong

```tsx
import { Button } from "@/components/ui/button"

const isActive = true

// Raw template-string concat.
<Button className={"bg-red-500 " + (isActive ? "ring-2" : "")}>
  Click
</Button>
```

### Why it fails

Two sub-failures :

1. The Button source already calls `cn(buttonVariants({ variant, size, className }))` internally. When the caller passes a raw string, `cn()` does merge it via `twMerge`, but the caller's INTENT to override `bg-primary` with `bg-red-500` only works because `bg-red-500` happens to appear later in the merge order. The mental model "the variant's class wins" is wrong, but so is "my raw class wins" : the actual rule is "the rightmost utility for any given Tailwind property family wins, per twMerge". Hand-concatenation makes this opaque.
2. The conditional `(isActive ? "ring-2" : "")` produces an empty string when false, which is harmless on its own but compounds badly when chained with more conditionals : `"a " + (b ? "b" : "") + " " + (c ? "c" : "")` produces double spaces and is hard to read.

### Fix

ALWAYS route conditional className composition through `cn()` directly :

```tsx
import { cn } from "@/lib/utils"
import { Button } from "@/components/ui/button"

<Button className={cn("bg-red-500", isActive && "ring-2")}>
  Click
</Button>
```

`cn()` (= `twMerge(clsx(...))`) handles three jobs at once : it joins arrays, filters falsy values from conditionals, and resolves Tailwind conflicts via twMerge. The caller's class still goes INTO the Button's own `cn(...)` call, but now both calls use the same merge primitive and the composition is predictable.

For deep overrides, prefer extending the cva definition (Recipe 9 in `references/examples.md`) over piling more classes onto every call site. The "Open Code" pillar permits editing `components/ui/button.tsx` directly.

---

## AP-5 : `<Button onClick={() => router.push()}>` instead of `<Button asChild><Link/></Button>`

### Wrong

```tsx
"use client"

import { useRouter } from "next/navigation"
import { Button } from "@/components/ui/button"

export function LoginButton() {
  const router = useRouter()
  return <Button onClick={() => router.push("/login")}>Login</Button>
}
```

### Why it fails

Four concrete regressions compared to `<Button asChild><Link href="/login">Login</Link></Button>` :

1. **No prefetch.** Next.js Link prefetches the destination on viewport intersection (default for App Router). `router.push()` does no prefetch ; the destination loads on click only.
2. **Middle-click does nothing.** Native anchors open in a new tab on middle-click or `Cmd+Click` / `Ctrl+Click`. A `<button onClick={router.push}>` is not an anchor, so the OS-level shortcuts are absent.
3. **Keyboard semantics shift.** A `<button>` activates on `Enter` and `Space` ; an `<a>` activates on `Enter` only. Screen readers announce them differently ("button" vs "link"). For navigation, "link" is the correct role.
4. **Server-side rendering crawls poorly.** Search-engine crawlers and link-graph tools read the `<a href>` ; they do not execute the onClick handler. A button-onClick-pushed route is invisible to crawlers.

Also : `useRouter` requires `"use client"`, forcing the entire enclosing module into the client bundle for no reason.

### Fix

ALWAYS use `asChild` + the framework's Link primitive :

```tsx
import Link from "next/link"
import { Button } from "@/components/ui/button"

export function LoginButton() {
  return (
    <Button asChild>
      <Link href="/login">Login</Link>
    </Button>
  )
}
```

This file is now an RSC. No `"use client"` needed. Next.js Link handles prefetch, middle-click, keyboard semantics, and crawler-visibility automatically. The Button's variant + size classes still apply via Slot.Root's prop merge.

The ONLY reason to use a button-with-onClick-router-push pattern is if the navigation is a side effect of a non-link action (e.g., "save the form, THEN navigate"). For that case, fire `router.push` from the form's `onSubmit` handler, not from a Button's `onClick`.

---

## AP-6 : Mixing `disabled` with a manual hover class override

### Wrong

```tsx
import { Button } from "@/components/ui/button"

<Button
  disabled={isPending}
  className="hover:bg-blue-500 cursor-not-allowed"
>
  Save
</Button>
```

### Why it fails

The cva base layer already includes `disabled:pointer-events-none disabled:opacity-50`. When `disabled` is true :

1. `pointer-events-none` REMOVES the hover state entirely : `hover:bg-blue-500` cannot fire because pointer events are turned off. The class is dead code.
2. `cursor-not-allowed` does nothing under Tailwind v4 because v4 sets `cursor: default` on `<button>` globally AND `pointer-events-none` overrides cursor styling anyway. Under v3, `cursor-not-allowed` does paint, but the disabled-opacity layer also paints, and the two read as "double-emphasised disabled" which is visually noisy.
3. Tailwind v4's cursor-on-buttons behaviour is a known and contentious change : shadcn-ui issue #6843 (126 reactions, open) tracks the broader pattern. ALWAYS handle the cursor preference at the project level via `globals.css`, NEVER per-Button.

When `disabled` is false :

1. `hover:bg-blue-500` paints on hover, but the variant's own `hover:bg-primary/90` already painted in the cascade. Whichever utility comes later in the merge wins ; the result is order-dependent and brittle.

### Fix

Trust the base layer. ALWAYS rely on the cva-supplied disabled state ; NEVER add manual `cursor-not-allowed` or hover overrides on top :

```tsx
<Button disabled={isPending}>
  {isPending && <Loader2 className="animate-spin" />}
  {isPending ? "Saving..." : "Save"}
</Button>
```

If a hover behaviour DIFFERENT from the variant is genuinely required, extend `buttonVariants` with a new variant (see Recipe 9 in `references/examples.md`) so the hover state lives in the cva definition alongside the rest of the variant. NEVER per-call-site override.

If the project requires `cursor: pointer` on buttons under Tailwind v4, set it once globally in `globals.css` :

```css
@layer base {
  button:not(:disabled),
  [role="button"]:not(:disabled) {
    cursor: pointer;
  }
}
```

See `shadcn-errors-tailwind-v3-v4-migration` for the full discussion of the cursor-default-on-buttons shift.

---

## Cross-references

- `references/methods.md` : the verbatim cva instance and Slot.Root resolution rules that make these anti-patterns observable.
- `references/examples.md` : the correct patterns for icon, loading, Link, and custom variant cases.
- `shadcn-errors-radix-controlled` : deeper coverage of the `asChild` + multi-child + non-forwardRef-child failures across Radix primitives, not just Button.
- `shadcn-errors-tailwind-v3-v4-migration` : the cursor-default-on-buttons issue, the HSL-to-oklch shift, and the ring-2 -> ring-[3px] focus-ring change.

Verified 2026-05-19.
