---
name: shadcn-errors-styling-conflicts
description: >
  Use when a className override on a shadcn component does not visually take effect,
  when two Tailwind utility classes appear in the rendered DOM but only one applies,
  when a cva variant looks correct in code but the consumer className does not win,
  when porting code from Tailwind v3 to v4 and class semantics shift, when adding
  arbitrary values (bg-[#fff], size-[28px]) next to preset utilities, when the
  important modifier (! prefix in v3, suffix in v4) behaves unexpectedly, when a
  component built with cva looks unstyled or doubled-up, or when reviewing PRs that
  build className strings with template literals or string concatenation. Prevents
  the canonical class-merging bugs: skipping cn() so duplicates leak through the
  DOM, calling cn() inside cva (no-op since cva already runs clsx), assuming the
  important modifier always overrides plain utilities, mixing arbitrary properties
  with their preset equivalents, and applying v3-only class names in a v4 project.
  Covers the cn() helper rule (clsx + tailwind-merge wrapper), tailwind-merge
  semantics (last-wins, arbitrary-value handling, important-tracking, dedup), the
  exact cva merge order (base then variants then compoundVariants then
  defaultVariants then consumer className), the Tailwind v3 to v4 class-semantics
  shifts that surface as merge failures, the !important precedence trap, and the
  six most common class-merging anti-patterns observed in shadcn projects.
  Keywords: tailwind-merge, twMerge, className not applied, className ignored, cn
  helper, cva variant override, duplicate tailwind classes, why is my className
  ignored, class conflict, tailwind class conflict, tailwind-merge surprise,
  arbitrary classes not merging, bg-[#fff] vs bg-white, important class merge,
  !important class merge, p-4 p-2 merge, last wins tailwind, my override does not
  work, my styles do not apply, cva default variant override, compoundVariants
  ignored, shadcn className not working, template string className, raw className
  concat, missing twMerge, clsx alone, ring-2 vs ring, size-4 vs w-4 h-4.
license: MIT
compatibility: "Designed for Claude Code. Requires shadcn ui evergreen-2026."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# shadcn ui Errors: Styling and Class-Merging Conflicts

When a className override on a shadcn component appears in the rendered DOM but does not affect the visible style, the cause is almost always one of three things: (1) the cn() helper was skipped and duplicate Tailwind utilities leaked through, (2) the cva merge order was misunderstood and the consumer className landed before the variant class instead of after it, or (3) Tailwind v3 and v4 class semantics were mixed in the same file. This skill enumerates the deterministic rules and the verified merge order so the diagnosis takes seconds.

## Quick Reference

### The cn() rule (memorize this)

```ts
// lib/utils.ts : every shadcn project ships this file
import { clsx, type ClassValue } from "clsx"
import { twMerge } from "tailwind-merge"
export function cn(...inputs: ClassValue[]) { return twMerge(clsx(inputs)) }
```

- ALWAYS use `cn(...)` when composing className from more than one source.
- NEVER concatenate className with `+` or template literals. Raw strings bypass conflict resolution and let duplicates leak through.
- NEVER use `clsx` alone for className that may collide. clsx joins conditionals but does NOT resolve Tailwind conflicts.
- NEVER use `twMerge` alone when you need conditionals. twMerge does not understand object syntax or falsy values; that is clsx's job.
- The order inside `cn(...)` is significant : LAST WINS for conflicting Tailwind utilities.

### tailwind-merge semantics (verified at github.com/dcastil/tailwind-merge)

| Input | Output | Reason |
|---|---|---|
| `twMerge('p-4 p-2')` | `p-2` | Same utility family : last wins |
| `twMerge('p-5 p-2 p-4')` | `p-4` | Last wins across any chain length |
| `twMerge('p-3 px-5')` | `p-3 px-5` | Non-conflicting refinements coexist |
| `twMerge('px-2 p-3')` | `p-3` | Broader utility (`p-*`) overrides narrower (`px-*`) |
| `twMerge('inset-x-px -inset-1')` | `-inset-1` | Broader override even with sign change |
| `twMerge('bg-[#fff] bg-white')` | `bg-white` | Arbitrary VALUES conflict with presets; last wins |
| `twMerge('[mask-type:luminance] [mask-type:alpha]')` | `[mask-type:alpha]` | Arbitrary PROPERTIES conflict with each other |
| `twMerge('[padding:1rem] p-4')` | `[padding:1rem] p-4` | Arbitrary PROPERTIES do NOT resolve against matching utilities (kept on purpose for bundle size) |
| `twMerge('p-3! p-4! p-5')` | `p-4! p-5` | Important variants are tracked in a SEPARATE namespace; non-important `p-5` does not displace `p-4!` |
| `twMerge('hover:focus:p-2 focus:hover:p-4')` | `focus:hover:p-4` | Stacked modifiers are normalized before comparison |
| `twMerge('p-5 p-2 my-custom p-4')` | `my-custom p-4` | Non-Tailwind classes are preserved unchanged |

The two non-obvious rules: (1) `[mask-type:luminance] p-4` does not resolve `[padding:1rem]` against `p-4`, so mixing arbitrary properties with their preset equivalents leaks both; (2) `!` (v3) and trailing `!` (v4) classes are tracked separately, so a non-important utility CANNOT override an important one of the same family even when it appears later.

### cva merge order (the canonical chain)

A cva-built component composes classes in this exact order:

```
[1] base classes        (the first cva argument)
[2] variants            (selected via { variant, size, ... } props)
[3] compoundVariants    (matched when ALL listed props match)
[4] defaultVariants     (applied when a variant prop is undefined)
[5] consumer className  (whatever the caller passed)
```

Everything from [1] through [5] is fed through `twMerge` exactly once, in that order, via the standard call site:

```tsx
className={cn(buttonVariants({ variant, size }), className)}
```

Because `twMerge` is last-wins, the consumer `className` always wins against the cva output for conflicting utilities. If a consumer override does NOT win, the call site is almost certainly missing `cn()` or has the cva output AFTER the consumer className.

### Top-5 visible symptoms and their root cause

| Symptom | Almost always | Fix |
|---|---|---|
| Two `p-*` classes in DOM, one wins, other ignored | cn() bypassed via template string | Wrap in `cn(...)` |
| Consumer `className` has zero visual effect | className passed BEFORE variant in cn() order, or cn() not used at all | Put consumer className LAST in cn() |
| `!bg-red-500` not winning over `bg-blue-500` | Mixed v3 and v4 important syntax in one file | Pick one syntax per project; v4 uses trailing `!` |
| cva variant only sometimes applies | `defaultVariants` set, but consumer passes `variant=""` (empty string, not undefined) | Treat unset variants as `undefined`, not `""` |
| Style flip on Tailwind v4 upgrade | Class semantics changed (see v3 to v4 table below) | Re-verify ring-*, size-*, space-* usage |

### Tailwind v3 to v4 class-semantics shifts that surface as merge bugs

| Behaviour | v3 | v4 |
|---|---|---|
| Default ring width | `ring` = 3px | `ring` = 1px (use `ring-3` for old default) |
| Important syntax | `!bg-red-500` (prefix) | `bg-red-500!` (suffix) |
| Square sizing | `w-4 h-4` (two utilities) | `size-4` (single utility ; tailwind-merge knows both shapes) |
| HSL token consumption | `bg-[hsl(var(--background))]` | `bg-background` (CSS var exposed via `@theme inline`) |
| Color-space default | HSL via `tailwind.config.js` | oklch via `@theme` in CSS |
| Animation plugin | `tailwindcss-animate` | `tw-animate-css` (CSS import) |

These are not tailwind-merge bugs : they are version-tagged class identities. Mixing them in one file silently produces double-up classes that `twMerge` cannot reconcile because they target different design tokens.

## Decision Tree : "Why is my className not applied?"

1. Open the rendered DOM. Are BOTH classes present?
   - YES, both present : cn() was bypassed. Search for `className=` near the component and convert any `"foo " + bar` or `` `foo ${bar}` `` into `cn("foo", bar)`. Go to Pattern A.
   - NO, only one present, but the wrong one wins : cn() ran, but order is wrong. Go to step 2.
   - NO, both missing : the component is not receiving className. Go to step 5.
2. Read the call site. Where is `className` in the cn() argument list?
   - LAST : the merge order is correct. Go to step 3.
   - Not last : reorder so the consumer className is the final argument. See Pattern B.
3. Does the conflicting class use the important modifier (`!` prefix in v3, suffix in v4)?
   - YES : important classes are tracked in a separate namespace by tailwind-merge. A plain class cannot override an important class of the same family. Use the matching important syntax or remove the original `!`. See Pattern D.
   - NO : go to step 4.
4. Is one of the classes an arbitrary PROPERTY (`[padding:1rem]`) rather than an arbitrary VALUE (`p-[1rem]`)?
   - YES : arbitrary properties do NOT resolve against matching Tailwind utilities. Switch to the arbitrary-value form (`p-[1rem]`) or drop one of the two. See Pattern C.
   - NO : check Tailwind version. If the project upgraded to v4, verify the class still means what it meant in v3 (see table above).
5. The component never receives the className. Open the component source. Does it accept and forward `className`?
   - NO : add `className?: string` to props and forward via `cn(componentVariants(...), className)`.
   - YES, but cva is the issue : see Pattern E in references/methods.md for the cva variant-override walk-through.

## Patterns : WRONG vs RIGHT

### Pattern A : Template-string concatenation leaks duplicates

```tsx
// WRONG : both p-4 and p-2 reach the DOM ; visual outcome is undefined and varies per Tailwind class order on disk.
<div className={"p-4 " + (compact && "p-2")} />

// RIGHT : cn() runs tailwind-merge ; p-2 wins when compact is true.
<div className={cn("p-4", compact && "p-2")} />
```

### Pattern B : Consumer className must be LAST in cn()

```tsx
// WRONG : caller's className is overwritten by variant defaults.
className={cn(className, buttonVariants({ variant, size }))}

// RIGHT : variant first, caller last, last-wins gives caller the override.
className={cn(buttonVariants({ variant, size }), className)}
```

### Pattern C : Arbitrary property does not resolve against utility

```tsx
// WRONG : both classes reach the DOM ; tailwind-merge intentionally skips this case.
<div className={cn("[padding:1rem]", "p-4")} />

// RIGHT : use the arbitrary-VALUE form (same family as p-4).
<div className={cn("p-[1rem]", "p-4")} />   // → p-4 wins
```

### Pattern D : Important modifier is tracked separately

```tsx
// WRONG (Tailwind v4) : the plain bg-blue-500 cannot override bg-red-500! because important lives in a separate namespace.
<div className={cn("bg-red-500!", "bg-blue-500")} />   // → "bg-red-500! bg-blue-500"

// RIGHT : either match the important namespace, or remove the !
<div className={cn("bg-red-500!", "bg-blue-500!")} />  // → "bg-blue-500!"
<div className={cn("bg-red-500", "bg-blue-500")} />    // → "bg-blue-500"
```

### Pattern E : cn() inside cva is a no-op

```ts
// WRONG : cva already runs clsx internally on the variant value ; wrapping with cn() does nothing useful and obscures intent.
const button = cva("...", {
  variants: { tone: { brand: cn("bg-primary", "text-primary-foreground") } }
})

// RIGHT : pass plain strings (or string arrays) to cva variants ; let cn() run ONCE at the call site.
const button = cva("...", {
  variants: { tone: { brand: "bg-primary text-primary-foreground" } }
})
```

## Companion Skills

- `shadcn-syntax-variant-cva` (Batch 3) : full cva API, VariantProps inference, compoundVariants matching rules. Read this first when designing a new component.
- `shadcn-core-stack` (Batch 2) : the runtime layer map (Radix + cva + tailwind-merge + clsx + Tailwind + lucide). Read this to understand which library owns which concern.
- `shadcn-errors-tailwind-v3-v4-migration` (Batch 12) : the full HSL-to-oklch and config-to-CSS shift. Read this when an upgrade introduces unexpected style flips.

## Reference Files

- `references/methods.md` : the `cn()` implementation, tailwind-merge config knobs (`extendTailwindMerge`, custom class groups), and the full cva-to-consumer merge-order walkthrough.
- `references/examples.md` : end-to-end WRONG-vs-RIGHT snippets for every pattern above, plus a v3 to v4 class rename impact table.
- `references/anti-patterns.md` : six verified anti-patterns with root-cause analysis and the deterministic fix for each.

## Verified Sources

- https://github.com/dcastil/tailwind-merge (merge semantics, important-modifier tracking, arbitrary-value vs arbitrary-property rule)
- https://github.com/dcastil/tailwind-merge/blob/main/docs/features.md (full rule table)
- https://cva.style/docs (variant signature, `class` parameter merge position)
- https://beta.cva.style (compose() and consumer-class handling)
- https://tailwindcss.com/docs/upgrade-guide (v3 to v4 class-semantics shifts)
- https://ui.shadcn.com/docs (cn() canonical helper in `lib/utils.ts`)

Last verified: 2026-05-19.
