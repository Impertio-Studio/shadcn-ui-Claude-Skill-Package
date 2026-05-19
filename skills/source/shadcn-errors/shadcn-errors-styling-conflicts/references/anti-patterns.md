# Anti-Patterns : class merging in shadcn projects

Reference material for `shadcn-errors-styling-conflicts`. Six verified anti-patterns. Each entry: name, what it looks like, what actually happens, the root cause, and the deterministic fix.

---

## AP-01 : Template-string className concatenation

### Symptom

```tsx
<div className={"p-4 rounded-md " + (active && "bg-primary") + " " + className} />
```

### What actually happens

The compiled string is a flat space-separated list. No merge step runs. Duplicate utility families (e.g., `p-4` plus a consumer `p-2`) BOTH reach the DOM. Tailwind's generated CSS source order, not the developer's intent, decides which wins.

### Root cause

Raw string concatenation is a JavaScript primitive. It does not know about Tailwind class groups, conditionals, or important-namespace tracking. clsx is needed for the conditional shape, twMerge is needed for the conflict resolution. cn() wraps both into one call.

### Deterministic fix

```tsx
import { cn } from "@/lib/utils"
<div className={cn("p-4 rounded-md", active && "bg-primary", className)} />
```

ALWAYS use `cn()` whenever className is composed from more than one source. NEVER use `+` or template literals to build className.

---

## AP-02 : Nested cn() inside cva variant values (no-op)

### Symptom

```ts
const button = cva("rounded", {
  variants: {
    tone: {
      brand: cn("bg-primary", "text-primary-foreground"),
      ghost: cn("bg-transparent", "text-foreground"),
    },
  },
})
```

### What actually happens

cva runs `clsx` internally on each variant value before producing the final string. Wrapping that value in `cn()` adds a redundant twMerge call against a fragment of classes that has not yet been combined with base, compoundVariants, or consumer className. The redundant merge runs on every call to `button(...)` (no memoization at this layer), wasting CPU. It does NOT make the final output more correct because the call site MUST still run cn() to bring in consumer className.

### Root cause

A misunderstanding of the cva pipeline. cva is responsible for combining base + variant + compoundVariants + defaultVariants. It outputs a string. That string is THEN merged once at the call site with consumer className via cn(). cn() inside the cva variant definition merges a half-built fragment.

### Deterministic fix

```ts
const button = cva("rounded", {
  variants: {
    tone: {
      brand: "bg-primary text-primary-foreground",
      ghost: "bg-transparent text-foreground",
    },
  },
})

// Call site does the merge that matters
className={cn(button({ tone }), className)}
```

NEVER call `cn()` inside a cva variant value. Use plain strings or string arrays.

---

## AP-03 : Consumer className passed BEFORE variant in cn()

### Symptom

```tsx
function Button({ size, className }: ButtonProps) {
  return <button className={cn(className, buttonVariants({ size }))} />
}
```

### What actually happens

tailwind-merge is last-wins. The variant string follows the consumer className inside cn(), so the variant's `px-3` displaces the consumer's `px-8`. The override never takes effect, and the consumer cannot understand why their `className="px-8"` is ignored.

### Root cause

Inverted merge order. Consumer override means consumer wins, which means consumer must be the LAST input to cn().

### Deterministic fix

```tsx
function Button({ size, className }: ButtonProps) {
  return <button className={cn(buttonVariants({ size }), className)} />
}
```

ALWAYS put consumer-supplied `className` as the LAST argument to cn(). The shadcn ui canonical Button file uses this exact order; do not deviate.

---

## AP-04 : Assuming `!important` always wins

### Symptom

```tsx
// Tailwind v4 syntax
<div className={cn("bg-red-500!", "bg-blue-500")} />
// Developer expects blue ; gets the rendered string "bg-red-500! bg-blue-500" with red applied (important wins in CSS specificity).
// OR sometimes:
<div className={cn("bg-red-500", "bg-blue-500!")} />
// Developer expects blue (important) ; gets blue, but ALSO bg-red-500 in DOM and is confused that the override is "noisy."
```

### What actually happens

tailwind-merge tracks important utilities in a SEPARATE conflict namespace. A non-important class CANNOT override an important class of the same family. Both classes end up in the DOM and the important one wins via CSS specificity. A plain `bg-blue-500` that the developer hoped would replace `bg-red-500!` is left as dead weight in the class list.

### Root cause

Misconception that `!` is just CSS specificity boost that participates in the normal merge order. In tailwind-merge it is a separate namespace by design (because !important participates in CSS specificity, not in left-to-right class-list order).

### Deterministic fix

Make both sides important OR neither side important. Do not mix.

```tsx
// Both important : last wins inside the important namespace
<div className={cn("bg-red-500!", "bg-blue-500!")} />   // → "bg-blue-500!"

// Neither important : last wins inside the plain namespace
<div className={cn("bg-red-500", "bg-blue-500")} />     // → "bg-blue-500"
```

ALWAYS keep important-modifier usage symmetric within a single merge call.

---

## AP-05 : Mixing arbitrary PROPERTIES with their utility equivalents

### Symptom

```tsx
<div className={cn("[padding:1rem]", "p-4")} />
```

### What actually happens

Both classes reach the DOM. tailwind-merge intentionally does NOT resolve arbitrary properties (the `[prop:value]` form) against their matching Tailwind utilities. The documented reason: indexing every CSS property against every utility would inflate tailwind-merge's bundle size, and the use case is rare.

### Root cause

Confusion between arbitrary VALUES and arbitrary PROPERTIES:

- Arbitrary VALUE : `p-[1rem]` ; targets the `p-*` utility family ; merges with `p-4`.
- Arbitrary PROPERTY : `[padding:1rem]` ; is a custom CSS property assignment ; does NOT merge with `p-4`.

### Deterministic fix

Use the arbitrary-VALUE form when you need merging, or pick one source for the same CSS property and stick to it.

```tsx
// Option A : arbitrary value form (merges)
<div className={cn("p-[1rem]", "p-4")} />               // → "p-4"

// Option B : single source of truth
<div className={cn("p-4")} />
```

NEVER mix `[css-prop:value]` syntax with utility classes that target the same CSS property in the same merge.

---

## AP-06 : Custom `cn()` helper missing twMerge

### Symptom

```ts
// lib/utils.ts (homegrown, missing the twMerge step)
import { clsx, type ClassValue } from "clsx"
export function cn(...inputs: ClassValue[]) {
  return clsx(inputs)
}
```

The cn() call sites compile and the project boots. But:

- `<Button className="px-8">` does NOT override the variant's `px-3`. Both reach the DOM.
- `cn("p-4", "p-2")` returns `"p-4 p-2"`, not `"p-2"`.
- Every component override silently produces double-up Tailwind classes.

### What actually happens

clsx alone handles conditional composition. It is great at flattening `{ a: true, b: false }` and falsy values. It does NOT know what `p-4` means, so it cannot remove `p-4` when `p-2` is also present. The merge step that resolves Tailwind family conflicts is `twMerge`, which is separate from clsx.

### Root cause

A developer copied the cn() shape from memory or a tutorial that omitted twMerge. Or the project pre-dates shadcn ui adoption and used clsx alone for years before adding Tailwind.

### Deterministic fix

Restore the canonical helper as shipped by `shadcn init`:

```ts
// lib/utils.ts
import { clsx, type ClassValue } from "clsx"
import { twMerge } from "tailwind-merge"

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs))
}
```

Audit other call sites in the project that might use a non-canonical cn(). The simplest verification: type `twMerge(` into your editor inside `lib/utils.ts`; if your IDE cannot resolve the import, tailwind-merge is not installed.

`pnpm add tailwind-merge` (or `npm i tailwind-merge`) if missing.

---

## 7. References

- https://github.com/dcastil/tailwind-merge/blob/main/docs/features.md
- https://cva.style/docs (variant value shape, defaultVariants behaviour)
- https://ui.shadcn.com/docs (canonical `cn()` helper)
- https://github.com/dcastil/tailwind-merge/blob/main/docs/api-reference.md (extendTailwindMerge)

Last verified: 2026-05-19.
