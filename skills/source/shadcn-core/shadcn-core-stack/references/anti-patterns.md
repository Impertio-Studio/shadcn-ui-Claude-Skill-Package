# shadcn ui Stack: Anti-Patterns

Real failure modes mined from GitHub issues and the shadcn ui docs.

## AP-1: Raw String Concatenation Instead of `cn()`

```tsx
// BROKEN
function Button({ className, ...props }) {
  return (
    <button
      className={`inline-flex rounded-md px-4 py-2 bg-primary ${className}`}
      {...props}
    />
  )
}

// Usage : caller wants to override padding
<Button className="px-8" />
// Resulting DOM : class="inline-flex rounded-md px-4 py-2 bg-primary px-8"
// Both px-4 and px-8 land in the DOM. Tailwind's source-order rules decide which wins,
// not the developer's intent.
```

### Why this fails

Tailwind generates utilities in a deterministic source order. When two utilities target the same property (`px-4` and `px-8`), the one Tailwind emitted last in the compiled CSS wins, REGARDLESS of the order they appear in the HTML `class` attribute. Without conflict resolution, the override silently fails or succeeds depending on emission order.

### Fix : ALWAYS use `cn()`

```tsx
import { cn } from "@/lib/utils"

function Button({ className, ...props }) {
  return (
    <button
      className={cn("inline-flex rounded-md px-4 py-2 bg-primary", className)}
      {...props}
    />
  )
}

// Now : cn("...px-4...", "px-8") => twMerge drops px-4. Final DOM : "...px-8..."
```

Source : https://github.com/dcastil/tailwind-merge (conflict resolution rationale).

## AP-2: Missing `tailwind-merge` (Only `clsx`)

```tsx
// BROKEN
import { clsx } from "clsx"

function cn(...inputs) {
  return clsx(inputs)
}

// Same problem as AP-1 : conditional composition works, conflict resolution does NOT.
```

### Why this fails

`clsx` joins strings with whitespace and drops falsy values. It does NOT understand Tailwind utility groups, so `clsx("px-4", "px-8")` returns `"px-4 px-8"` (both kept). Without `twMerge`, every component override leaves zombie utilities in the class list.

### Fix : ALWAYS combine clsx + twMerge

```tsx
import { clsx, type ClassValue } from "clsx"
import { twMerge } from "tailwind-merge"

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs))
}
```

## AP-3: Bypassing the shadcn-Wrapped Component

```tsx
// BROKEN : importing Radix directly in app code
import * as DialogPrimitive from "@radix-ui/react-dialog"

function MyDialog() {
  return (
    <DialogPrimitive.Root>
      <DialogPrimitive.Trigger>Open</DialogPrimitive.Trigger>
      <DialogPrimitive.Content>
        Hello
      </DialogPrimitive.Content>
    </DialogPrimitive.Root>
  )
}
```

### Why this fails

Radix primitives ship UNSTYLED on purpose. The shadcn-wrapped `@/components/ui/dialog` provides ALL the visible styling : overlay, positioning, animations, close button, focus rings. Bypassing it gives you a working but invisible Dialog with no overlay, no centering, no close button.

### Fix : ALWAYS import from `@/components/ui/...`

```tsx
import {
  Dialog, DialogTrigger, DialogContent, DialogTitle, DialogDescription,
} from "@/components/ui/dialog"

function MyDialog() {
  return (
    <Dialog>
      <DialogTrigger>Open</DialogTrigger>
      <DialogContent>
        <DialogTitle>Title</DialogTitle>
        <DialogDescription>Description</DialogDescription>
      </DialogContent>
    </Dialog>
  )
}
```

The shadcn-copied `dialog.tsx` is YOURS to modify. Want a different overlay color? Edit `components/ui/dialog.tsx` once and every consumer picks it up. NEVER re-wrap Radix in your own ad-hoc components when shadcn ships the canonical wrapper.

Source : https://ui.shadcn.com/docs (open-code + composition pillars).

## AP-4: Wrong cva Variant Ordering with `compoundVariants`

```tsx
// BROKEN : compoundVariants references a variant that does not exist
const buttonVariants = cva("base", {
  variants: {
    variant: { default: "...", destructive: "..." },
    // size variant DEFINED AFTER compoundVariants references it
  },
  compoundVariants: [
    { variant: "destructive", size: "lg", class: "border-2" }, // size? undefined
  ],
})
```

### Why this fails

Within a single `cva()` call this specific order issue is type-safe in modern cva, but a more common variant of this bug is real :

```tsx
// Real bug : compoundVariants list order matters when MULTIPLE rules match
const card = cva("base", {
  variants: {
    intent: { primary: "...", danger: "..." },
    size: { sm: "...", lg: "..." },
  },
  compoundVariants: [
    { intent: "danger", class: "ring-1 ring-destructive" },        // rule A
    { intent: "danger", size: "lg", class: "ring-2" },             // rule B
  ],
})
// card({ intent: "danger", size: "lg" }) appends BOTH rule A and rule B
// => "...base... ring-1 ring-destructive ring-2"
// Without twMerge, BOTH ring-1 and ring-2 stay. The intent of "ring-2 overrides ring-1"
// only resolves after cn() passes the output through twMerge.
```

### Fix : ALWAYS wrap the cva output in `cn()`

```tsx
<button className={cn(card({ intent: "danger", size: "lg" }), className)} />
// twMerge resolves ring-1 vs ring-2 to ring-2.
```

ALWAYS pass the cva return through `cn()` ; never use the raw string directly in `className=`.

Source : https://cva.style/docs (compoundVariants semantics).

## AP-5: Mixing `radix-ui` Unified With `@radix-ui/react-*` Per-Component

```tsx
// BROKEN : two imports for the same primitive
import { Dialog } from "radix-ui"
import * as DialogPrimitive from "@radix-ui/react-dialog"
```

### Why this fails

The unified `radix-ui` package and the per-component `@radix-ui/react-*` packages are SEPARATE NPM installs. Each ships its own module instance. Components from different instances do NOT share React context, so a `Dialog.Root` from `radix-ui` and a `Dialog.Content` from `@radix-ui/react-dialog` will NOT find each other (the Content thinks no Dialog Root provider exists).

Symptoms : "Cannot read properties of null" inside the Radix internals, or a Dialog that opens but never traps focus, or a Dropdown that fires `onOpenChange` but renders no Content.

### Fix : pick ONE package surface and stick to it

For new projects (post Feb 2026) :

```bash
npm install radix-ui
```

```tsx
import { Dialog, DropdownMenu, Popover, Tooltip } from "radix-ui"
```

For existing projects already on `@radix-ui/react-*` :

```bash
npm install @radix-ui/react-dialog @radix-ui/react-dropdown-menu @radix-ui/react-popover
```

```tsx
import * as DialogPrimitive from "@radix-ui/react-dialog"
import * as DropdownMenuPrimitive from "@radix-ui/react-dropdown-menu"
```

A wholesale migration `@radix-ui/react-* -> radix-ui` is straightforward (rename imports, drop separate packages) but MUST happen atomically.

Source : https://www.radix-ui.com/primitives/docs/overview/getting-started (unified package introduction).

## AP-6: `tailwind-merge` Version Mismatch With Tailwind Version

```json
// package.json
{
  "dependencies": {
    "tailwindcss": "^4.1.0",
    "tailwind-merge": "^2.6.0"
  }
}
```

### Why this fails

`tailwind-merge@2.x` was built against Tailwind v3's utility surface. Tailwind v4 introduced (a) new utilities (`shadow-xs`, `inset-shadow-*`, `inset-ring-*`, `bg-linear-*`, `field-sizing-content`, `rotate-x-*`, `perspective-*`), (b) renamed scale steps (every shadow/blur/radius shifted by one), and (c) the parens arbitrary-variable syntax (`bg-(--brand)`). v2.x of tailwind-merge does NOT know about any of these and will either keep duplicate utilities silently or drop legitimate ones.

### Fix : match versions

| Tailwind | tailwind-merge |
|----------|----------------|
| v3.4.x | `^2.6.0` |
| v4.0.x to v4.3.x | `^3.0.0` |

```bash
# Tailwind v4 project
npm install tailwind-merge@^3
```

Source : https://github.com/dcastil/tailwind-merge (Tailwind compatibility table).

## AP-7: Calling `cva()` Inside the Component Body

```tsx
// BROKEN : cva() runs on every render
function Button({ variant, size, className, ...props }: ButtonProps) {
  const buttonVariants = cva("base", {
    variants: {
      variant: { default: "...", destructive: "..." },
      size: { sm: "...", lg: "..." },
    },
  })
  return <button className={cn(buttonVariants({ variant, size }), className)} {...props} />
}
```

### Why this fails

`cva()` returns a function. Calling `cva()` inside the component body recreates that function on every render, throwing away cva's internal lookup-cache optimisations and wasting allocations. The returned `VariantProps` type also gets re-derived per call, defeating IDE caching.

### Fix : ALWAYS define cva at module scope

```tsx
// At the top of the file, outside any component
const buttonVariants = cva("base", { variants: { /* ... */ } })

function Button({ variant, size, className, ...props }: ButtonProps) {
  return <button className={cn(buttonVariants({ variant, size }), className)} {...props} />
}
```

## AP-8: Passing `className` BEFORE Variants to `cn()`

```tsx
// BROKEN : caller's className comes first, gets overridden by variants
function Button({ className, variant, size, ...props }: ButtonProps) {
  return (
    <button
      className={cn(className, buttonVariants({ variant, size }))}
      {...props}
    />
  )
}

<Button className="rounded-full" />
// Caller wanted rounded-full ; the variant's rounded-md OVERRIDES it.
```

### Why this fails

`twMerge` resolves LEFT TO RIGHT : later utilities win on conflicts. If `className` is the first argument, the variant strings appearing after it OVERRIDE the caller's customisation.

### Fix : `className` LAST in `cn()`

```tsx
className={cn(buttonVariants({ variant, size }), className)}
// caller wins on conflicts as expected
```

Source : https://github.com/shadcn-ui/ui (every shadcn-generated component places caller `className` last).

## AP-9: Assuming `defaultVariants` Applies to Explicit Falsy Values

```tsx
const buttonVariants = cva("base", {
  variants: { variant: { default: "...", destructive: "..." } },
  defaultVariants: { variant: "default" },
})

<Button variant={undefined as "default" | "destructive" | undefined} />
// => default variant applied. Good.

<Button variant={(false ? "destructive" : "") as any} />
// => NOTHING applied. defaultVariants does NOT trigger on "" empty-string.

<Button variant={null} />
// => null EXPLICITLY skips the default. defaultVariants does NOT trigger on null.
```

### Why this fails

cva's `defaultVariants` apply when the prop is `undefined`. An explicit empty string or `null` is treated as "caller intentionally chose no variant". This trips users coming from React conditional-render habits where `false`/`""` are treated as "absent".

### Fix : ALWAYS guard with `undefined`

```tsx
const safeVariant = (props.variant ?? undefined)
<Button variant={safeVariant} />
```

Or use cva's `defaultVariants` with explicit `null` opt-out documentation.

Source : https://cva.style/docs (defaultVariants behaviour).

## AP-10: Treating shadcn ui as a Runtime Dependency

```json
// BROKEN package.json
{
  "dependencies": {
    "shadcn-ui": "^4.0.0",
    "@shadcn/ui": "*"
  }
}
```

### Why this fails

There is no `shadcn-ui` runtime package. The shadcn ui CLI (`shadcn` binary) is invoked via `pnpm dlx shadcn@latest add button` (or `npx shadcn@latest add button`), copies the source into your project, and disappears. Nothing remains as a runtime dependency.

Including `shadcn-ui` or `@shadcn/ui` in `dependencies` installs a non-existent package, breaks `npm install`, and signals fundamental misunderstanding of the ownership model.

### Fix : use the CLI, not `dependencies`

```bash
pnpm dlx shadcn@latest init
pnpm dlx shadcn@latest add button card dialog
```

The actual runtime dependencies added by these commands :

```json
{
  "dependencies": {
    "@radix-ui/react-dialog": "^1.x", // or "radix-ui": "^x.y"
    "class-variance-authority": "^0.7 or ^1.0-beta",
    "clsx": "^2.x",
    "tailwind-merge": "^3.x", // (^2.x for Tailwind v3)
    "lucide-react": "^x.y",
    "tailwindcss": "^3.4 or ^4.0"
  }
}
```

Source : https://ui.shadcn.com/docs/installation and https://ui.shadcn.com/docs (open-code distribution model).
