# Command : Full API Reference

Verbatim signatures sourced from `apps/v4/registry/new-york-v4/ui/command.tsx` (shadcn-ui/ui, evergreen-2026 registry) and https://cmdk.paco.me (cmdk README, the underlying library that shadcn wraps). Verified 2026-05-19.

## File header

```tsx
"use client"

import * as React from "react"
import { Command as CommandPrimitive } from "cmdk"
import { SearchIcon } from "lucide-react"

import { cn } from "@/lib/utils"
import {
  Dialog,
  DialogContent,
  DialogDescription,
  DialogHeader,
  DialogTitle,
} from "@/registry/new-york-v4/ui/dialog"
```

`"use client"` is REQUIRED. cmdk uses `useId`, `useSyncExternalStore`, and manual DOM ordering ; it is a hard client-only library. Note that shadcn imports the cmdk primitive AS `CommandPrimitive`, NOT from `radix-ui`. cmdk is independent of the Radix unified package.

## 1. Command (Root)

```tsx
function Command({
  className,
  ...props
}: React.ComponentProps<typeof CommandPrimitive>) {
  return (
    <CommandPrimitive
      data-slot="command"
      className={cn(
        "flex h-full w-full flex-col overflow-hidden rounded-md bg-popover text-popover-foreground",
        className
      )}
      {...props}
    />
  )
}
```

### Props (forwarded to cmdk `Command.Root`)

| Prop | Type | Default | Notes |
|------|------|---------|-------|
| `value` | `string` | - | Controlled HIGHLIGHTED item value. Pair with `onValueChange`. NOT the search query. |
| `onValueChange` | `(value: string) => void` | - | Fires when the highlighted item changes (arrow keys, hover, programmatic). |
| `defaultValue` | `string` | - | Uncontrolled initial highlight. EXCLUDES `value`. |
| `shouldFilter` | `boolean` | `true` | When `false`, cmdk renders every CommandItem verbatim ; you filter externally. |
| `filter` | `(value: string, search: string, keywords?: string[]) => number` | built-in fuzzy | Custom ranker. Return 0 to hide. Higher number = higher rank. |
| `loop` | `boolean` | `false` | When `true`, ArrowDown past the last item wraps to the first. |
| `label` | `string` | - | Accessible name for the listbox. Ignored when inside CommandDialog. |
| `vimBindings` | `boolean` | `true` | When `true`, Ctrl+N / Ctrl+P / Ctrl+J / Ctrl+K also move selection. |
| `disablePointerSelection` | `boolean` | `false` | When `true`, pointer hover does NOT change selection (keyboard-only highlight). |
| `className` | `string` | - | Merged via `cn()` with the shadcn default. |

### Controlled-state contract on Command

Pass BOTH `value` and `onValueChange`, or NEITHER. The `value` here is the SELECTED item, not the search input.

```tsx
// CORRECT : controlled highlight
const [highlighted, setHighlighted] = useState<string>("")
<Command value={highlighted} onValueChange={setHighlighted}>...</Command>

// CORRECT : uncontrolled highlight (default)
<Command>...</Command>

// CORRECT : uncontrolled initial highlight
<Command defaultValue="apple">...</Command>
```

### filter signature : three arguments

The latest cmdk filter takes THREE arguments :

```tsx
<Command
  filter={(value, search, keywords) => {
    const haystack = (value + " " + (keywords ?? []).join(" ")).toLowerCase()
    return haystack.includes(search.toLowerCase()) ? 1 : 0
  }}
>
```

OLDER cmdk versions accepted `(value, search) => number` only. AI-generated code routinely emits the two-arg form. Both still compile (the third argument is silently ignored if omitted), but the keywords array does NOT flow into the score. See `shadcn-errors-cmdk-version-drift` for the full version history.

`value` AND `search` are both already TRIMMED and LOWERCASED by cmdk before they reach the filter callback. NEVER apply your own lowercase to `value` without also lowercasing the source-of-truth values you pass on each CommandItem ; mismatched casing produces zero matches.

## 2. CommandDialog

```tsx
function CommandDialog({
  title = "Command Palette",
  description = "Search for a command to run...",
  children,
  className,
  showCloseButton = true,
  ...props
}: React.ComponentProps<typeof Dialog> & {
  title?: string
  description?: string
  className?: string
  showCloseButton?: boolean
}) {
  return (
    <Dialog {...props}>
      <DialogHeader className="sr-only">
        <DialogTitle>{title}</DialogTitle>
        <DialogDescription>{description}</DialogDescription>
      </DialogHeader>
      <DialogContent
        className={cn("overflow-hidden p-0", className)}
        showCloseButton={showCloseButton}
      >
        <Command className="...long cmdk-targeting selector chain...">
          {children}
        </Command>
      </DialogContent>
    </Dialog>
  )
}
```

### Props

| Prop | Type | Default | Notes |
|------|------|---------|-------|
| `title` | `string` | `"Command Palette"` | sr-only DialogTitle text. Required for WAI-ARIA accessible name. |
| `description` | `string` | `"Search for a command to run..."` | sr-only DialogDescription. |
| `showCloseButton` | `boolean` | `true` | Forwards to DialogContent's `showCloseButton`. |
| `open` | `boolean` | - | Controlled visibility (Dialog prop). Pair with `onOpenChange`. |
| `onOpenChange` | `(open: boolean) => void` | - | Dialog prop. Fires on Esc, outside-click, Close, programmatic. |
| `defaultOpen` | `boolean` | `false` | Uncontrolled initial state. |
| `modal` | `boolean` | `true` | Dialog prop. |
| `container` | `HTMLElement` | `document.body` | Forwarded to DialogPortal. |
| `className` | `string` | - | Merged onto DialogContent. |

### Auto-injected accessibility

CommandDialog renders a `<DialogHeader className="sr-only">` containing DialogTitle + DialogDescription. NEVER add a SECOND visible DialogTitle as a child of CommandDialog ; Radix accepts it but the screen-reader announcement doubles. Customise the sr-only copy via the `title` / `description` props.

## 3. CommandInput

```tsx
function CommandInput({
  className,
  ...props
}: React.ComponentProps<typeof CommandPrimitive.Input>) {
  return (
    <div
      data-slot="command-input-wrapper"
      className="flex h-9 items-center gap-2 border-b px-3"
    >
      <SearchIcon className="size-4 shrink-0 opacity-50" />
      <CommandPrimitive.Input
        data-slot="command-input"
        className={cn(
          "flex h-10 w-full rounded-md bg-transparent py-3 text-sm outline-hidden placeholder:text-muted-foreground disabled:cursor-not-allowed disabled:opacity-50",
          className
        )}
        {...props}
      />
    </div>
  )
}
```

### Props (forwarded to cmdk `Command.Input`)

| Prop | Type | Default | Notes |
|------|------|---------|-------|
| `value` | `string` | - | Controlled search query. Pair with `onValueChange`. |
| `onValueChange` | `(value: string) => void` | - | Fires on every keystroke. |
| `defaultValue` | `string` | - | Uncontrolled initial query. |
| `placeholder` | `string` | - | Standard input placeholder. |
| `disabled` | `boolean` | `false` | Standard input disabled. |
| All native `<input>` props | - | - | Forwarded. |

### Value normalisation

cmdk applies `.trim()` to the input value before comparing to item values, and the COMPARISON is case-insensitive. The `value` prop on Command root and the `value` prop on CommandInput are NOT the same value : root.value = highlighted item, input.value = search query.

## 4. CommandList

```tsx
function CommandList({
  className,
  ...props
}: React.ComponentProps<typeof CommandPrimitive.List>) {
  return (
    <CommandPrimitive.List
      data-slot="command-list"
      className={cn(
        "max-h-[300px] scroll-py-1 overflow-x-hidden overflow-y-auto",
        className
      )}
      {...props}
    />
  )
}
```

### Props

| Prop | Type | Default | Notes |
|------|------|---------|-------|
| `className` | `string` | - | Merged with shadcn default `max-h-[300px]`. Override to allow taller lists. |
| All `<div>` props | - | - | Forwarded. |

### CSS variable

cmdk sets `--cmdk-list-height` to the natural height of the visible items. To animate the list height, target the variable :

```css
[cmdk-list] {
  min-height: 300px;
  height: var(--cmdk-list-height);
  transition: height 100ms ease;
}
```

## 5. CommandEmpty

```tsx
function CommandEmpty({
  ...props
}: React.ComponentProps<typeof CommandPrimitive.Empty>) {
  return (
    <CommandPrimitive.Empty
      data-slot="command-empty"
      className="py-6 text-center text-sm"
      {...props}
    />
  )
}
```

### Props

| Prop | Type | Default | Notes |
|------|------|---------|-------|
| `children` | `ReactNode` | - | Message shown when filter produces zero results. |
| `className` | `string` | - | Merged with shadcn default. |

REQUIRED for accessible empty-state UX. Renders automatically when the filter matches zero items. If you set `shouldFilter={false}`, you decide manually when to render CommandEmpty (e.g., `items.length === 0 && !loading`).

## 6. CommandGroup

```tsx
function CommandGroup({
  className,
  ...props
}: React.ComponentProps<typeof CommandPrimitive.Group>) {
  return (
    <CommandPrimitive.Group
      data-slot="command-group"
      className={cn(
        "overflow-hidden p-1 text-foreground [&_[cmdk-group-heading]]:px-2 [&_[cmdk-group-heading]]:py-1.5 [&_[cmdk-group-heading]]:text-xs [&_[cmdk-group-heading]]:font-medium [&_[cmdk-group-heading]]:text-muted-foreground",
        className
      )}
      {...props}
    />
  )
}
```

### Props

| Prop | Type | Default | Notes |
|------|------|---------|-------|
| `heading` | `ReactNode` | - | Group label rendered above the items. |
| `value` | `string` | - | Optional group identity (used by filter). |
| `forceMount` | `boolean` | `false` | Keep the group rendered even when every item is filtered out. |
| `className` | `string` | - | Merged with shadcn default. |

When all children are filtered away, cmdk applies the `hidden` HTML attribute to the group container (the group does NOT unmount).

## 7. CommandItem

```tsx
function CommandItem({
  className,
  ...props
}: React.ComponentProps<typeof CommandPrimitive.Item>) {
  return (
    <CommandPrimitive.Item
      data-slot="command-item"
      className={cn(
        "relative flex cursor-default items-center gap-2 rounded-sm px-2 py-1.5 text-sm outline-hidden select-none data-[disabled=true]:pointer-events-none data-[disabled=true]:opacity-50 data-[selected=true]:bg-accent data-[selected=true]:text-accent-foreground [&_svg]:pointer-events-none [&_svg]:shrink-0 [&_svg:not([class*='size-'])]:size-4 [&_svg:not([class*='text-'])]:text-muted-foreground",
        className
      )}
      {...props}
    />
  )
}
```

### Props

| Prop | Type | Default | Notes |
|------|------|---------|-------|
| `value` | `string` | inferred from `textContent` | Filter key + onSelect payload. UNIQUE across the Command. |
| `keywords` | `string[]` | `[]` | Alias terms that match this item during filtering. Trimmed. |
| `onSelect` | `(value: string) => void` | - | Fires on Enter or click. `value` is the trimmed `value` prop or inferred text. |
| `disabled` | `boolean` | `false` | Skipped by keyboard nav. Renders with `data-disabled="true"`. |
| `forceMount` | `boolean` | `false` | Keep rendered even when filter would hide. |
| `className` | `string` | - | Merged with shadcn default. |

### Data attributes

- `data-selected="true"` : currently highlighted (arrow keys / hover).
- `data-disabled="true"` : disabled.
- `data-value="..."` : the resolved value.

### Inferred value

If you omit `value`, cmdk reads `textContent` from the rendered DOM. This is fragile when items contain icons or complex children. ALWAYS provide an explicit `value` for items whose visible text might collide with another item or contain non-text nodes.

## 8. CommandSeparator

```tsx
function CommandSeparator({
  className,
  ...props
}: React.ComponentProps<typeof CommandPrimitive.Separator>) {
  return (
    <CommandPrimitive.Separator
      data-slot="command-separator"
      className={cn("-mx-1 h-px bg-border", className)}
      {...props}
    />
  )
}
```

### Props

| Prop | Type | Default | Notes |
|------|------|---------|-------|
| `alwaysRender` | `boolean` | `false` | When `true`, separator stays visible even with an active search query. |
| `className` | `string` | - | Merged with shadcn default. |

Default behaviour : hidden via `hidden` attribute while a search query is non-empty.

## 9. CommandShortcut

```tsx
function CommandShortcut({
  className,
  ...props
}: React.ComponentProps<"span">) {
  return (
    <span
      data-slot="command-shortcut"
      className={cn(
        "ml-auto text-xs tracking-widest text-muted-foreground",
        className
      )}
      {...props}
    />
  )
}
```

Not a cmdk part. Pure `<span>` placed inside a CommandItem to render a keyboard hint pill. Standard `<span>` props only.

## Extras imported directly from cmdk

shadcn's `command.tsx` does NOT re-export every cmdk part. Two practical ones to import directly :

### CommandLoading

```tsx
import { Command as CommandPrimitive } from "cmdk"
const CommandLoading = CommandPrimitive.Loading
```

Renders inside CommandList while async data is in flight. cmdk reports the loading state via `aria-busy` for screen readers. Accepts an optional `progress` prop (number 0-100).

### useCommandState

```tsx
import { useCommandState } from "cmdk"

const search = useCommandState((state) => state.search)
const value = useCommandState((state) => state.value)
```

Hook for advanced empty-state messages, sub-item gating, or analytics. Pass a selector that returns a slice ; the component re-renders when the slice changes. Use sparingly ; cmdk's internal store is not a public API.

## Keyboard interactions (verbatim from cmdk README + WAI-ARIA Combobox)

| Key | Behaviour |
|-----|-----------|
| `ArrowDown` | Move to next item. Wraps if `loop` is true. |
| `ArrowUp` | Move to previous item. Wraps if `loop` is true. |
| `Enter` | Fire `onSelect` on highlighted item. |
| `Home` | Jump to first item. |
| `End` | Jump to last item. |
| `Ctrl+N` / `Ctrl+J` | Next item (when `vimBindings={true}`, default). |
| `Ctrl+P` / `Ctrl+K` | Previous item (when `vimBindings={true}`, default). |
| Typing | Narrows the list via the filter. |
| `Esc` | Closes CommandDialog (Dialog handler). |

## Imports

All nine shadcn primitives are exported from `@/components/ui/command` :

```tsx
import {
  Command,
  CommandDialog,
  CommandInput,
  CommandList,
  CommandEmpty,
  CommandGroup,
  CommandItem,
  CommandShortcut,
  CommandSeparator,
} from "@/components/ui/command"
```

Two extras imported directly from cmdk when needed :

```tsx
import { Command as CommandPrimitive, useCommandState } from "cmdk"
const CommandLoading = CommandPrimitive.Loading
```

ALWAYS import the nine wrappers from the local `@/components/ui/command` alias ; they apply the shadcn `data-slot` attributes and default Tailwind classes. NEVER import them from `cmdk` directly ; you lose the styling and the cmdk part names use a different casing (`Command.Item` vs `CommandItem`).
