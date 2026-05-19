---
name: shadcn-syntax-selectors
description: >
  Use when picking the right "selector" primitive in a shadcn ui app
  and you are unsure whether to reach for a Native HTML `<select>`,
  a shadcn Select (Radix Select wrapped, 10 primitives), a Combobox
  recipe (Popover + Command composed by hand or the new 2026
  Combobox primitive), or a Command palette (CommandDialog as a
  Cmd+K action launcher) ; covers the four-way decision criteria
  (list size, searchability, async loading, palette UX,
  mobile-native-picker need), the form-binding contract per
  selector (Native works with `register`, Radix Select and
  Combobox require `Controller`), the Combobox-as-recipe
  composition (Popover + Command + CommandInput + CommandList +
  CommandEmpty + CommandGroup + CommandItem), the Command-palette
  hotkey wiring (Cmd+K via `useEffect` + `keydown` listener),
  the async options pattern (server-fetched, loading state,
  empty state, error state), and the a11y wiring per selector
  (`aria-expanded` / `aria-controls` / `aria-activedescendant`).
  Prevents the canonical selector failures : binding Radix
  Select with React Hook Form's `register` (the controlled
  `onValueChange` is swallowed and the form silently captures
  the placeholder string forever), forgetting `modal={true}` on
  the Popover wrapping a Combobox (focus escapes when an item
  is clicked and the list traps the user inside a phantom open
  state), reaching for Native `<select>` when the list has 30+
  items (mobile native picker becomes a scroll trap and desktop
  styling is unstylable past `appearance`), opening a Command
  palette without `<CommandDialog>` so it is not portalled and
  no DialogTitle exists (Radix logs a critical a11y warning,
  axe-core flags it), rendering an async-loaded Combobox without
  a loading + empty + error tri-state (users see a frozen
  "No results" while the request is still in-flight), and
  omitting `<SelectValue>` inside `<SelectTrigger>` (the
  selected label never paints, the trigger shows the
  placeholder permanently).
  Covers the four selector surfaces verbatim from
  `apps/v4/registry/new-york-v4/ui/select.tsx`, the
  shadcn Select composition (10 primitives : Select /
  SelectTrigger / SelectValue / SelectContent / SelectGroup /
  SelectLabel / SelectItem / SelectSeparator /
  SelectScrollUpButton / SelectScrollDownButton), the Radix
  Select root contract (`value` + `onValueChange` controlled,
  `defaultValue` uncontrolled, `name`, `dir`, `disabled`,
  `required`), the SelectContent `position="item-aligned" |
  "popper"` and `align` defaults, the Combobox legacy recipe
  (Popover + Command, still the dominant pattern in the wild),
  the new 2026 Combobox primitive surface (ComboboxInput,
  ComboboxContent, ComboboxList, ComboboxItem, ComboboxEmpty,
  ComboboxChips, ComboboxChipsInput, ComboboxGroup,
  ComboboxLabel, ComboboxCollection, `multiple`,
  `itemToStringValue`, `showClear`, `autoHighlight`, `render`),
  the Command-as-palette wiring via `<CommandDialog>` + global
  hotkey listener, native `<select>` form binding via
  `register`, the controlled / uncontrolled split, and the
  `aria-invalid` + `data-invalid` error-state hookup that
  shadcn-syntax-field consumes.
  Keywords: shadcn select, shadcn selector, combobox, command
  palette, native select, when to use Combobox, searchable
  select, dropdown, how do I make a dropdown, cmd+k, ctrl+k,
  autocomplete, Select vs Combobox, Combobox vs Command, when
  to use native select, mobile select picker, SelectTrigger,
  SelectValue, SelectContent, SelectItem, SelectGroup,
  SelectLabel, SelectSeparator, position popper, position
  item-aligned, defaultValue Select, onValueChange Select,
  Controller Select, register Select, Combobox recipe,
  Popover Command, CommandInput, CommandList, CommandEmpty,
  CommandGroup, CommandItem, CommandDialog, ComboboxInput,
  ComboboxContent, ComboboxItem, multiple Combobox, async
  Combobox, loading state Combobox, no results Combobox,
  searchable dropdown, typeahead, autocomplete input, command
  palette hotkey, useEffect keydown, app-wide command palette,
  action launcher, my dropdown shows the placeholder, select
  shows placeholder after pick, register Select swallows
  onValueChange, Combobox items not clickable, Combobox
  Popover focus escape, Combobox missing modal, dropdown
  cannot be searched, dropdown too long, select for 50
  options, list of 100 items, dropdown for countries, country
  picker, framework picker.
license: MIT
compatibility: "Designed for Claude Code. Requires shadcn ui evergreen-2026."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# shadcn ui : Selectors Decision Tree

The shadcn ui catalogue exposes four primitives that all answer the same user need ("let me pick one or more values from a list"). They are NOT interchangeable. Choosing the wrong one creates predictable UX failures (frozen dropdowns, swallowed form state, lost focus, scroll traps on mobile, missing a11y names). This skill is the decision tree.

The four selectors :

1. **Native `<select>`** : the browser's built-in element ; shadcn ships a tiny styled wrapper but the semantics are unchanged.
2. **shadcn Select** : Radix Select wrapped with shadcn theming ; 10-primitive composition surface ; styled like a custom dropdown but the underlying contract is Radix's controlled-state pattern.
3. **Combobox** : a RECIPE, NOT a single primitive. Built by composing Popover + Command (the cmdk wrapper) by hand, or the new 2026 dedicated Combobox primitive surface (ComboboxInput / ComboboxContent / ComboboxList / ComboboxItem / ComboboxEmpty).
4. **Command palette** : the same Command primitive as Combobox, but wrapped in `<CommandDialog>` and triggered by a global hotkey (Cmd+K / Ctrl+K) ; used for app-wide action launching, NOT for picking a value in a form field.

## Quick Reference

### Decision table

| Criterion | Native `<select>` | shadcn Select | Combobox | Command palette |
|-----------|-------------------|---------------|----------|-----------------|
| List size | up to ~10 items | ~5 to 30 items | 20 to thousands | irrelevant (search-driven) |
| User search / typeahead | NO (browser-specific letter-jump) | NO (single-letter jump, fragile) | YES (CommandInput filter) | YES (CommandInput filter, fuzzy) |
| Async / server-loaded options | possible but UX poor | possible but UX poor | YES (canonical use case) | YES (action search) |
| Mobile native picker (iOS wheel, Android sheet) | YES (free) | NO (custom overlay) | NO | NO |
| Custom styling beyond `appearance` | NO (severely limited) | YES (full Tailwind) | YES (full Tailwind) | YES (full Tailwind) |
| Multi-select | YES (`multiple` attr ; UX poor) | NO (single-value only) | YES (manual via ComboboxChips or array state) | YES (palette can fire multi-action chains) |
| Triggered by hotkey (Cmd+K) | NO | NO | uncommon | YES (canonical use case) |
| Form binding | `register("field")` | `Controller` (REQUIRED) | `Controller` (REQUIRED) | NOT a form input |
| `aria-expanded` / `aria-controls` on trigger | automatic | automatic (Radix) | MANUAL (you add) | automatic (CommandDialog) |
| Component count | 1 | 10 primitives | 5+ (Popover + Command + filter logic) | 1 wrapper (CommandDialog) + global hotkey |

### Decision tree

```
Q1. Is the user picking ONE value from a known, short, static list (<= 10 items)
    AND a default mobile picker would be acceptable?
    yes -> Native <select>. Bind via register("field").
    no  -> Q2

Q2. Is the user picking ONE value from a known, medium list (~5-30 items)
    AND the list does NOT need search / autocomplete?
    yes -> shadcn Select. Bind via Controller render-prop.
    no  -> Q3

Q3. Is the user picking ONE or MORE values from a list that
    is searchable, possibly async / server-loaded, or longer than 30 items?
    yes -> Combobox (Popover + Command recipe, OR new 2026 Combobox primitive).
           Bind via Controller render-prop.
    no  -> Q4

Q4. Is the user INVOKING an action (not picking a value), via a
    global hotkey (Cmd+K / Ctrl+K), with the dropdown spanning the
    whole app (not bound to a single form field)?
    yes -> Command palette via <CommandDialog>. NOT a form input.
    no  -> Re-read Q1-Q3 ; you are probably overthinking and want shadcn Select.
```

### Five invariants

1. ALWAYS use `Controller` (from `react-hook-form`) when binding a shadcn Select or a Combobox to a form. NEVER use `register("field")` ; the controlled `onValueChange` callback is swallowed and your form captures `undefined` forever.
2. ALWAYS render `<SelectValue placeholder="...">` inside `<SelectTrigger>`. NEVER omit it ; without SelectValue the trigger never paints the selected label and the placeholder is permanent.
3. ALWAYS pass `modal={true}` to the `<Popover>` that wraps a Combobox. NEVER leave Popover non-modal in a Combobox recipe ; focus escapes when the user clicks an item, the list never closes, and keyboard navigation breaks.
4. ALWAYS wrap a Command-palette UX in `<CommandDialog>`, NEVER in `<Command>` alone. CommandDialog provides the Portal, the Overlay, the focus trap, and the required DialogTitle. Plain `<Command>` does NOT.
5. ALWAYS show three states for an async Combobox : loading (skeleton or "Loading..."), empty (`<CommandEmpty>`), and error (an error row). NEVER render only `<CommandEmpty>` while the request is still in-flight ; users will read "No results" for a successful query in progress and abandon.

## Selector 1 : Native HTML `<select>`

### When

- List is short (<= 10 items), known at build time, no search.
- Mobile is a first-class target and you want the OS-native picker (iOS wheel, Android bottom sheet).
- Form is plain HTML / React Hook Form `register` and you do NOT want a Radix client boundary.
- Styling needs are minimal (border, padding, font ; nothing exotic).

### Composition

Single element. No primitives. shadcn evergreen-2026 ships a thin wrapper (`<NativeSelect>`) that applies Tailwind tokens but uses the real `<select>` underneath.

```tsx
<select
  {...register("country")}
  className="h-9 rounded-md border border-input bg-transparent px-3 text-sm"
>
  <option value="">Pick a country</option>
  <option value="nl">Netherlands</option>
  <option value="de">Germany</option>
</select>
```

### Form binding

`register("field")` works directly. The `<select>` element is an uncontrolled input by React's standards and React Hook Form's `register` wires `name`, `ref`, `onChange`, `onBlur` automatically.

### a11y

The browser handles `aria-expanded`, `aria-haspopup`, keyboard navigation, screen-reader semantics. Label via `<label htmlFor>` or wrap.

### Limits

- Styling past `appearance: none` + a chevron-via-background-image is fragile across browsers.
- No filter / typeahead beyond the browser's letter-jump.
- For >30 items, mobile native picker becomes a scroll trap.

## Selector 2 : shadcn Select (Radix Select wrapped)

### When

- List is medium (~5 to 30 items), known at runtime.
- You need custom styling matching the rest of the shadcn theme.
- Single-value selection, no search.
- Form-bound via Controller.

### Composition (10 primitives, verbatim from `select.tsx`)

| Primitive | Purpose |
|-----------|---------|
| `Select` | Root state container ; owns `value` / `onValueChange` / `defaultValue` / `open` / `onOpenChange` / `defaultOpen` / `name` / `dir` / `disabled` / `required` |
| `SelectTrigger` | Button that opens the dropdown ; `size?: "sm" \| "default"` (shadcn-only addition) ; renders the ChevronDown icon |
| `SelectValue` | Renders the selected item's text inside the trigger ; takes `placeholder` |
| `SelectContent` | Dropdown surface ; auto-portals via `SelectPrimitive.Portal` ; `position="item-aligned" \| "popper"` (default `item-aligned`) ; `align="center"` default ; `side` / `sideOffset` for popper mode |
| `SelectGroup` | Groups items into sections |
| `SelectLabel` | Label for a SelectGroup (NOT a form label) |
| `SelectItem` | Individual option ; `value` (REQUIRED, NEVER empty string) ; `disabled` |
| `SelectSeparator` | Horizontal rule between groups |
| `SelectScrollUpButton` | Renders inside SelectContent when content overflows above |
| `SelectScrollDownButton` | Renders inside SelectContent when content overflows below |

### Minimal example (controlled)

```tsx
"use client"

import {
  Select, SelectTrigger, SelectValue, SelectContent,
  SelectGroup, SelectLabel, SelectItem,
} from "@/components/ui/select"

const [value, setValue] = React.useState<string>("")

<Select value={value} onValueChange={setValue}>
  <SelectTrigger>
    <SelectValue placeholder="Pick a framework" />
  </SelectTrigger>
  <SelectContent>
    <SelectGroup>
      <SelectLabel>Meta-frameworks</SelectLabel>
      <SelectItem value="next">Next.js</SelectItem>
      <SelectItem value="remix">Remix</SelectItem>
      <SelectItem value="astro">Astro</SelectItem>
    </SelectGroup>
  </SelectContent>
</Select>
```

### Form binding (via Controller)

```tsx
<Controller
  control={form.control}
  name="framework"
  render={({ field }) => (
    <Select value={field.value} onValueChange={field.onChange}>
      <SelectTrigger><SelectValue placeholder="Pick" /></SelectTrigger>
      <SelectContent>
        <SelectItem value="next">Next.js</SelectItem>
      </SelectContent>
    </Select>
  )}
/>
```

ALWAYS go through `Controller`. NEVER `register("framework")` ; Radix Select's `onValueChange` is the only state writer, and `register` wires `onChange` (which never fires).

### `position` choice

- `"item-aligned"` (default) : the selected item appears under the trigger, list shifts to align ; mimics native macOS select. Use when the list is short and centred alignment is desired.
- `"popper"` : the list opens below the trigger, fixed origin ; behaves like a Popover. Use when content can grow and you want consistent placement under the trigger.

### a11y

Radix handles `aria-expanded`, `aria-controls`, `aria-activedescendant`, focus-trapping inside the open list, type-ahead letter-jump. The trigger reads its name from `SelectValue` (when populated) or from a paired `<Label htmlFor>`.

Error states : pair with shadcn `Field` (see shadcn-syntax-field) ; `data-invalid` on Field cascades to `aria-invalid` on SelectTrigger via the trigger's `aria-invalid:` Tailwind variant.

## Selector 3 : Combobox

Combobox is a RECIPE, not a single primitive. Two valid surfaces exist in evergreen-2026 :

### Surface A : legacy recipe (Popover + Command)

Still the dominant pattern in the wild (used by ~80% of shadcn installs as of v0/Cursor/Vercel template surveys). Composed by hand from :

| Primitive | Source | Role |
|-----------|--------|------|
| `Popover` | `@radix-ui/react-popover` | Floating container, owns open state |
| `PopoverTrigger` | same | Button that toggles the list |
| `PopoverContent` | same | Floating surface ; `modal={true}` REQUIRED via root Popover prop |
| `Command` | `cmdk` (via shadcn) | Filterable list root |
| `CommandInput` | same | Search input ; auto-wires `cmdk`'s filter |
| `CommandList` | same | Scrollable list container |
| `CommandEmpty` | same | Shown when filter matches nothing |
| `CommandGroup` | same | Section of items |
| `CommandItem` | same | One option ; `value` (auto-filtered against input) ; `onSelect(value)` |

The recipe (full code in `references/examples.md`) :

```tsx
"use client"

const [open, setOpen] = React.useState(false)
const [value, setValue] = React.useState("")

<Popover open={open} onOpenChange={setOpen} modal>
  <PopoverTrigger asChild>
    <Button variant="outline" role="combobox" aria-expanded={open}>
      {value ? frameworks.find(f => f.value === value)?.label : "Pick a framework"}
    </Button>
  </PopoverTrigger>
  <PopoverContent className="w-[200px] p-0">
    <Command>
      <CommandInput placeholder="Search framework..." />
      <CommandList>
        <CommandEmpty>No framework found.</CommandEmpty>
        <CommandGroup>
          {frameworks.map(f => (
            <CommandItem
              key={f.value}
              value={f.value}
              onSelect={v => { setValue(v); setOpen(false) }}
            >
              {f.label}
            </CommandItem>
          ))}
        </CommandGroup>
      </CommandList>
    </Command>
  </PopoverContent>
</Popover>
```

### Surface B : new 2026 Combobox primitive

As of late-2025 the shadcn docs document a dedicated Combobox surface with first-class subcomponents : `Combobox`, `ComboboxInput`, `ComboboxContent`, `ComboboxList`, `ComboboxItem`, `ComboboxEmpty`, `ComboboxChips`, `ComboboxChipsInput`, `ComboboxGroup`, `ComboboxLabel`, `ComboboxCollection`. Props on the root : `items`, `value`, `onValueChange`, `multiple`, `itemToStringValue`, `showClear`, `autoHighlight`, `render`. Full example in `references/examples.md` Recipe 4.

Prefer Surface B for new projects. Existing projects on Surface A do NOT need to migrate. Both are supported.

### a11y for Combobox

Combobox does NOT automatically wire `aria-expanded` and `aria-controls` to the trigger. ALWAYS set them by hand on the PopoverTrigger button :

```tsx
<Button role="combobox" aria-expanded={open} aria-controls="combobox-list">
```

`cmdk` handles `aria-activedescendant` on the list internally.

### Form binding (via Controller)

```tsx
<Controller
  control={form.control}
  name="framework"
  render={({ field }) => (
    <Popover open={open} onOpenChange={setOpen} modal>
      <PopoverTrigger asChild>...</PopoverTrigger>
      <PopoverContent>
        <Command>
          <CommandInput />
          <CommandList>
            <CommandGroup>
              {opts.map(o => (
                <CommandItem
                  key={o.value}
                  value={o.value}
                  onSelect={v => { field.onChange(v); setOpen(false) }}
                >{o.label}</CommandItem>
              ))}
            </CommandGroup>
          </CommandList>
        </Command>
      </PopoverContent>
    </Popover>
  )}
/>
```

The `Controller`'s `field.onChange` replaces the local `setValue` ; the local `open` state stays local (it is not form data).

## Selector 4 : Command palette

### When

- App-wide action launcher (open files, run commands, navigate routes).
- Triggered by a global hotkey (`Cmd+K` on macOS, `Ctrl+K` on Windows/Linux).
- Spans the full app, NOT bound to a single form field.
- Items are actions (callbacks), not values to capture.

### Composition

ONE wrapper component : `<CommandDialog>` (from `@/components/ui/command`). It internally combines `<Dialog>` + `<Command>` so you get the Portal, the Overlay, the focus trap, and the mandatory DialogTitle for free.

```tsx
"use client"

import * as React from "react"
import {
  CommandDialog, CommandInput, CommandList, CommandEmpty,
  CommandGroup, CommandItem,
} from "@/components/ui/command"

export function CommandPalette() {
  const [open, setOpen] = React.useState(false)

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

  return (
    <CommandDialog open={open} onOpenChange={setOpen}>
      <CommandInput placeholder="Type a command or search..." />
      <CommandList>
        <CommandEmpty>No results found.</CommandEmpty>
        <CommandGroup heading="Suggestions">
          <CommandItem onSelect={() => router.push("/inbox")}>Inbox</CommandItem>
          <CommandItem onSelect={() => router.push("/settings")}>Settings</CommandItem>
        </CommandGroup>
      </CommandList>
    </CommandDialog>
  )
}
```

### CommandDialog vs raw Command

- `<CommandDialog>` : portalled, focus-trapped, DialogTitle wired (sr-only by default in shadcn evergreen-2026) ; this is the palette UX.
- `<Command>` : inline, no portal, no focus trap ; this is the Combobox recipe's filtering primitive, NOT a palette.

NEVER use plain `<Command>` for a Cmd+K palette. It will render inline wherever it sits in the tree, miss the focus trap, and skip the DialogTitle requirement (Radix logs a critical a11y warning).

### Deeper Command surface

For Command-specific topics (`shouldFilter`, custom filter function, fuzzy ranking, async value loading inside Command, Loading state, `forceMount` on items), see the sibling skill **shadcn-syntax-command**. This skill ONLY covers the decision of when to reach for Command-as-palette vs Combobox vs Select.

## Form integration matrix

| Selector | Form binding | Why |
|----------|--------------|-----|
| Native `<select>` | `register("field")` | Uncontrolled DOM element ; React Hook Form's `register` wires `name` / `ref` / `onChange` / `onBlur` |
| shadcn Select | `Controller` | Radix Select is controlled by `onValueChange` ; `register` wires `onChange` which never fires |
| Combobox | `Controller` | Combobox state is owned by Popover + Command ; `field.onChange` replaces local `setValue` |
| Command palette | NOT a form input | Items are actions, not values |

## Async-loaded options (Combobox)

When options come from a server (search-as-you-type, paginated list, autocomplete) :

```tsx
const [query, setQuery] = React.useState("")
const { data, isLoading, error } = useQuery({
  queryKey: ["users", query],
  queryFn: () => searchUsers(query),
  enabled: query.length >= 2,
})

<CommandList>
  {isLoading && <CommandLoading>Searching...</CommandLoading>}
  {error && <div role="alert">Failed to load</div>}
  {!isLoading && !error && data?.length === 0 && <CommandEmpty>No users found.</CommandEmpty>}
  {data?.map(u => <CommandItem key={u.id} value={u.id} onSelect={...}>{u.name}</CommandItem>)}
</CommandList>
```

ALWAYS render the three states (loading, empty, error) as MUTUALLY EXCLUSIVE branches. NEVER let `<CommandEmpty>` paint while `isLoading` is true ; users read "No results" during a successful in-flight request and abandon.

## Companion Skills

- [shadcn-syntax-command](../shadcn-syntax-command/SKILL.md) : the Command primitive in depth (cmdk root, filter prop, custom filter function, shouldFilter, CommandLoading, async patterns). Read when the recipe-level decision is settled and you need Command internals.
- [shadcn-syntax-form](../shadcn-syntax-form/SKILL.md) : Form / FormField / FormItem / FormLabel / FormControl / FormDescription / FormMessage and the Controller wiring contract. Read when binding a Selector to React Hook Form.
- [shadcn-impl-form-validation](../../shadcn-impl/shadcn-impl-form-validation/SKILL.md) : zod schema + resolver + error-state cascade (`aria-invalid` / `data-invalid` on Field). Read when error states must visually flag the Selector.
- [shadcn-syntax-dialog](../shadcn-syntax-dialog/SKILL.md) : the underlying focus-trap + Portal contract that CommandDialog inherits. Read when Cmd+K palette behaves unexpectedly.
- [shadcn-syntax-field](../shadcn-syntax-field/SKILL.md) : the v4 Field wrapper that cascades `data-invalid` to descendants ; SelectTrigger picks up `aria-invalid:border-destructive` from this.

## Reference Links

- [references/methods.md](references/methods.md) : primitive signatures (10 Select primitives with prop tables, Combobox recipe primitives list, Combobox 2026 primitive surface, Command palette primitives, Form-Controller path per selector).
- [references/examples.md](references/examples.md) : six canonical recipes (Native in Form via register, Select with Controller, Combobox legacy recipe, Combobox 2026 primitive, Command palette with Cmd+K hotkey, async Combobox with loading/empty/error tri-state).
- [references/anti-patterns.md](references/anti-patterns.md) : six anti-patterns with WHY and FIX.

## Sources

All claims in this skill trace to URLs in `SOURCES.md` :

- https://ui.shadcn.com/docs/components/radix/select (canonical Select docs, prop table, controlled/uncontrolled split, position, form integration via Controller)
- https://ui.shadcn.com/docs/components/radix/combobox (Combobox page, 2026 primitive surface, multi-select with ComboboxChips, recipe context)
- https://github.com/shadcn-ui/ui (apps/v4/registry/new-york-v4/ui/select.tsx verbatim ; 10 primitives, position default `item-aligned`, size variants)
- https://www.radix-ui.com/primitives/docs/components/select (Radix Select primitive API, value/onValueChange contract, Portal behaviour, position/side/align)
- https://cmdk.paco.me (cmdk root, CommandInput filter, CommandList, CommandEmpty, CommandLoading)

Verified 2026-05-19.
