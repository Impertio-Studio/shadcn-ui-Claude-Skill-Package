# Examples : WRONG vs RIGHT snippets

Reference material for `shadcn-errors-styling-conflicts`. Every example is a real, common shadcn project mistake matched to its deterministic fix.

---

## 1. Template-string concatenation leaks duplicates

The single most common class-merging bug. The template-string version compiles, runs, and produces visibly broken styling because both classes reach the DOM and the second one only sometimes wins (depends on Tailwind's compiled CSS order, not on the developer's intent).

```tsx
// WRONG
function Card({ compact, className }: { compact?: boolean; className?: string }) {
  return (
    <div className={"p-4 rounded-md " + (compact ? "p-2" : "") + " " + className} />
  )
}
// Rendered DOM when compact=true and consumer passes "p-6":
//   <div class="p-4 rounded-md p-2  p-6" />
// All three p-* classes reach CSS. Visual padding is determined by source order, not intent.

// RIGHT
import { cn } from "@/lib/utils"
function Card({ compact, className }: { compact?: boolean; className?: string }) {
  return (
    <div className={cn("p-4 rounded-md", compact && "p-2", className)} />
  )
}
// Rendered DOM when compact=true and consumer passes "p-6":
//   <div class="rounded-md p-6" />
// twMerge resolved p-4 -> p-2 -> p-6 ; only the last one survives.
```

---

## 2. cva variant override via consumer className

A consumer passes `className="px-8"` to override a button's variant padding. The override only wins when cn() is applied at the call site AND the consumer className is the final argument.

```tsx
const buttonVariants = cva("rounded-md font-medium", {
  variants: { size: { sm: "h-8 px-3", md: "h-9 px-4" } },
  defaultVariants: { size: "md" },
})

// WRONG : consumer className applied BEFORE variant ; variant wins.
function Button({ size, className }: ButtonProps) {
  return <button className={cn(className, buttonVariants({ size }))} />
}
// <Button size="sm" className="px-8" /> renders class="px-3 ..."  ← px-8 lost

// RIGHT : consumer className applied LAST.
function Button({ size, className }: ButtonProps) {
  return <button className={cn(buttonVariants({ size }), className)} />
}
// <Button size="sm" className="px-8" /> renders class="px-8 h-8 rounded-md font-medium"
```

This pattern is enshrined in every shadcn ui component file (see `components/ui/button.tsx` after `shadcn add button`).

---

## 3. Arbitrary VALUE vs arbitrary PROPERTY

tailwind-merge intentionally does NOT resolve arbitrary properties (the bracketed `[prop:value]` form) against their utility equivalents. Arbitrary values (`p-[1rem]`) DO resolve.

```tsx
// WRONG : arbitrary property + utility ; both reach DOM, neither wins predictably.
<div className={cn("[padding:1rem]", "p-4")} />
// Output: "[padding:1rem] p-4"  ← both kept on purpose, for bundle-size reasons.

// RIGHT (option A) : use arbitrary VALUE form, which merges against utility families.
<div className={cn("p-[1rem]", "p-4")} />
// Output: "p-4"

// RIGHT (option B) : pick one source of truth ; do not mix arbitrary-property and utility for the same CSS property.
<div className={cn("p-4")} />
```

Hex-color arbitraries follow the same rule : `bg-[#fff]` is an arbitrary VALUE (it targets the `bg-*` utility group), so it merges with `bg-white`:

```tsx
cn("bg-[#fff]", "bg-white")
// Output: "bg-white"  ← last wins ; both belong to the same class group.
```

---

## 4. !important precedence is namespace-separated

A common assumption is that `!` always wins. tailwind-merge tracks important utilities in a SEPARATE conflict namespace. The order is:

- important class can override another important class of the same family (last-wins inside the important namespace)
- non-important class CANNOT override an important class of the same family
- important class is NOT used as a conflict target for non-important classes either ; both reach DOM

```tsx
// Tailwind v4 syntax (suffix !)
cn("bg-red-500!", "bg-blue-500")     // "bg-red-500! bg-blue-500"   ← both kept ; plain cannot displace important
cn("bg-red-500!", "bg-blue-500!")    // "bg-blue-500!"              ← important vs important : last wins
cn("bg-red-500", "bg-blue-500!")     // "bg-red-500 bg-blue-500!"   ← both kept
cn("bg-red-500", "bg-blue-500")      // "bg-blue-500"               ← plain vs plain : last wins

// Tailwind v3 syntax (prefix !)
cn("!bg-red-500", "bg-blue-500")     // "!bg-red-500 bg-blue-500"   ← same rule, different syntax
```

The fix when an `!important` "is not winning" is almost always to make BOTH sides important or NEITHER side important.

---

## 5. cn() inside cva is a no-op (anti-pattern)

cva already runs `clsx` internally on the value of each variant. Wrapping that value in `cn()` adds a redundant call AND runs `twMerge` against an arrangement of classes that cva has not yet finalized. The result is wasted work and harder-to-read code.

```ts
// WRONG : cn() inside cva
const button = cva("rounded-md", {
  variants: {
    tone: {
      brand: cn("bg-primary", "text-primary-foreground"),  // no benefit ; cva will run clsx anyway
    },
  },
})

// RIGHT : plain strings or string arrays inside cva ; let the call site run cn() ONCE.
const button = cva("rounded-md", {
  variants: {
    tone: {
      brand: "bg-primary text-primary-foreground",
    },
  },
})

// The call site does the merging that actually matters
className={cn(button({ tone: "brand" }), className)}
```

---

## 6. Tailwind v3 to v4 class-semantics shifts

These are not tailwind-merge bugs. They are version-tagged class identities. Mixing them in one project after an upgrade silently produces double-up classes that twMerge cannot reconcile because they target different design tokens.

| What you wrote | v3 meaning | v4 meaning | Outcome if mixed |
|---|---|---|---|
| `ring` | 3px focus ring | 1px focus ring | Visual size flip after upgrade |
| `ring-2` | 2px ring | 2px ring (unchanged value, but `ring` alone is now 1px) | Confusion : `ring ring-2` no longer redundant |
| `!bg-red-500` | important red bg | not valid syntax in v4 (use `bg-red-500!`) | Important does not apply ; appears in DOM but ignored |
| `bg-red-500!` | not v3 syntax (warning) | important red bg | v3 build fails or ignores |
| `w-4 h-4` | 16x16 square | same effect, but `size-4` is preferred | OK ; tailwind-merge recognises both shapes |
| `size-4` | not valid in v3 (no `size-*` utility) | 16x16 square | v3 ignores it ; layout breaks |
| `space-x-4` | gap via margins | gap via gap-* (preferred) ; space-* still works but generated CSS differs | Subtle gap-vs-margin differences in edge cases |
| `bg-[hsl(var(--background))]` | the only way to consume a CSS var | works, but `bg-background` is the v4 way | Two valid forms ; pick one per project |
| `text-balance` | requires plugin | built-in | Plugin import becomes redundant in v4 |
| `tailwindcss-animate` import | required for Accordion etc. | replaced by `tw-animate-css` CSS import | Animations stop working after upgrade if old plugin is left in |

When a class-merge bug appears after a Tailwind upgrade, check this table BEFORE blaming tailwind-merge. See `shadcn-errors-tailwind-v3-v4-migration` for the full migration walk-through.

---

## 7. `defaultVariants` does not fire on empty string

This is a cva trap, not a tailwind-merge trap, but it surfaces as "the variant class is wrong."

```ts
const button = cva("rounded", {
  variants: { tone: { brand: "bg-primary", ghost: "bg-transparent" } },
  defaultVariants: { tone: "brand" },
})

button({})                  // → "rounded bg-primary"      ← default applied
button({ tone: undefined }) // → "rounded bg-primary"      ← default applied
button({ tone: "" })        // → "rounded"                 ← NO variant applied ; default does NOT fire
button({ tone: null as any })// → "rounded"                ← same trap
```

Coerce empty strings to `undefined` before passing to cva, or use TypeScript to forbid the empty-string case entirely.

---

## 8. End-to-end : a correct shadcn-style Button

The reference shape every shadcn project converges on. Note `cn()` runs ONCE at the JSX call site, cva runs internally, and consumer className is LAST.

```tsx
import * as React from "react"
import { cva, type VariantProps } from "class-variance-authority"
import { cn } from "@/lib/utils"

const buttonVariants = cva(
  "inline-flex items-center justify-center rounded-md text-sm font-medium transition-colors focus-visible:outline-none focus-visible:ring-1 focus-visible:ring-ring disabled:pointer-events-none disabled:opacity-50",
  {
    variants: {
      variant: {
        default: "bg-primary text-primary-foreground hover:bg-primary/90",
        destructive: "bg-destructive text-destructive-foreground hover:bg-destructive/90",
        outline: "border border-input bg-background hover:bg-accent hover:text-accent-foreground",
        ghost: "hover:bg-accent hover:text-accent-foreground",
      },
      size: {
        default: "h-9 px-4 py-2",
        sm: "h-8 rounded-md px-3 text-xs",
        lg: "h-10 rounded-md px-8",
        icon: "size-9",
      },
    },
    defaultVariants: { variant: "default", size: "default" },
  }
)

export interface ButtonProps
  extends React.ButtonHTMLAttributes<HTMLButtonElement>,
    VariantProps<typeof buttonVariants> {}

export function Button({ className, variant, size, ...props }: ButtonProps) {
  return (
    <button
      className={cn(buttonVariants({ variant, size }), className)}
      {...props}
    />
  )
}
```

Consumer overrides:

```tsx
<Button>Default</Button>                                       // variant=default, size=default
<Button variant="outline" size="sm">Outline Small</Button>
<Button className="bg-emerald-600 hover:bg-emerald-700">Brand</Button>
// twMerge drops bg-primary (variant), keeps bg-emerald-600 (consumer)
```

---

## 9. References

- https://github.com/dcastil/tailwind-merge/blob/main/docs/features.md
- https://cva.style/docs
- https://ui.shadcn.com/docs/components/radix/button (the canonical Button source)
- https://tailwindcss.com/docs/upgrade-guide

Last verified: 2026-05-19.
