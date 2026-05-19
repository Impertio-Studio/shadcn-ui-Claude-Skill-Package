# Methods Reference : cva, VariantProps, cn, Slot

All signatures verified at the listed sources on 2026-05-19.

## cva

Source : https://cva.style/docs/api-reference

### Signature

```ts
import type { ClassValue } from "clsx"

type ClassPropKey = "class" | "className"

type ClassProp =
  | { class: ClassValue;     className?: never }
  | { class?: never;         className: ClassValue }
  | { class?: never;         className?: never }

type ConfigSchema = Record<string, Record<string, ClassValue>>

type ConfigVariants<T extends ConfigSchema> = {
  [Variant in keyof T]?: keyof T[Variant] | null | undefined
}

type Config<T extends ConfigSchema> = {
  variants?: T
  compoundVariants?: (T extends ConfigSchema
    ? (ConfigVariants<T> & ClassProp)[]
    : ClassProp[])
  defaultVariants?: ConfigVariants<T>
}

declare function cva<T extends ConfigSchema>(
  base: ClassValue | ClassValue[],
  config?: Config<T>
): (props?: ConfigVariants<T> & ClassProp) => string
```

### Parameter notes

- **base** : a `ClassValue` (string, array, object, or any value `clsx` accepts) OR an array of `ClassValue`. Classes here ALWAYS apply.
- **config.variants** : a record. Each top-level key is one variant axis ; each leaf maps a value-name to a `ClassValue` (string or string-array).
- **config.compoundVariants** : an array. Each entry is an object combining one or more variant-key matches plus a `class` (or `className`) field carrying the classes to add when ALL listed keys match.
- **config.defaultVariants** : a record of variant-axis to default value-name. Setting an axis to `null` opts out of any default.

### Return value

The cva call returns a function that accepts `(props?)` of type `ConfigVariants<T> & ClassProp` and returns a `string`. Invocation looks like :

```ts
buttonVariants()                              // returns base + defaults
buttonVariants({ variant: "outline" })        // base + outline + size-default
buttonVariants({ size: "sm", className: "x" }) // base + variant-default + sm + "x"
```

The `class` and `className` props are mutually exclusive in the return-call's argument ; pass ONE or NEITHER.

### cx export (alias for clsx)

cva also exports `cx`, which is a re-export of `clsx`. It is rarely used directly because `cn()` already wraps clsx ; use `cn()` for all consumer code.

## VariantProps

Source : https://cva.style/docs/getting-started/typescript

### Signature

```ts
import type { cva } from "class-variance-authority"

type VariantProps<Component extends (...args: any) => any> = Omit<
  Parameters<Component>[0],
  "class" | "className"
>
```

### Usage

```ts
const button = cva("base", {
  variants: {
    variant: { default: "...", outline: "..." },
    size: { sm: "...", lg: "..." },
  },
})

type ButtonVariantProps = VariantProps<typeof button>
//   ^ {
//       variant?: "default" | "outline" | null | undefined
//       size?: "sm" | "lg" | null | undefined
//     }
```

`VariantProps` strips the `class` / `className` keys from the parameters tuple and exposes only the variant axes. Each axis is optional and accepts the literal union of value-keys, plus `null` (explicit opt-out) and `undefined` (use default).

### Required variant pattern

To force a single variant to be required while keeping the rest optional :

```ts
type RequiredButtonProps =
  Omit<VariantProps<typeof button>, "variant"> &
  Required<Pick<VariantProps<typeof button>, "variant">>
```

## cn

Source : https://github.com/dcastil/tailwind-merge and the shadcn CLI scaffold (file `lib/utils.ts` produced by `shadcn init`).

### Signature

```ts
import { clsx, type ClassValue } from "clsx"
import { twMerge } from "tailwind-merge"

export function cn(...inputs: ClassValue[]): string {
  return twMerge(clsx(inputs))
}
```

### Behaviour layers

1. **clsx** : accepts strings, arrays, objects with boolean values, and `undefined`/`null`/`false`. Flattens and joins truthy strings with a space.
2. **twMerge** : parses the joined string for known Tailwind utilities, identifies pairs that target the same CSS property, drops earlier-position losers, and returns the deduplicated string.

### Conflict examples (verified at https://github.com/dcastil/tailwind-merge)

```ts
twMerge("px-2 py-1 bg-red hover:bg-dark-red", "p-3 bg-[#B91C1C]")
// "hover:bg-dark-red p-3 bg-[#B91C1C]"
//  ↑ pseudo preserved   ↑ last-write wins   ↑ arbitrary value wins
```

### Tailwind version support

`tailwind-merge` supports Tailwind v4.0 through v4.3 (current). It also remains compatible with Tailwind v3.4 ; the merge knowledge ships with the library so no consumer config is required for default Tailwind utilities. Custom theme extensions require `extendTailwindMerge` or `createTailwindMerge` (out of scope for this skill ; see https://github.com/dcastil/tailwind-merge#tailwind-merge-internals).

## Slot

Source : https://www.radix-ui.com/primitives/docs/utilities/slot

### Signature (post Feb-2026 unified `radix-ui` package)

```ts
import { Slot } from "radix-ui"

// Slot.Root : merges its single child with the props passed to Slot
declare const SlotRoot: React.ForwardRefExoticComponent<
  React.HTMLAttributes<HTMLElement> & React.RefAttributes<HTMLElement>
>

// Slot.Slottable : marks a child as the slot target inside Slot.Root
declare const SlottableSlot: React.FC<{ children: React.ReactNode }>
```

### Signature (pre Feb-2026, individual package)

```ts
import { Slot } from "@radix-ui/react-slot"
// Same surface : Slot (root), Slottable (child marker)
```

### Behaviour

- `<Slot.Root>` MUST receive exactly ONE React child. The Slot copies its own props (including `className`, `onClick`, ref) onto that child via `React.cloneElement` + ref-merging.
- The child's existing props win over Slot props for `className` (concatenated, not overridden) and lose for event handlers (Slot's handler runs first, then the child's).
- Passing zero or multiple children throws : `React.Children.only expected to receive a single React element child`.

### Usage in shadcn components

```tsx
const Comp = asChild ? Slot.Root : "button"
return <Comp className={cn(buttonVariants({ variant, size, className }))} {...props} />
```

This pattern lets the component render either a native `<button>` (the default) or whatever React element the caller passes as the single child (when `asChild` is true).
