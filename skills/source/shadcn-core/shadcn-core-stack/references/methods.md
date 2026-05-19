# shadcn ui Stack: Complete API Reference

Verified 2026-05-19.

## class-variance-authority (cva)

### Package
- npm : `class-variance-authority`
- Current stable : `0.7.1`
- Current beta : `1.0.0-beta` (mostly source-compatible upgrade)
- Import path : `class-variance-authority`

### Signature

```ts
function cva<T>(
  base: ClassValue,
  config?: {
    variants?: { [K in keyof T]: { [variant: string]: ClassValue } }
    compoundVariants?: Array<
      { [K in keyof T]?: T[K] | T[K][] } & { class: ClassValue }
    >
    defaultVariants?: { [K in keyof T]?: T[K] | null }
  }
): (props?: { [K in keyof T]?: T[K] | null } & { class?: ClassValue, className?: ClassValue }) => string
```

### Parameters

| Field | Type | Purpose |
|-------|------|---------|
| `base` | `ClassValue` | Classes ALWAYS applied (the foundation) |
| `variants` | `Record<variantName, Record<variantValue, ClassValue>>` | Variant families, one per prop |
| `compoundVariants` | `Array<{ ...variantsMatch, class: ClassValue }>` | Extra classes for variant combinations |
| `defaultVariants` | `Record<variantName, variantValue \| null>` | Default values when prop is `undefined` |

### `VariantProps<typeof fn>`

Extracts the typed prop interface from a cva return :

```ts
import { cva, type VariantProps } from "class-variance-authority"
const button = cva("base", { variants: { intent: { primary: "...", danger: "..." } } })
type ButtonVariants = VariantProps<typeof button>
// => { intent?: "primary" | "danger" | null }
```

Optional props (`?`) and `null`-accepting (`| null`) are forced by cva. Pass `intent={null}` to skip the default.

### Caller-supplied `class` / `className`

```ts
button({ intent: "primary", class: "extra-utility" })
button({ intent: "primary", className: "extra-utility" })
```

Both work, both append to the output. shadcn convention is to NOT use this and instead wrap with `cn()` for twMerge conflict resolution.

## tailwind-merge

### Package
- npm : `tailwind-merge`
- For Tailwind v3.4 : `tailwind-merge@^2.6.0`
- For Tailwind v4.0 to v4.3 : `tailwind-merge@^3.0.0`
- Import path : `tailwind-merge`

### `twMerge`

```ts
import { twMerge } from "tailwind-merge"

twMerge(...inputs: string[]): string
```

Returns the concatenated input with later utilities winning over earlier conflicting utilities.

```ts
twMerge("px-2 py-1 bg-red-500", "p-3 bg-blue-500")
// => "p-3 bg-blue-500"
twMerge("hover:bg-red-500 hover:p-2", "hover:bg-blue-500")
// => "hover:p-2 hover:bg-blue-500" (variants are scoped per-variant)
```

### `twJoin`

```ts
import { twJoin } from "tailwind-merge"
twJoin("a", null, undefined, false, "b") // => "a b"
```

Like `clsx` but only string concatenation without conflict resolution. Rarely used directly in shadcn projects (cn already wraps clsx + twMerge).

### `extendTailwindMerge`

```ts
import { extendTailwindMerge } from "tailwind-merge"

const twMerge = extendTailwindMerge({
  extend: {
    classGroups: {
      shadow: ["my-custom-shadow"],
      "font-size": ["text-fluid"],
    },
  },
})
```

Use when registering custom utility classes (from plugins or `@utility` declarations) so twMerge can resolve their conflicts.

### `createTailwindMerge`

```ts
import { createTailwindMerge, getDefaultConfig } from "tailwind-merge"
const myMerge = createTailwindMerge(getDefaultConfig)
```

Use for completely custom merge configurations (renames, custom prefixes, theme-specific groups).

## clsx

### Package
- npm : `clsx`
- Import path : `clsx`

### Signature

```ts
import { clsx, type ClassValue } from "clsx"

type ClassValue = ClassArray | ClassDictionary | string | number | bigint | null | boolean | undefined
type ClassDictionary = Record<string, any>
type ClassArray = ClassValue[]

function clsx(...inputs: ClassValue[]): string
```

Falsy values are dropped. Object keys become classes when their value is truthy.

```ts
clsx("a", true && "b", false && "c", null, { d: 1, e: 0 }, ["f", "g"])
// => "a b d f g"
```

## The `cn()` Helper (shadcn-canonical)

### Source

```ts
// lib/utils.ts
import { clsx, type ClassValue } from "clsx"
import { twMerge } from "tailwind-merge"

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs))
}
```

### Signature

```ts
function cn(...inputs: ClassValue[]): string
```

### Order semantics

twMerge resolves left-to-right : later utilities of the SAME group override earlier ones. Pass caller-supplied `className` LAST so callers can override component defaults.

## Radix UI Primitives

### Two package shapes

#### Legacy per-component (pre-Feb 2026)

```bash
npm install @radix-ui/react-dialog @radix-ui/react-dropdown-menu
```

```ts
import * as DialogPrimitive from "@radix-ui/react-dialog"
import * as DropdownMenuPrimitive from "@radix-ui/react-dropdown-menu"
```

#### Unified (since Feb 2026)

```bash
npm install radix-ui
```

```ts
import { Dialog, DropdownMenu, Popover } from "radix-ui"

// Or namespace-import individual primitives
import { Dialog as DialogPrimitive } from "radix-ui"
```

### Primitive catalog (39 primitives)

| Primitive | Subcomponents |
|-----------|---------------|
| `Accordion` | Root, Item, Header, Trigger, Content |
| `AlertDialog` | Root, Trigger, Portal, Overlay, Content, Title, Description, Action, Cancel |
| `AspectRatio` | Root |
| `Avatar` | Root, Image, Fallback |
| `Checkbox` | Root, Indicator |
| `Collapsible` | Root, Trigger, Content |
| `ContextMenu` | Root, Trigger, Portal, Content, Item, CheckboxItem, RadioGroup, RadioItem, ItemIndicator, TriggerItem, Group, Label, Separator, Arrow, Sub, SubTrigger, SubContent |
| `Dialog` | Root, Trigger, Portal, Overlay, Content, Title, Description, Close |
| `Direction` | Provider |
| `DropdownMenu` | Root, Trigger, Portal, Content, Arrow, Item, Group, Label, CheckboxItem, RadioGroup, RadioItem, ItemIndicator, Separator, Sub, SubTrigger, SubContent |
| `Form` | Root, Field, Label, Control, Message, Submit, ValidityState |
| `HoverCard` | Root, Trigger, Portal, Content, Arrow |
| `Label` | Root |
| `Menubar` | Root, Menu, Trigger, Portal, Content, Arrow, Item, Group, Label, CheckboxItem, RadioGroup, RadioItem, ItemIndicator, Separator, Sub, SubTrigger, SubContent |
| `NavigationMenu` | Root, List, Item, Trigger, Content, Link, Indicator, Viewport |
| `OneTimePasswordField` | (newer Radix primitive ; shadcn uses `input-otp` instead) |
| `Popover` | Root, Trigger, Anchor, Portal, Content, Close, Arrow |
| `Progress` | Root, Indicator |
| `RadioGroup` | Root, Item, Indicator |
| `ScrollArea` | Root, Viewport, Scrollbar, Thumb, Corner |
| `Select` | Root, Trigger, Value, Icon, Portal, Content, Viewport, Item, ItemText, ItemIndicator, Group, Label, Separator, Arrow, ScrollUpButton, ScrollDownButton |
| `Separator` | Root |
| `Slider` | Root, Track, Range, Thumb |
| `Slot` | Slot, Slottable |
| `Switch` | Root, Thumb |
| `Tabs` | Root, List, Trigger, Content |
| `Toast` (legacy) | Provider, Root, Viewport, Title, Description, Action, Close (deprecated ; use `sonner`) |
| `Toggle` | Root |
| `ToggleGroup` | Root, Item |
| `Toolbar` | Root, Button, Link, ToggleGroup, ToggleItem, Separator |
| `Tooltip` | Provider, Root, Trigger, Portal, Content, Arrow |
| `VisuallyHidden` | Root |

The primitives marked `(deprecated)` or `(newer)` should be cross-checked against the latest Radix docs at install time.

### Common Radix-shared props

| Prop | Type | Purpose |
|------|------|---------|
| `asChild` | `boolean` | Render via Slot composition instead of an extra DOM node |
| `open` / `onOpenChange` | `boolean` / `(open: boolean) => void` | Controlled mode (Dialog, Popover, DropdownMenu, etc.) |
| `defaultOpen` | `boolean` | Uncontrolled initial state |
| `forceMount` | `boolean` | Render content even when closed (for custom animations) |
| `modal` | `boolean` | Trap focus and block background interaction |
| `loop` | `boolean` | Wrap keyboard navigation at list ends |
| `dir` | `"ltr" \| "rtl"` | RTL support (also provided globally by Direction.Provider) |

## lucide-react

### Package
- npm : `lucide-react`
- Import path : `lucide-react`

### Usage

```ts
import { Check, ChevronRight, Loader2, X } from "lucide-react"
```

Each import is tree-shaken individually. The default size in shadcn convention is `size-4` (16px). Animated `<Loader2 className="animate-spin" />` is the canonical loading-spinner.

### Migrating away

The shadcn CLI provides `pnpm dlx shadcn migrate icons` to swap lucide for another icon library while preserving all uses in your codebase.

## Tailwind CSS

### Versions

| Version | Required for |
|---------|--------------|
| v3.4+ | shadcn evergreen-2026 with `tailwind-merge@^2.6.0` |
| v4.0+ | shadcn evergreen-2026 with `tailwind-merge@^3.0.0` |

### Theme tokens used by every shadcn component

shadcn's `theme.css` (v4) or `globals.css` (v3) defines CSS variables consumed via theme colors :

```css
:root {
  --background: oklch(1 0 0);
  --foreground: oklch(0.15 0.02 270);
  --primary: oklch(0.65 0.196 254);
  --primary-foreground: oklch(0.98 0.005 254);
  --secondary: oklch(0.96 0.005 250);
  --secondary-foreground: oklch(0.15 0.02 270);
  --destructive: oklch(0.65 0.21 30);
  --destructive-foreground: oklch(0.98 0.005 30);
  --muted: oklch(0.96 0.005 250);
  --muted-foreground: oklch(0.55 0.02 270);
  --accent: oklch(0.96 0.005 250);
  --accent-foreground: oklch(0.15 0.02 270);
  --border: oklch(0.91 0.01 250);
  --input: oklch(0.91 0.01 250);
  --ring: oklch(0.65 0.196 254);
  --radius: 0.5rem;
}
```

cva-driven components reference these via `bg-primary`, `text-primary-foreground`, `border-input`, etc.
