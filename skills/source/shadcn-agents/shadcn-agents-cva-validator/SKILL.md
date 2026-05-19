---
name: shadcn-agents-cva-validator
description: >
  Use when reviewing, validating, or auditing a shadcn ui component that
  uses class-variance-authority (cva), when an LLM has just produced a
  Button-like, Badge-like, Alert-like, or any other cva-driven component
  and you must verify the variant API before accepting the diff, when a
  consumer className does not visually win and you suspect cva merge
  order or a missing cn() call, when extending an existing shadcn
  component with a new variant or compoundVariant and you want a
  deterministic checklist before commit, when porting a hand-written
  variants-via-if-branches component to cva, when writing a custom
  primitive that should follow the shadcn convention, when verifying
  that VariantProps inference is exported so consumers can extend the
  component type, or when running a code-review pass on AI-generated
  cva code to catch the canonical mistakes (nested cn inside cva,
  compoundVariants placed before variants, missing defaultVariants,
  raw className concatenation, asChild without Slot, variant key
  typos that silently disable a default).
  Prevents the seven canonical cva failures : (1) variant definition
  shape wrong (string instead of class string or array), (2)
  compoundVariants placed in the wrong object position so the parser
  silently accepts but defaults never resolve, (3) defaultVariants
  missing or with a typo that no TypeScript check catches because cva
  values are stringly-typed, (4) VariantProps export missing so
  consumers cannot extend, (5) caller className concatenated via
  template string instead of merged via cn() so twMerge is bypassed
  and duplicate utilities leak, (6) asChild rendered without Radix
  Slot so polymorphic composition silently breaks, (7) cn() called
  INSIDE a cva variant string (no-op, cva already runs clsx
  internally on its inputs).
  Covers the 7-point validator checklist (base, variants shape,
  compoundVariants order, defaultVariants, VariantProps export, cn()
  merge, asChild Slot), per-checkpoint pass and fail criteria with
  verbatim code patterns, sample-code-to-verdict mappings for at least
  a dozen common cva shapes, the canonical shadcn Button reference
  template that all checks reduce to, the cva evaluation order
  (defaults, then variants, then compoundVariants, then caller
  className) and how to verify it from the diff alone, and the
  Companion Skills cross-links to shadcn-syntax-variant-cva (B3,
  reference) and shadcn-errors-styling-conflicts (B11, debugging).
  Keywords: cva validator, validate cva component, variant API check,
  cva correctness check, VariantProps validation, cn helper
  validation, compoundVariants order check, defaultVariants typo,
  validate variant code, review cva component, cva self-check, how
  do I verify a cva component, is my cva component correct, cva
  checklist, cva audit, class-variance-authority review, shadcn
  variant audit, variant code review, cva diff review, asChild Slot
  check, missing defaultVariants, variant prop not applying, override
  className not winning, template string concat variant, raw
  className concatenation, nested cn inside cva, cva inside cva,
  compoundVariants before variants, VariantProps not exported, why
  is my variant not applying, audit variant component before commit,
  pre-commit cva check.
license: MIT
compatibility: "Designed for Claude Code. Requires shadcn ui evergreen-2026."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# shadcn ui Agent : cva Variant API Validator

This skill is the deterministic checklist a code-review or pre-commit agent uses to verify that any cva-driven shadcn component is shaped correctly. It is a VALIDATOR, not a tutorial. For the underlying cva theory, ALWAYS read `shadcn-syntax-variant-cva` first. For debugging a className override that does not visually win, ALWAYS read `shadcn-errors-styling-conflicts`.

The validator answers exactly one question per checkpoint : PASS or FAIL. It does not propose creative refactors. When a check fails, the verdict cites the failing line and points to the canonical fix.

## Quick Reference

### Validator workflow (top of every review)

1. Locate the cva call (search for `cva(` in the diff or file).
2. Run the 7-point checklist from top to bottom in declared order.
3. Stop at the first FAIL and emit the verdict with line reference.
4. If all 7 pass, emit `VERDICT: PASS` plus a one-line summary.
5. When called on a multi-file diff, repeat per component file independently.

The 7 checks are ordered by failure frequency observed in real shadcn projects : base, variants shape, compoundVariants order, defaultVariants, VariantProps export, cn() merge, asChild Slot. The first three catch over half of every cva regression seen in `shadcn-ui/ui` issues.

### The 7-point checklist (memorize)

| # | Check | One-line pass criterion |
|---|-------|--------------------------|
| 1 | Base | First positional argument to cva is a string OR a string array |
| 2 | Variants shape | Every variant value is a string OR a string array, never a function, never nested cn() |
| 3 | compoundVariants order | `compoundVariants` declared AFTER `variants` in the config object literal |
| 4 | defaultVariants | Every variant axis has an entry in `defaultVariants` (or explicit `null`) |
| 5 | VariantProps export | The cva return value is exported AND `VariantProps<typeof X>` is in the props type |
| 6 | cn() merge | Component returns `className={cn(xVariants({ ..., className }))}`, NEVER template-string concat |
| 7 | asChild Slot | If `asChild?: boolean` is in props, body renders `<Slot.Root>` (or `<Slot>`) when true, native tag when false |

When the answer to any of the seven is "no" the verdict is FAIL ; cite the exact line, name the check, and reference the canonical pattern in `references/examples.md`.

### One-glance verdict format

Every validator pass MUST emit the verdict in this exact shape so downstream pipes can grep it :

```
VERDICT: PASS (or FAIL)
COMPONENT: <file path>:<line of cva call>
FAILED CHECK: <check number and name>   ← omit on PASS
REASON: <one sentence>                  ← omit on PASS
CANONICAL FIX: references/examples.md § <section name>   ← omit on PASS
```

## The 7 checkpoints in detail

### Check 1 : Base classes shape

PASS if the first argument to `cva` is a string literal, a template literal without interpolation, or an array of those :

```ts
cva("inline-flex items-center")                 // PASS : string
cva(["inline-flex", "items-center"])            // PASS : array
cva(`inline-flex items-center`)                 // PASS : template, no interp
```

FAIL if any of these shapes appear :

```ts
cva(cn("inline-flex", "items-center"), { ... })          // FAIL : cn nested inside cva ; cva already runs clsx
cva(twMerge("inline-flex", "items-center"), { ... })     // FAIL : twMerge has nothing to merge
cva("inline-flex" + " " + "items-center", { ... })       // FAIL : plus-concat ; use array
cva(getBaseClasses(), { ... })                           // FAIL : dynamic base defeats static analysis
```

REASON the nested cn() and twMerge cases fail : cva internally passes its argument through clsx, so a pre-flattened string adds no value and obscures static analysis. The plus-concat case fails because TypeScript loses the literal type and the array form is the documented convention.

### Check 2 : Variants object shape

PASS if every leaf value under `variants.<axis>.<value>` is a string or a string array. Quoted keys, no boolean keys, no nested objects :

```ts
variants: {
  variant: {
    default: "bg-primary text-primary-foreground",
    outline: ["border", "bg-background", "hover:bg-accent"],
  },
  size: {
    default: "h-9 px-4 py-2",
    sm: "h-8 rounded-md px-3",
  },
}
```

FAIL on any of these shapes :

```ts
// FAIL : nested cn() ; cva already calls clsx
variant: { default: cn("bg-primary", "text-primary-foreground") }

// FAIL : function instead of class string ; cva does not invoke variant values
variant: { default: () => "bg-primary" }

// FAIL : object nesting ; cva expects a flat string-to-string map
variant: { default: { base: "bg-primary", hover: "hover:bg-primary/90" } }

// FAIL : unquoted boolean key ; cva accepts but VariantProps inference becomes awkward
variant: { true: "bg-primary", false: "bg-secondary" }
```

REASON : cva runs each value through clsx once, NOT recursively. Functions, nested objects, and pre-merged cn() strings either no-op or break TypeScript inference.

### Check 3 : compoundVariants order and shape

PASS if `compoundVariants` is declared AFTER `variants` and BEFORE `defaultVariants` in the config-object source order, and every entry is `{ ...matchKeys, class: "..." }` (or `className: "..."` ; both are valid per cva docs) :

```ts
const buttonVariants = cva("inline-flex", {
  variants: { /* ... */ },
  compoundVariants: [
    { variant: "outline", size: "sm", class: "ring-1 ring-ring/20" },
    { variant: "outline", size: "lg", class: "ring-2 ring-ring/30" },
  ],
  defaultVariants: { variant: "default", size: "default" },
})
```

FAIL on :

```ts
// FAIL : compoundVariants BEFORE variants ; cva still parses but readability and merge-debugging suffer
cva("inline-flex", {
  compoundVariants: [{ variant: "outline", size: "sm", class: "ring-1" }],
  variants: { /* ... */ },
})

// FAIL : entry missing class or className key
compoundVariants: [{ variant: "outline", size: "sm" }]

// FAIL : entry references a variant or value name that does not exist in variants
compoundVariants: [{ variant: "ghostly", size: "sm", class: "ring-1" }]
```

REASON : cva is permissive on parse order, but compound-before-variants is the canonical reading-order anti-pattern called out in `shadcn-errors-styling-conflicts`. A reviewer reading top-to-bottom sees a compound match for variants that have not been defined yet and is more likely to miss a typo in the match-keys. Misspelled match-keys silently never match ; cva does NOT throw and TypeScript does NOT catch a stringly-typed mismatch unless the consumer also runs `VariantProps` validation downstream.

### Check 4 : defaultVariants completeness

PASS if `defaultVariants` exists AND every axis declared in `variants` has either a string value OR an explicit `null` in `defaultVariants` :

```ts
defaultVariants: { variant: "default", size: "default" }      // PASS : both axes covered
defaultVariants: { variant: "default", size: null }            // PASS : size opted out explicitly
```

FAIL if :

```ts
// FAIL : missing defaultVariants entirely ; every undefined prop yields no class for that axis
const x = cva("inline-flex", { variants: { variant: { default: "..." } } })

// FAIL : typo in axis name ; cva treats "varient" as an unknown axis and silently ignores it
defaultVariants: { varient: "default", size: "default" }

// FAIL : typo in value name ; cva looks up variants.variant["defualt"] and finds undefined ; no classes
defaultVariants: { variant: "defualt", size: "default" }
```

REASON : cva is stringly-typed inside the config object. TypeScript does NOT catch axis-name typos in `defaultVariants` because the config is a structural object literal accepted as `Config`. The typo silently disables the default and components render unstyled when the consumer omits the prop. The validator MUST cross-check each `defaultVariants` key against the `variants` keys, and each value against the value-keys of the matching axis.

### Check 5 : VariantProps export

PASS if BOTH of the following hold :

1. The cva return value is exported (`export { Button, buttonVariants }` OR `export const buttonVariants = cva(...)`).
2. The component props type intersects `VariantProps<typeof buttonVariants>` directly OR via a type alias.

```ts
import { cva, type VariantProps } from "class-variance-authority"

const buttonVariants = cva(/* ... */)

type ButtonProps =
  React.ComponentProps<"button"> &
  VariantProps<typeof buttonVariants> &
  { asChild?: boolean }

function Button({ variant, size, className, ...props }: ButtonProps) { /* ... */ }

export { Button, buttonVariants }
```

FAIL if :

```ts
// FAIL : variants object not exported ; consumer cannot reuse classes for a link styled as a button
const buttonVariants = cva(/* ... */)
export { Button }

// FAIL : variant union hand-written ; loses type-sync if a variant value is renamed
type ButtonProps = { variant?: "default" | "outline", ... }

// FAIL : VariantProps reference missing entirely ; component accepts the prop but type has no union
type ButtonProps = React.ComponentProps<"button"> & { variant?: string }
```

REASON : without `VariantProps<typeof buttonVariants>` the consumer surface is detached from the cva source of truth. Renaming `outline` to `outlined` then no longer surfaces as a TypeScript error at the call site, and the runtime simply silently produces no classes.

### Check 6 : cn() merge of consumer className

PASS if the component renders className as `cn(xVariants({ ..., className }))` (the canonical shadcn pattern, passing `className` as a key of the variants call) OR `cn(xVariants({ ... }), className)` (the older pattern, passing className as a second argument to cn) :

```tsx
// PASS : canonical evergreen-2026 pattern
<Comp className={cn(buttonVariants({ variant, size, className }))} {...props} />

// PASS : older but valid pattern (still seen in older shadcn revisions)
<Comp className={cn(buttonVariants({ variant, size }), className)} {...props} />
```

FAIL on any of these :

```tsx
// FAIL : template-string concat ; twMerge bypassed, duplicate utilities leak to DOM
<button className={`${buttonVariants({ variant, size })} ${className}`} />

// FAIL : plus-concat ; same problem ; also brittle on undefined className
<button className={buttonVariants({ variant, size }) + " " + (className ?? "")} />

// FAIL : caller className dropped entirely
<button className={buttonVariants({ variant, size })} />

// FAIL : double cn() ; redundant, indicates the author misunderstands the API
<button className={cn(cn(buttonVariants({ variant, size }), className))} />
```

REASON : `cn()` is the only path that runs `tailwind-merge`. Without twMerge a caller `className="bg-red-500"` and a variant class `"bg-primary"` both appear in the DOM ; CSS specificity decides arbitrarily by source order. The shadcn convention is that caller className ALWAYS wins.

### Check 7 : asChild Slot pattern

If the component accepts `asChild?: boolean`, PASS only if all three hold :

1. The component imports `Slot` from `radix-ui` (evergreen-2026 unified package) OR from `@radix-ui/react-slot` (pre-Feb-2026 packages still on the legacy split layout).
2. The body picks `Slot.Root` (unified) or `Slot` (legacy) when `asChild === true`, and the native tag string otherwise.
3. The same className expression (the `cn(xVariants(...))` call) wraps both branches.

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

FAIL on any of these :

```tsx
// FAIL : asChild prop accepted but never branched ; renders <button> regardless
function Button({ asChild = false, ...props }) { return <button {...props} /> }

// FAIL : Slot not imported ; runtime ReferenceError
function Button({ asChild }) { const Comp = asChild ? Slot.Root : "button"; ... }

// FAIL : className applied only to one branch ; asChild path loses variant classes
return asChild ? <Slot.Root {...props} /> : <button className={cn(...)} {...props} />
```

REASON : the asChild pattern only works because Radix `Slot` forwards the parent's className, ref, and onClick into the child. Skipping `Slot` makes `asChild` non-functional ; applying className to only one branch silently produces unstyled output exactly half the time.

## Sample-code-to-verdict mapping (illustrative)

The full table lives in `references/methods.md` § Sample-code-to-verdict. A few high-yield examples summarised here :

| Symptom | Likely failed check | Verdict pointer |
|---------|---------------------|-----------------|
| `cn()` appears INSIDE a `variants.variant.default` string | Check 2 | examples.md WRONG-A |
| compoundVariants declared as the first key in the cva config | Check 3 | examples.md WRONG-B |
| `defaultVariants: { varient: "default" }` (typo) | Check 4 | examples.md WRONG-C |
| Consumer className passed but `cn` not called | Check 6 | examples.md WRONG-D |
| `asChild` prop in interface but body always renders `<button>` | Check 7 | examples.md WRONG-E |
| `export { Button }` without `buttonVariants` | Check 5 | examples.md WRONG-F |

## Cross-check with downstream skills

This validator alone CANNOT detect runtime issues caused by Tailwind v3-vs-v4 class semantics ; that diagnosis lives in `shadcn-errors-styling-conflicts` (B11). When Check 6 PASSES but the caller still reports "override does not visually win", redirect the user to that skill for the merge-order debug. The B11 skill enumerates the `arbitrary-vs-preset`, `important-modifier`, and `v3-vs-v4-rename` traps that survive a correct cn() call.

## Companion Skills

- `shadcn-syntax-variant-cva` (B3, reference for the underlying API : signature, base, variants, compoundVariants, defaultVariants, VariantProps, evaluation order, asChild Slot, the cn() helper composition). ALWAYS read it BEFORE writing or auditing a cva component for the first time.
- `shadcn-errors-styling-conflicts` (B11, debugging for the case where the validator passes all 7 checks but the consumer still sees a visual override failure). Read it AFTER a PASS verdict if the bug persists.
- `shadcn-agents-component-selector` (sibling agent : picks the correct shadcn primitive in the first place ; out of scope for this validator).
- `shadcn-impl-extending-components` (impl-layer recipe for adding a new variant or compoundVariant to an existing shadcn component without breaking VariantProps inference).

## Reference files

- `references/methods.md` : the 7 checks as a numbered enforcement spec plus the sample-code-to-verdict table.
- `references/examples.md` : one WRONG cva component with multiple violations and the verdict, one RIGHT cva component verified, and edge cases (variant value as class-array, compoundVariants with multiple match keys, custom variant extension via spread).
- `references/anti-patterns.md` : at least six observed cva anti-patterns from real shadcn projects with the verdict and the canonical fix.
