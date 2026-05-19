# Methods : the cva validator as enforcement rules

This file is the numbered, machine-readable spec the agent applies. Every rule is one PASS criterion plus the verbatim shape of a FAIL example. Rules are ordered the same as the SKILL.md checklist (1 through 7). After the rule list, a sample-code-to-verdict table maps observed code shapes to the exact failing check and the canonical fix.

## Numbered rules

### Rule 1. Base classes shape

R1.1. The first positional argument to `cva(...)` MUST be a string literal, a template literal without interpolation, OR an array whose elements are strings or template literals without interpolation.

R1.2. The argument MUST NOT be the return value of `cn(...)`, `clsx(...)`, `twMerge(...)`, or any other helper function. cva already runs `clsx` internally on its inputs.

R1.3. The argument MUST NOT be the return value of a user-defined function (`getBase()`, `computeClasses(...)`, etc.). Dynamic base classes defeat static analysis and the `VariantProps` inference still works only because cva ignores the runtime value at type level.

R1.4. PASS shapes :
  - `cva("inline-flex items-center")`
  - `cva(["inline-flex", "items-center"])`
  - `cva(\`inline-flex items-center\`)` (template, no `${...}`)

R1.5. FAIL shapes :
  - `cva(cn("inline-flex", "items-center"), { ... })`
  - `cva(twMerge("a", "b"), { ... })`
  - `cva("a" + " " + "b", { ... })`
  - `cva(getBase(), { ... })`

### Rule 2. Variants object shape

R2.1. The `variants` property of the cva config object MUST be a plain object literal whose top-level keys name the variant axes (`variant`, `size`, `tone`, etc.).

R2.2. For each axis, the value MUST be a plain object literal whose keys are quoted string variant-value names, and whose values are either a string OR an array of strings.

R2.3. Variant values MUST NOT be functions, nested objects, or pre-flattened cn() calls.

R2.4. Variant-value names SHOULD be quoted ("default", "outline") rather than bare identifiers (default, outline) ; cva accepts both, but bare identifiers reduce parser robustness for tooling that grep-scans the source.

R2.5. PASS shape :

```ts
variants: {
  variant: { default: "bg-primary", outline: ["border", "bg-background"] },
  size: { default: "h-9 px-4 py-2", sm: "h-8 px-3" },
}
```

R2.6. FAIL shapes :

```ts
variant: { default: cn("bg-primary", "text-primary-foreground") }       // nested cn
variant: { default: () => "bg-primary" }                                // function
variant: { default: { base: "bg-primary", hover: "hover:bg-primary" } } // nested object
variant: { true: "bg-primary", false: "bg-secondary" }                  // boolean key, unquoted
```

### Rule 3. compoundVariants position and shape

R3.1. If `compoundVariants` is declared, it MUST appear AFTER the `variants` key and BEFORE the `defaultVariants` key in the cva config object literal. cva does not enforce this at runtime, but a human reviewer reading top-to-bottom relies on this order.

R3.2. Every entry MUST be an object literal containing one OR MORE keys that name a variant axis declared in `variants`, plus exactly ONE of `class` or `className` whose value is a string or an array of strings.

R3.3. Every variant-axis key referenced in a `compoundVariants` entry MUST exist in `variants`, and its value MUST equal a value declared in `variants[axis]`. Typos in either name produce silent no-match.

R3.4. Multiple match-keys in a single entry produce AND semantics (all keys must match the caller's invocation). To express OR semantics, declare multiple `compoundVariants` entries.

R3.5. PASS shape :

```ts
compoundVariants: [
  { variant: "outline", size: "sm", class: "ring-1 ring-ring/20" },
  { variant: "outline", size: "lg", class: "ring-2 ring-ring/30" },
  { variant: "ghost", class: "shadow-none" },                       // single match-key is fine
]
```

R3.6. FAIL shapes :

```ts
compoundVariants: [{ variant: "outline", size: "sm" }]               // missing class/className
compoundVariants: [{ variant: "ghostly", size: "sm", class: "..." }] // ghostly is not in variants.variant
compoundVariants: [{ variant: "outline", size: "smol", class: "..." }] // smol is not in variants.size
{ compoundVariants: [...], variants: {...}, defaultVariants: {...} } // compoundVariants before variants
```

### Rule 4. defaultVariants completeness

R4.1. The `defaultVariants` object MUST contain a key for every variant axis declared in `variants`.

R4.2. Each value in `defaultVariants` MUST equal a value-key declared under the matching axis in `variants`, OR be the explicit literal `null` to opt out of any default.

R4.3. Axis names and value names in `defaultVariants` are STRING-typed in cva ; TypeScript does NOT catch typos at the config-literal site. The validator MUST cross-check both names against `variants`.

R4.4. PASS shapes :

```ts
defaultVariants: { variant: "default", size: "default" }              // both axes covered
defaultVariants: { variant: "default", size: null }                   // size opted out explicitly
```

R4.5. FAIL shapes :

```ts
// missing defaultVariants entirely
const x = cva("inline-flex", { variants: { variant: { default: "..." } } })

// axis-name typo
defaultVariants: { varient: "default", size: "default" }

// value-name typo
defaultVariants: { variant: "defualt", size: "default" }

// partial coverage (size axis declared but no default and not opted out)
const x = cva("inline-flex", {
  variants: { variant: { default: "..." }, size: { default: "...", sm: "..." } },
  defaultVariants: { variant: "default" },                                       // missing size
})
```

### Rule 5. VariantProps export

R5.1. The result of `cva(...)` MUST be assigned to a named const (`const buttonVariants = cva(...)`) and that name MUST appear in an `export` statement of the file.

R5.2. The component's props type MUST intersect `VariantProps<typeof <varName>>` (directly OR via a type alias) so the variant union is sourced from the cva return value.

R5.3. The component MUST NOT hand-roll the variant union (`variant?: "default" | "outline"`) as a parallel type.

R5.4. PASS shape :

```ts
const buttonVariants = cva(/* ... */)

type ButtonProps =
  React.ComponentProps<"button"> &
  VariantProps<typeof buttonVariants> &
  { asChild?: boolean }

function Button({ /* ... */ }: ButtonProps) { /* ... */ }

export { Button, buttonVariants }
```

R5.5. FAIL shapes :

```ts
// variants not exported
const buttonVariants = cva(/* ... */)
export { Button }

// hand-rolled union
type ButtonProps = { variant?: "default" | "outline" }

// VariantProps reference missing
type ButtonProps = React.ComponentProps<"button"> & { variant?: string }
```

### Rule 6. cn() merge of consumer className

R6.1. The component MUST forward the consumer-supplied `className` prop into the variant call OR into the outer `cn()` call.

R6.2. The final className expression MUST be `cn(<varName>({ ..., className }))` (canonical) OR `cn(<varName>({ ... }), className)` (legacy but valid).

R6.3. The final className expression MUST NOT use template-string interpolation (\`${variants(...)} ${className}\`) or plus-concatenation (`variants(...) + " " + className`).

R6.4. The component MUST NOT drop the consumer className. A render that produces `className={buttonVariants({ variant, size })}` without folding `className` is FAIL.

R6.5. PASS shapes :

```tsx
<Comp className={cn(buttonVariants({ variant, size, className }))} {...props} />
<Comp className={cn(buttonVariants({ variant, size }), className)} {...props} />
```

R6.6. FAIL shapes :

```tsx
<button className={`${buttonVariants({ variant, size })} ${className}`} />
<button className={buttonVariants({ variant, size }) + " " + (className ?? "")} />
<button className={buttonVariants({ variant, size })} />
<button className={cn(cn(buttonVariants({ variant, size }), className))} />  // double cn
```

### Rule 7. asChild Slot pattern

R7.1. If `asChild?: boolean` is part of the component props type, the body MUST import `Slot` (from `radix-ui` in evergreen-2026 OR from `@radix-ui/react-slot` in legacy projects) AND branch the rendered element on the prop.

R7.2. The branch MUST be `const Comp = asChild ? Slot.Root : "<native-tag>"` (or `Slot` for legacy import), and the JSX MUST render `<Comp ...>` rather than two parallel branches.

R7.3. The cn() className expression MUST wrap BOTH branches identically. Applying className only to the native-tag branch is FAIL.

R7.4. The component MUST forward exactly one child when `asChild` is true. The validator cannot enforce this at the component-source level ; it lives at the call site. The validator MAY flag it in the description as a downstream contract.

R7.5. PASS shape :

```tsx
import { Slot } from "radix-ui"

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

R7.6. FAIL shapes :

```tsx
// asChild accepted but never used
function Button({ asChild = false, ...props }: ButtonProps) {
  return <button {...props} />
}

// Slot not imported
const Comp = asChild ? Slot.Root : "button"   // ReferenceError at runtime

// Different className on each branch
return asChild
  ? <Slot.Root {...props} />
  : <button className={cn(buttonVariants({ variant, size, className }))} {...props} />
```

## Sample-code-to-verdict mapping

The table below maps observed code shapes to the failing check and the canonical fix. The validator MUST emit a verdict block (see SKILL.md § One-glance verdict format) for each FAIL.

| Code shape (observed) | Failing rule | Verdict |
|------------------------|--------------|---------|
| `cva(cn("a", "b"), { ... })` | R1.2 | FAIL Check 1 : remove the cn() wrap ; cva runs clsx internally |
| `cva("a" + " " + "b", { ... })` | R1.1, R1.4 | FAIL Check 1 : use array form `cva(["a", "b"])` |
| `variant: { default: () => "bg-primary" }` | R2.3 | FAIL Check 2 : variant values are strings or string arrays, never functions |
| `variant: { default: { base: "...", hover: "..." } }` | R2.3 | FAIL Check 2 : flatten into a single string ; cva does not recurse |
| `variants: {}, compoundVariants: [...]` declared with compound BEFORE variants | R3.1 | FAIL Check 3 : reorder ; declare variants first |
| `compoundVariants: [{ variant: "ghostly", class: "..." }]` while no `ghostly` variant exists | R3.3 | FAIL Check 3 : fix the match-key typo or add the missing variant value |
| `defaultVariants: { varient: "default" }` (axis typo) | R4.3 | FAIL Check 4 : rename `varient` to `variant` ; cva does not catch this |
| `defaultVariants: { variant: "defualt" }` (value typo) | R4.3 | FAIL Check 4 : rename `defualt` to `default` |
| no `defaultVariants` at all | R4.1 | FAIL Check 4 : add an entry for every axis or set unwanted axes to `null` |
| `export { Button }` while `const buttonVariants = cva(...)` exists | R5.1 | FAIL Check 5 : also export `buttonVariants` |
| `type Props = { variant?: "a" | "b" }` (hand-rolled union) | R5.3 | FAIL Check 5 : replace with `VariantProps<typeof X>` |
| `<button className={\`${variants(...)} ${className}\`} />` | R6.3 | FAIL Check 6 : wrap in cn() and pass className as a key of the variants call |
| `<button className={buttonVariants({ variant, size })} />` (className dropped) | R6.4 | FAIL Check 6 : add className to the variants call OR a second cn() argument |
| `asChild?: boolean` declared but `<button>` rendered unconditionally | R7.1 | FAIL Check 7 : import Slot and branch on asChild |
| `<Slot.Root />` rendered without forwarding the cn() className | R7.3 | FAIL Check 7 : apply the same cn() className to both branches |

## Output contract

For each PASS, emit :

```
VERDICT: PASS
COMPONENT: <file path>:<line of cva call>
SUMMARY: All 7 checks passed.
```

For each FAIL, emit :

```
VERDICT: FAIL
COMPONENT: <file path>:<line of cva call>
FAILED CHECK: Check <n> : <name>
REASON: <one sentence quoting the failing rule R<n>.<m>>
CANONICAL FIX: references/examples.md § <RIGHT-anchor>
```

Multiple failures in the same file MUST emit one block per failure, ordered by check number. The validator never blends multiple verdicts into a single block.
