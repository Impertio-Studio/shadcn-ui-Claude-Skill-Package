# shadcn ui Selectors : Methods Reference

Verbatim prop signatures and primitive surfaces for the four selectors. All entries traced to `apps/v4/registry/new-york-v4/ui/select.tsx`, `ui.shadcn.com/docs/components/radix/select`, `ui.shadcn.com/docs/components/radix/combobox`, `radix-ui.com/primitives/docs/components/select`, and `cmdk.paco.me`.

## Selector 1 : Native `<select>`

### Signature

```tsx
<select
  name: string
  defaultValue?: string
  value?: string
  onChange?: (e: ChangeEvent<HTMLSelectElement>) => void
  multiple?: boolean
  required?: boolean
  disabled?: boolean
  size?: number
  // standard React HTML props
/>
```

### React Hook Form wiring

```tsx
const { register } = useForm()
<select {...register("field")}>...</select>
```

`register("field")` returns `{ name, ref, onChange, onBlur }` and spreads onto the element.

## Selector 2 : shadcn Select (Radix Select wrapped)

### Primitive exports (verbatim from `select.tsx`)

```tsx
export {
  Select,
  SelectContent,
  SelectGroup,
  SelectItem,
  SelectLabel,
  SelectScrollDownButton,
  SelectScrollUpButton,
  SelectSeparator,
  SelectTrigger,
  SelectValue,
}
```

Ten primitives. The shadcn file imports `Select as SelectPrimitive` from the unified `radix-ui` package.

### `Select` (Root) props

| Prop | Type | Default | Purpose |
|------|------|---------|---------|
| `value` | `string` | - | Controlled selected value. PAIR WITH `onValueChange` |
| `defaultValue` | `string` | - | Uncontrolled initial value. EXCLUDES `value` |
| `onValueChange` | `(value: string) => void` | - | Called when selection changes |
| `open` | `boolean` | - | Controlled open state. PAIR WITH `onOpenChange` |
| `defaultOpen` | `boolean` | `false` | Uncontrolled initial open state |
| `onOpenChange` | `(open: boolean) => void` | - | Called when open state changes |
| `name` | `string` | - | Form field name (for native form fallback) |
| `dir` | `"ltr" \| "rtl"` | inherited | Reading direction |
| `disabled` | `boolean` | `false` | Disables the entire Select |
| `required` | `boolean` | `false` | Marks as required for native form validation |

### `SelectTrigger` props

| Prop | Type | Default | Purpose |
|------|------|---------|---------|
| `size` | `"sm" \| "default"` | `"default"` | shadcn-only addition ; sets `data-size` |
| `className` | `string` | - | Tailwind classes ; merged via `cn()` |
| `disabled` | `boolean` | `false` | Disables the trigger |
| `asChild` | `boolean` | `false` | Render via Radix Slot |

The trigger renders a `<ChevronDownIcon>` via `SelectPrimitive.Icon` automatically. `children` is the `<SelectValue>` slot.

### `SelectValue` props

| Prop | Type | Default | Purpose |
|------|------|---------|---------|
| `placeholder` | `ReactNode` | - | Rendered when no value is selected |
| `children` | `ReactNode` | - | Custom render of the selected value |
| `asChild` | `boolean` | `false` | Render via Radix Slot |

### `SelectContent` props

| Prop | Type | Default | Purpose |
|------|------|---------|---------|
| `position` | `"item-aligned" \| "popper"` | `"item-aligned"` | Layout strategy |
| `align` | `"start" \| "center" \| "end"` | `"center"` | Alignment relative to trigger |
| `side` | `"top" \| "right" \| "bottom" \| "left"` | (popper only) | Preferred side (popper mode) |
| `sideOffset` | `number` | `0` | Distance from trigger (popper mode) |
| `alignOffset` | `number` | `0` | Offset along `align` axis |
| `avoidCollisions` | `boolean` | `true` | Flip to opposite side on collision |
| `collisionBoundary` | `Element \| Element[]` | viewport | Boundary for collision detection |
| `collisionPadding` | `number \| Padding` | `0` | Padding inside boundary |
| `sticky` | `"partial" \| "always"` | `"partial"` | Content sticks during scroll |
| `hideWhenDetached` | `boolean` | `false` | Hide when trigger is off-screen |

`SelectContent` auto-wraps itself in `SelectPrimitive.Portal` (verified in select.tsx line ~50). The viewport is `SelectPrimitive.Viewport`, rendered with `p-1` styling. ScrollUp/ScrollDown buttons are rendered automatically.

### `SelectItem` props

| Prop | Type | Default | Purpose |
|------|------|---------|---------|
| `value` | `string` | REQUIRED | The value passed to `onValueChange`. MUST be non-empty string. NEVER `""` |
| `disabled` | `boolean` | `false` | Disables the item |
| `textValue` | `string` | item text | Value used for typeahead matching |
| `asChild` | `boolean` | `false` | Render via Radix Slot |

The item renders a `<CheckIcon>` indicator via `SelectPrimitive.ItemIndicator` when selected (right-aligned, absolute).

### `SelectGroup` / `SelectLabel` / `SelectSeparator`

Pass-through wrappers around `SelectPrimitive.Group`, `Label`, `Separator`. No custom props beyond `className`.

### `SelectScrollUpButton` / `SelectScrollDownButton`

Pass-through wrappers around `SelectPrimitive.ScrollUpButton` / `ScrollDownButton`. Rendered automatically by SelectContent ; you typically do NOT import them directly.

### Form binding signature

```tsx
import { Controller } from "react-hook-form"

<Controller
  control={form.control}
  name="field"
  render={({ field, fieldState }) => (
    <Select value={field.value} onValueChange={field.onChange}>
      <SelectTrigger aria-invalid={fieldState.invalid}>
        <SelectValue placeholder="Pick" />
      </SelectTrigger>
      <SelectContent>...</SelectContent>
    </Select>
  )}
/>
```

`Controller` exposes `field` ( `value`, `onChange`, `onBlur`, `name`, `ref`, `disabled` ) and `fieldState` ( `invalid`, `isDirty`, `isTouched`, `error` ).

## Selector 3 : Combobox

### Surface A : legacy recipe primitives

| Source | Primitive | Role |
|--------|-----------|------|
| `@/components/ui/popover` | `Popover` | Root, owns `open` / `onOpenChange` / `defaultOpen` / `modal` |
| same | `PopoverTrigger` | `asChild`-aware |
| same | `PopoverContent` | Floating surface ; `align`, `side`, `sideOffset` |
| `@/components/ui/command` | `Command` | cmdk root ; `value`, `onValueChange`, `filter`, `shouldFilter`, `loop` |
| same | `CommandInput` | Search input |
| same | `CommandList` | Scrollable list container |
| same | `CommandEmpty` | Rendered when filter matches zero items |
| same | `CommandGroup` | Section header + items ; `heading`, `value`, `forceMount` |
| same | `CommandItem` | One option ; `value`, `onSelect(value)`, `disabled`, `forceMount` |
| same | `CommandLoading` | Async loading row ; renders during in-flight requests |
| same | `CommandSeparator` | Horizontal rule |
| same | `CommandShortcut` | Keyboard shortcut hint (right-aligned text) |

### Popover root props (load-bearing for Combobox)

| Prop | Type | Default | Purpose |
|------|------|---------|---------|
| `open` | `boolean` | - | Controlled open state |
| `onOpenChange` | `(open: boolean) => void` | - | Called on toggle |
| `defaultOpen` | `boolean` | `false` | Uncontrolled initial |
| `modal` | `boolean` | `false` | **REQUIRED `true` for Combobox** ; traps focus inside list |

NEVER omit `modal={true}` in a Combobox recipe. Without it, clicking an item lets focus escape before `onOpenChange(false)` fires ; the popover stays open and unresponsive.

### CommandItem `onSelect` signature

```tsx
<CommandItem
  value={string}            // value used for cmdk filter matching
  onSelect={(value: string) => void}
  disabled={boolean}
  forceMount={boolean}      // keep mounted when filter would hide it
/>
```

The `onSelect` callback receives the item's `value` prop (lowercased by cmdk by default). NEVER assume `event.currentTarget` ; cmdk passes only the value string.

### Surface B : 2026 Combobox primitive props

Root `<Combobox>` :

| Prop | Type | Purpose |
|------|------|---------|
| `items` | `T[]` | The source list |
| `value` | `T \| T[]` | Controlled selected value ; array when `multiple` |
| `onValueChange` | `(value: T \| T[]) => void` | Called on selection |
| `defaultValue` | `T \| T[]` | Uncontrolled initial |
| `multiple` | `boolean` | Multi-select mode |
| `itemToStringValue` | `(item: T) => string` | Maps item to a string for filtering |
| `showClear` | `boolean` | Render a clear button |
| `autoHighlight` | `boolean` | Highlight first match on open |
| `render` | render-prop | Custom render shape |

Subcomponents : `ComboboxInput`, `ComboboxContent`, `ComboboxList`, `ComboboxItem`, `ComboboxEmpty`, `ComboboxGroup`, `ComboboxLabel`, `ComboboxCollection`, `ComboboxChips`, `ComboboxChipsInput`.

### Form binding (Controller)

Same shape as Select : wrap the entire Popover (or Combobox primitive) in a Controller, let `field.onChange` write the value, keep local `open` state outside the form.

## Selector 4 : Command palette

### Primitive exports

| Primitive | Purpose |
|-----------|---------|
| `Command` | Inline cmdk root ; NOT a palette ; used inside Combobox |
| `CommandDialog` | Palette wrapper ; combines `<Dialog>` + `<Command>` ; portal + focus trap + sr-only DialogTitle |
| `CommandInput` | Search input |
| `CommandList` | Scrollable list |
| `CommandEmpty` | No-results state |
| `CommandGroup` | Section with `heading` |
| `CommandItem` | Action item ; `value`, `onSelect`, `disabled` |
| `CommandLoading` | Async loading row |
| `CommandSeparator` | Horizontal rule |
| `CommandShortcut` | Right-aligned shortcut hint |

### CommandDialog props

| Prop | Type | Default | Purpose |
|------|------|---------|---------|
| `open` | `boolean` | - | Controlled open state |
| `onOpenChange` | `(open: boolean) => void` | - | Called on toggle |
| `title` | `string` | `"Command Palette"` | Accessible name (sr-only DialogTitle by default) |
| `description` | `string` | - | Optional sr-only DialogDescription |
| `showCloseButton` | `boolean` | `true` | Toggle the built-in close X |
| `container` | `HTMLElement` | `document.body` | Portal target override |

### Global hotkey wiring (canonical pattern)

```tsx
React.useEffect(() => {
  const onKey = (e: KeyboardEvent) => {
    if (e.key === "k" && (e.metaKey || e.ctrlKey)) {
      e.preventDefault()
      setOpen((o) => !o)
    }
  }
  document.addEventListener("keydown", onKey)
  return () => document.removeEventListener("keydown", onKey)
}, [])
```

ALWAYS check BOTH `metaKey` (macOS) and `ctrlKey` (Windows/Linux). ALWAYS call `e.preventDefault()` to suppress the browser's default behavior on Cmd+K (focus the address bar in some browsers). ALWAYS clean up the listener in the effect's return.

## Form Controller path per selector

| Selector | Controller? | `field.onChange` consumed by | `field.value` rendered by |
|----------|-------------|------------------------------|---------------------------|
| Native `<select>` | NO (use `register`) | native `onChange` event | native `value` / `defaultValue` |
| shadcn Select | YES | `onValueChange` on `<Select>` | `value` on `<Select>` |
| Combobox (recipe A) | YES | `onSelect` on each `<CommandItem>` | trigger label render |
| Combobox (primitive B) | YES | `onValueChange` on `<Combobox>` | `value` on `<Combobox>` |
| Command palette | N/A | item `onSelect` fires a navigation / action | n/a (palette is not a form input) |

## Verification

- `select.tsx` SHA `c0dc7120b4f78008ae5efc6d1c8b7ec8205c35d5` on `main` branch of `shadcn-ui/ui`, fetched 2026-05-19.
- 10 primitives confirmed in the `export { ... }` block.
- `position = "item-aligned"` is the default in `SelectContent`, confirmed verbatim.
- `align = "center"` is the default in `SelectContent`, confirmed verbatim.
- `SelectPrimitive.Portal` wraps `SelectPrimitive.Content` inside the shadcn `SelectContent` function ; no manual Portal needed.
- `size = "default"` is the SelectTrigger default ; valid values `"sm" | "default"` ; written to `data-size` attribute.
