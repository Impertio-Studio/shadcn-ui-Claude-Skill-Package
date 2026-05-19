# Anti-patterns : observed cva mistakes with verdicts

The following anti-patterns are documented from real shadcn-ui/ui issues, AI-generated component reviews, and shadcn project audits. Each entry gives the symptom, the failed check, the canonical fix, and the reason the mistake is easy to miss.

## AP-1. Variant value as a plain string when an array is structurally clearer

### Shape

```ts
const card = cva("rounded-lg border bg-card text-card-foreground shadow-sm transition-colors hover:shadow-md focus-within:ring-1 focus-within:ring-ring/20")
```

### Verdict

PASS on the 7-point checklist, BUT informational soft-warning.

### Why this is a soft anti-pattern

Cva accepts a single long string for base classes (Rule R1.4 allows it). The validator does NOT fail this shape. However, when the base list exceeds ~8 classes, the canonical convention is to switch to the array form for readability and so each class lands on its own line in a formatter :

```ts
const card = cva([
  "rounded-lg border bg-card text-card-foreground shadow-sm",
  "transition-colors hover:shadow-md",
  "focus-within:ring-1 focus-within:ring-ring/20",
])
```

The validator MAY emit a SOFT warning when a string-form base exceeds 80 characters or 8 classes. The verdict stays PASS ; the warning is informational only.

### Canonical fix

Switch to the array form, group classes by responsibility (layout, color, transition, interaction), one group per array element.

## AP-2. compoundVariants declared before variants in the config object

### Shape

```ts
const buttonVariants = cva("inline-flex", {
  compoundVariants: [
    { variant: "outline", size: "sm", class: "ring-1 ring-ring/20" },
  ],
  variants: {
    variant: { default: "...", outline: "..." },
    size: { default: "...", sm: "..." },
  },
  defaultVariants: { variant: "default", size: "default" },
})
```

### Verdict

FAIL Check 3 (Rule R3.1). Cva still parses this at runtime ; the failure is human-review-grade, not runtime-grade.

### Why this slips through

Cva's `Config` type is an object literal with no ordering constraint. TypeScript accepts any key order. Eslint and prettier do not reorder object keys by default. The compound-first ordering reads "extra classes applied when these variants match" before the reader knows what `outline` and `sm` even are, which makes typo-spotting in the match-keys significantly harder.

In a real review, the most common bug enabled by compound-first ordering is a typo in `variant: "ouline"` (missing `t`) that no test catches because the compound entry silently never matches at runtime.

### Canonical fix

Reorder to : variants, then compoundVariants, then defaultVariants. This is the order in the official cva docs example at https://cva.style/docs/getting-started/installation, in the shadcn registry source, and in every reference component shipped by the shadcn CLI.

## AP-3. Missing VariantProps export (component exported, variants not)

### Shape

```tsx
import { cva, type VariantProps } from "class-variance-authority"

const buttonVariants = cva(/* ... */)

type ButtonProps =
  React.ComponentProps<"button"> &
  VariantProps<typeof buttonVariants>

function Button(props: ButtonProps) { /* ... */ }

export { Button }              // ← only the component
```

### Verdict

FAIL Check 5 (Rule R5.1).

### Why this slips through

The component compiles and runs perfectly. The bug surfaces only when a downstream developer wants to reuse the variant classes for a sibling component (a Next.js `<Link>` styled as a button, a react-router `NavLink` carrying button visuals, a third-party Button that needs to match the shadcn button's hover state). Without the export, the consumer hand-copies the variant class strings and the two diverge silently on the next variant edit.

The verified shadcn registry pattern ALWAYS exports BOTH :

```tsx
export { Button, buttonVariants }
```

### Canonical fix

Add the cva return value name to the export statement. If the file is consumed as a barrel (`index.ts`), re-export from the barrel as well.

## AP-4. cn() nested inside a cva variant string

### Shape

```ts
const buttonVariants = cva("inline-flex", {
  variants: {
    variant: {
      default: cn("bg-primary", "text-primary-foreground", "hover:bg-primary/90"),
      outline: cn("border", "bg-background"),
    },
  },
  defaultVariants: { variant: "default" },
})
```

### Verdict

FAIL Check 2 (Rule R2.3).

### Why this slips through

The component compiles and runs. The output `className` looks identical to the non-cn version because cva's internal `clsx` invocation flattens the cn() output into the same final string. The mistake is doubly invisible : (1) it costs nothing at runtime, (2) it is a structural smell, not a bug.

The reason to fail it anyway is signal preservation. A reader of the cva config block expects "variant value is a class string". Seeing `cn(...)` there triggers the question "is there a conditional happening?", which there is not. Worse, when a developer later adds `cn(active && "ring-2", ...)` to that variant value thinking conditionals work, cva does NOT pass per-call state through to the variant evaluation, and the conditional silently evaluates at module-load time (so it never updates).

### Canonical fix

Remove the cn() wrapper. Pass the classes as a single string or a string array directly :

```ts
default: "bg-primary text-primary-foreground hover:bg-primary/90"
// or
default: ["bg-primary", "text-primary-foreground", "hover:bg-primary/90"]
```

If the goal was a class-array form, use the array directly. Cva runs clsx on it ; cn() in this position is redundant.

## AP-5. Raw className + variant concatenation in the JSX

### Shape

```tsx
function Button({ className, variant, size, ...props }: ButtonProps) {
  return (
    <button
      className={buttonVariants({ variant, size }) + " " + (className ?? "")}
      {...props}
    />
  )
}
```

### Verdict

FAIL Check 6 (Rule R6.3).

### Why this slips through

The component compiles. The DOM receives a className string that visually appears to include all classes. The bug is silent : because twMerge is bypassed, conflicting Tailwind utilities BOTH appear in the final className. CSS specificity then decides which one paints, and the answer depends on Tailwind's generated stylesheet source order, which is NOT stable across builds or across class additions.

A real-world example : a caller writes `<Button className="bg-red-500" />` and the rendered `<button>` carries `class="bg-primary bg-red-500"`. In some Tailwind builds the red wins ; in others the primary wins. The developer reports "the override sometimes works, sometimes does not" and the root cause is not the variant, not the cn() helper itself, but the failure to call it.

### Canonical fix

Replace the raw concat with `cn()` and fold the caller className into the variants call :

```tsx
<button className={cn(buttonVariants({ variant, size, className }))} {...props} />
```

This routes the entire class string through twMerge, which resolves the bg-primary-vs-bg-red-500 conflict deterministically by source position (last wins). The caller className arrives last in the variants call, so it wins.

## AP-6. defaultVariants typo silently disables the default

### Shape

```ts
const buttonVariants = cva("inline-flex", {
  variants: {
    variant: { default: "bg-primary", outline: "border" },
    size: { default: "h-9", sm: "h-8" },
  },
  defaultVariants: { varient: "default", size: "default" },
  //                ^^^^^^^ axis-name typo : "varient" is not a declared axis
})
```

### Verdict

FAIL Check 4 (Rule R4.3).

### Why this slips through

Cva's `Config` type accepts ANY string key in `defaultVariants` because the config type is structural and the validator on the value side only checks that the value-key exists for the matching axis-key (which here it does not match anything, so cva silently ignores the entry). TypeScript catches NOTHING. ESLint catches NOTHING. The component runs.

The visible symptom is that calling `<Button />` with no variant prop renders WITHOUT the default variant classes : the `variant` axis falls back to undefined and `variants.variant[undefined]` produces no classes. The component appears unstyled. The developer often blames the variant classes themselves and starts editing the wrong file.

The same trap fires on value-key typos : `defaultVariants: { variant: "defualt" }` where "defualt" is misspelled. Cva looks up `variants.variant["defualt"]`, finds undefined, and produces no classes. Silent miss.

### Canonical fix

Cross-check every key and value in `defaultVariants` against `variants` :

```ts
defaultVariants: { variant: "default", size: "default" }
```

If a default for an axis is unwanted, opt out explicitly with `null` :

```ts
defaultVariants: { variant: "default", size: null }
```

The explicit `null` is the only way to communicate to a future reader "yes, this axis is deliberately default-less". A missing key is ambiguous : it could be a typo or an opt-out.

## AP-7. asChild prop accepted but Slot never invoked

### Shape

```tsx
type ButtonProps = React.ComponentProps<"button"> &
  VariantProps<typeof buttonVariants> &
  { asChild?: boolean }

function Button({ className, variant, size, asChild = false, ...props }: ButtonProps) {
  // asChild destructured but never branched on
  return (
    <button
      className={cn(buttonVariants({ variant, size, className }))}
      {...props}
    />
  )
}
```

### Verdict

FAIL Check 7 (Rule R7.1).

### Why this slips through

The component compiles. The prop accepts `asChild={true}` without error because TypeScript sees it in the props type. The behavior is the silent failure : `<Button asChild><Link href="/about">About</Link></Button>` renders as a `<button>` wrapping the `<Link>`, NOT as a `<Link>` carrying the button classes. The result is invalid HTML (a button cannot contain an anchor) AND the button's interaction handlers do not fire on the link's click area in the expected polymorphic way.

The check fires when the validator sees `asChild` in the destructure or in the props type but does NOT find `Slot` imported or a `const Comp = asChild ? ...` branch in the render body.

### Canonical fix

Import Slot and branch on asChild :

```tsx
import { Slot } from "radix-ui"     // evergreen-2026

function Button({ className, variant, size, asChild = false, ...props }: ButtonProps) {
  const Comp = asChild ? Slot.Root : "button"
  return (
    <Comp
      data-slot="button"
      className={cn(buttonVariants({ variant, size, className }))}
      {...props}
    />
  )
}
```

On legacy projects still using `@radix-ui/react-slot`, the import is `import { Slot } from "@radix-ui/react-slot"` and the branch is `const Comp = asChild ? Slot : "button"`. Both are valid per the cva-validator ; the validator accepts either shape based on the import source.

## AP-8. Hand-rolled variant union in parallel to cva

### Shape

```tsx
const buttonVariants = cva("inline-flex", {
  variants: {
    variant: { default: "bg-primary", outline: "border", ghost: "hover:bg-accent" },
    size: { default: "h-9", sm: "h-8", lg: "h-10" },
  },
  defaultVariants: { variant: "default", size: "default" },
})

type ButtonProps = React.ComponentProps<"button"> & {
  variant?: "default" | "outline" | "ghost"    // hand-rolled
  size?: "default" | "sm" | "lg"               // hand-rolled
}
```

### Verdict

FAIL Check 5 (Rule R5.3).

### Why this slips through

The component compiles. The TypeScript inference at the call site works correctly for `<Button variant="default" />`. The failure is drift : the day a developer adds `link: "text-primary underline"` to the cva config, the hand-rolled union does NOT update. Consumers can no longer pass `variant="link"` without a type error, even though the runtime would accept it. Worse, the type error often makes the developer remove the variant from the cva config to "fix" the TypeScript error, and the variant disappears entirely.

The canonical pattern derives the union from the cva return value via `VariantProps<typeof buttonVariants>` so the type is always in sync with the runtime.

### Canonical fix

```tsx
import { type VariantProps } from "class-variance-authority"

type ButtonProps =
  React.ComponentProps<"button"> &
  VariantProps<typeof buttonVariants>
```

The `VariantProps` helper extracts the union from the cva return value at TYPE level only, so renaming a variant in cva immediately surfaces as a type error at every call site.

## Summary table

| # | Anti-pattern | Failed check | Severity |
|---|--------------|--------------|----------|
| AP-1 | Long string base when array is clearer | (none, soft warning) | informational |
| AP-2 | compoundVariants before variants in config | Check 3 | structural |
| AP-3 | Missing VariantProps export | Check 5 | structural |
| AP-4 | cn() nested inside cva variant string | Check 2 | structural |
| AP-5 | Raw className concatenation in JSX | Check 6 | behavioural |
| AP-6 | defaultVariants typo silently disables default | Check 4 | behavioural (silent) |
| AP-7 | asChild prop without Slot in body | Check 7 | behavioural |
| AP-8 | Hand-rolled variant union in parallel to cva | Check 5 | drift risk |

The validator emits a verdict block for each anti-pattern found, ordered by check number. Severity is informational only ; PASS-vs-FAIL is binary per check.
