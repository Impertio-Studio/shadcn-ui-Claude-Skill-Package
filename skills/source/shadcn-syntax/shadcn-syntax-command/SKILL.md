---
name: shadcn-syntax-command
description: >
  Use when adding, debugging, or refactoring a shadcn ui Command primitive
  (the cmdk-backed command-palette / fuzzy-filter list), composing its nine
  parts (Command / CommandDialog / CommandInput / CommandList /
  CommandEmpty / CommandGroup / CommandItem / CommandSeparator /
  CommandShortcut), deciding between an inline Command surface and the
  modal CommandDialog wrapping, wiring the Cmd+K / Ctrl+K global hotkey
  via useEffect + keydown listener, switching from cmdk's built-in
  fuzzy filter to server-side / async filtering via `shouldFilter={false}`
  with externally-managed `items` state plus CommandLoading, providing a
  stable `value` prop on every CommandItem so cmdk can identify it for
  filtering and keyboard navigation, handling `onSelect: (value: string)
  => void` to route the user's choice, building a Combobox by rendering
  Command inside a Popover (the canonical shadcn recipe), or building
  nested / multi-page command palettes with sub-routes via a React-state
  `pages` stack.
  Prevents the canonical Command failures : writing a `filter` prop with
  the wrong arity (cmdk accepts `(value, search, keywords) => number`,
  AI training data still suggests the older two-arg form which silently
  ignores the keywords argument), missing the implicit lowercase / trim
  normalisation that cmdk applies to every value so a case-sensitive
  match never triggers, registering the Cmd+K keydown listener without
  the matching `removeEventListener` cleanup (memory leak + stacked
  toggles on every mount), passing `shouldFilter={false}` without doing
  the manual filtering yourself (every render shows the full unfiltered
  list), giving two `CommandItem` elements the same `value` (cmdk
  collapses them and keyboard nav stops mid-list), omitting CommandEmpty
  so a zero-result search renders a silent blank list, and combining
  Command with a vaul Drawer or a parent Dialog without forwarding focus
  through the cmdk input (the search field never receives focus).
  Covers the nine command primitives verbatim from `apps/v4/registry/
  new-york-v4/ui/command.tsx`, the cmdk underlying API at
  https://cmdk.paco.me (Command.Root + nine parts + useCommandState hook),
  the controlled-state contract on Command (`value` + `onValueChange`)
  and on CommandInput (`value` + `onValueChange`) and on CommandDialog
  (`open` + `onOpenChange`), the three-arg `filter: (value, search,
  keywords) => number` signature, the `shouldFilter` opt-out for
  server-side search, the `loop` prop that wraps arrow-key navigation,
  the `forceMount` prop on Item / Group to keep entries visible
  regardless of filter, the keywords array on CommandItem for filter
  aliases, the CommandLoading slot for async fetches, the sr-only
  DialogTitle / DialogDescription that CommandDialog injects so screen-
  reader users hear a name, and the `useCommandState` hook for advanced
  empty-state copies.
  Keywords: shadcn command, cmdk, command palette, command menu,
  CommandDialog, CommandInput, CommandList, CommandEmpty, CommandGroup,
  CommandItem, CommandSeparator, CommandShortcut, CommandLoading,
  shouldFilter, filter prop, onValueChange, onSelect, async command,
  server side filter, cmdk filter signature, cmdk version drift, cmd+k
  hotkey, ctrl+k hotkey, global hotkey, useEffect keydown, removeEventListener
  cleanup, fuzzy search, value normalization, lowercase value, trim value,
  keywords prop, forceMount item, loop prop, useCommandState, sub-pages,
  nested command, command inside popover, command inside drawer, vaul cmdk,
  combobox recipe, how do I build a command palette, what is cmdk,
  command not filtering, command not showing results, command items
  unclickable, command keyboard nav broken, command input has no focus,
  command shows blank list, command not closing on select, command memory
  leak, command duplicate values.
license: MIT
compatibility: "Designed for Claude Code. Requires shadcn ui evergreen-2026."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# shadcn ui : Command Syntax

The Command component is shadcn's command-palette and fuzzy-filter primitive. It is a thin styling wrapper around the `cmdk` library by Paco Coursey, exposing a nine-part composition surface. cmdk does the heavy work : it filters items by the input value, sorts by match score, manages keyboard navigation, and wires the ARIA roles for an accessible combobox. shadcn adds the visual layer plus a `CommandDialog` that drops the same primitive inside a shadcn Dialog with an sr-only title.

Command is the foundation of the Combobox recipe (Popover + Command), of the global "Press Cmd+K" launcher, and of every fuzzy filter surface in the catalogue (Sidebar searches, Combobox inputs, dashboard quick-actions). Mastering Command is mastering the cmdk surface that ships verbatim through shadcn.

## Quick Reference

### The 9 primitives

| Primitive | Purpose |
|-----------|---------|
| `Command` | Root state container ; owns `value` / `onValueChange` / `shouldFilter` / `filter` / `loop` ; wraps cmdk `Command.Root` |
| `CommandDialog` | Wraps `Command` in shadcn `Dialog` ; forwards `open` / `onOpenChange` ; injects sr-only DialogTitle + DialogDescription |
| `CommandInput` | Search input ; controlled via `value` + `onValueChange` ; wrapped with a search icon by shadcn |
| `CommandList` | Scrollable container for groups, items, empty, loading ; max-h-[300px] by default |
| `CommandEmpty` | Auto-rendered when filtered results are zero ; REQUIRED for non-silent no-match UX |
| `CommandGroup` | Groups items with a `heading` ; hidden via `hidden` attribute when all children are filtered out |
| `CommandItem` | Selectable row ; `value` (filter key), `keywords` (aliases), `onSelect`, `disabled`, `forceMount` |
| `CommandSeparator` | Visual divider ; hidden by default while a search query is active (use `alwaysRender` to override) |
| `CommandShortcut` | Pure `<span>` styled as a keyboard-hint pill ; placed at the end of a CommandItem |

### Minimal inline Command

```tsx
"use client"

import {
  Command, CommandInput, CommandList, CommandEmpty,
  CommandGroup, CommandItem, CommandSeparator, CommandShortcut,
} from "@/components/ui/command"

<Command className="rounded-lg border shadow-md max-w-sm">
  <CommandInput placeholder="Type a command or search..." />
  <CommandList>
    <CommandEmpty>No results found.</CommandEmpty>
    <CommandGroup heading="Suggestions">
      <CommandItem value="calendar" onSelect={(v) => console.log(v)}>
        Calendar
      </CommandItem>
      <CommandItem value="search-emoji">
        Search Emoji
      </CommandItem>
    </CommandGroup>
    <CommandSeparator />
    <CommandGroup heading="Settings">
      <CommandItem value="profile">
        Profile
        <CommandShortcut>Ctrl+P</CommandShortcut>
      </CommandItem>
    </CommandGroup>
  </CommandList>
</Command>
```

### Six invariants

1. ALWAYS render `CommandEmpty` inside `CommandList`. NEVER omit it ; a zero-result search renders a silent empty container that looks like a broken layout.
2. ALWAYS give every `CommandItem` a stable, UNIQUE `value` prop (or rely on `textContent` only when it is provably unique). NEVER let two items share the same value ; cmdk uses value as the identity key for filtering and arrow-key navigation, and duplicates break both.
3. ALWAYS use the three-argument `filter: (value, search, keywords) => number` signature on the latest cmdk. NEVER write a two-argument filter on the current library and assume keywords flow in ; the two-arg form still compiles but the third argument disappears, and AI training data routinely emits the stale signature.
4. ALWAYS register Cmd+K with a matching `removeEventListener` in the `useEffect` cleanup. NEVER drop the cleanup ; the listener keeps stacking on every component mount, the global toggles multiply, and React DevTools surfaces a memory leak.
5. ALWAYS pair `shouldFilter={false}` with externally-managed `items` state that you filter yourself in JavaScript before render. NEVER set `shouldFilter={false}` and expect cmdk to still narrow the list ; with the flag off, cmdk renders every CommandItem verbatim and your search input becomes decorative.
6. ALWAYS keep the `"use client"` directive at the top of `components/ui/command.tsx`. NEVER strip it ; cmdk uses `useId`, `useSyncExternalStore`, manual DOM ordering, and is a hard client-only library.

## Decision Tree 1 : CommandDialog or Command inline?

```
Q1. Is the command surface a GLOBAL launcher (Cmd+K, app-wide
    quick actions, route navigation) ?
    yes -> CommandDialog. Pair with a useEffect that listens for
           Cmd+K / Ctrl+K and toggles the controlled `open` state.
    no  -> Q2

Q2. Is the command surface anchored to a SPECIFIC trigger
    button (e.g., a combobox in a form field, a filter button
    in a table header) ?
    yes -> Use Popover + Command. Popover handles the trigger
           and positioning ; Command handles the fuzzy filter.
           See `references/examples.md` Recipe 4.
    no  -> inline Command. Render <Command> directly inside the
           parent surface (e.g., a Card, a Sidebar panel).
```

CommandDialog wraps Command inside shadcn's Dialog. It injects an sr-only `DialogTitle` and `DialogDescription` so the WAI-ARIA Dialog pattern is satisfied without a visible heading. NEVER add a second visible DialogTitle inside CommandDialog ; cmdk's input is the de-facto label.

## Decision Tree 2 : shouldFilter on or off?

```
Q1. Is the FULL set of items already in memory client-side
    (typed array of objects, JSON imported at build time) ?
    yes -> default. shouldFilter={true}. cmdk does the fuzzy
           ranking. You provide `value` + `keywords` for hints.
    no  -> Q2

Q2. Are you fetching results from a server / API as the user
    types (debounced query against /search?q=...) ?
    yes -> shouldFilter={false}. You own the filtering :
           - Track `query` state on CommandInput via `value` +
             `onValueChange`.
           - Debounce + fetch ; set `items` state with the
             server response.
           - Render `items.map(...)` inside CommandList.
           - Render <CommandLoading> while the fetch is in
             flight (cmdk shows it inside the list).
    no  -> Q3

Q3. Do you need a CUSTOM ranking (e.g., user-history-weighted,
    prefix-only, regex) ?
    yes -> shouldFilter={true} with a `filter` prop :
           filter={(value, search, keywords) => number}.
           Return 0 to hide, > 0 to show ; higher number = higher rank.
           Trim/lowercase your own values before comparing ;
           cmdk has already lowercased + trimmed `value`.
    no  -> default.
```

cmdk's default filter is a case-insensitive fuzzy match against the trimmed `value` of each item plus its `keywords` array. The `filter` prop only overrides the SCORE computation ; it does not disable filtering. To disable filtering entirely use `shouldFilter={false}`.

## Decision Tree 3 : Cmd+K hotkey or trigger button?

```
Q1. Does the palette need to open from ANYWHERE in the app
    without a visible button (the classic Vercel / Linear UX) ?
    yes -> Global Cmd+K listener inside a top-level layout
           component (e.g., RootLayout). Use a controlled
           `<CommandDialog open onOpenChange>`. See Recipe 1
           in `references/examples.md`.
    no  -> Q2

Q2. Does the palette open from a specific button (header
    search icon, sidebar item) ?
    yes -> Render a Button that calls `setOpen(true)` and a
           controlled `<CommandDialog open onOpenChange>`.
           No keydown listener needed.
    no  -> Inline <Command> with no Dialog wrapping. Visible
           by default ; no open/close state at all.
```

The Cmd+K listener pattern is documented verbatim in the cmdk README and shadcn's component page. The key detail : the listener MUST clean up on unmount, and `e.preventDefault()` MUST run before the state toggle so the browser does not also handle Cmd+K (Chrome opens the location-bar suggestions panel by default).

## The 9 primitives in detail

### Command (Root)

Owns state, filtering, and keyboard nav. Wraps cmdk `Command.Root`. Key props :

- `value?: string` : controlled SELECTED item value. Pair with `onValueChange`.
- `onValueChange?: (value: string) => void` : fires when the highlighted item changes (arrow keys, pointer hover, programmatic).
- `shouldFilter?: boolean` (default `true`) : enable / disable cmdk's built-in filter+sort.
- `filter?: (value: string, search: string, keywords?: string[]) => number` : custom ranker. Return 0 to hide. Return any positive number to show ; higher = better rank.
- `loop?: boolean` (default `false`) : when true, ArrowDown past the last item wraps to the first.
- `label?: string` : accessible name for the listbox. Ignored when used inside CommandDialog (the Dialog's aria-labelledby takes over).

ALWAYS treat `value` (here on Command root) as the HIGHLIGHTED item, NOT the search query. The query lives on `<CommandInput value={query} onValueChange={setQuery}>`.

### CommandDialog

Wraps `Command` inside shadcn's `Dialog`. Source-verified at `apps/v4/registry/new-york-v4/ui/command.tsx` :

- Injects a `<DialogHeader className="sr-only"><DialogTitle>{title}</DialogTitle><DialogDescription>{description}</DialogDescription></DialogHeader>` so the dialog has an accessible name.
- Default `title` is "Command Palette", default `description` is "Search for a command to run...".
- Forwards every Dialog prop (`open`, `onOpenChange`, `defaultOpen`, `modal`, `container`) via spread.
- Applies a long `[&_[cmdk-*]]` Tailwind selector chain to style the embedded cmdk parts.

Use it for global launchers and modal palettes. NEVER add a visible DialogTitle child inside CommandDialog (the sr-only one already wires aria-labelledby).

### CommandInput

Search input. Controlled via `value` + `onValueChange`. cmdk normalises the value (trim + lowercase) before comparing to item values. shadcn wraps the input in a `<div data-slot="command-input-wrapper">` with a SearchIcon prefix.

```tsx
const [query, setQuery] = useState("")
<CommandInput value={query} onValueChange={setQuery} placeholder="Search..." />
```

ALWAYS use `value` + `onValueChange` when you need server-side filtering or analytics. For pure client-side filtering you can omit both and let cmdk track the query internally.

### CommandList

Scrollable container. Default styling : `max-h-[300px] scroll-py-1 overflow-x-hidden overflow-y-auto`. The `--cmdk-list-height` CSS variable is set by cmdk to the natural height of the visible items, enabling smooth height animation if you opt in.

### CommandEmpty

Renders automatically when the filtered result count is zero. REQUIRED for accessible empty-state communication. Without it, the list area silently collapses to zero height.

To customise the message with the live query :

```tsx
import { useCommandState } from "cmdk"

function EmptyWithQuery() {
  const search = useCommandState((state) => state.search)
  return <CommandEmpty>No results for "{search}".</CommandEmpty>
}
```

### CommandGroup

Groups items under a `heading` string. cmdk applies a `hidden` HTML attribute to the group when every child has been filtered out (the group does NOT unmount). Use the `forceMount` prop to keep the group visible regardless of filter.

### CommandItem

Selectable row. Key props :

- `value?: string` : filter key. If omitted, cmdk infers from `textContent`. ALWAYS provide explicitly for items whose visible text might collide or contain icons.
- `keywords?: string[]` : extra terms that match this item during filtering (treated as aliases ; trimmed).
- `onSelect?: (value: string) => void` : fires when the user picks this item (Enter or click). `value` is the trimmed `value` prop or inferred text.
- `disabled?: boolean` : when true, the item is skipped by keyboard nav and visually muted via `data-disabled="true"`.
- `forceMount?: boolean` : keep this item rendered even when the filter would hide it. Useful for "always-show" entries (e.g., "Create new...").

Data attributes : `data-selected="true"` when highlighted, `data-disabled="true"` when disabled.

### CommandSeparator

Visual `<div role="separator">`. Hidden via cmdk's `hidden` attribute while the search query is non-empty (because the filter has reordered items and separators break the visual hierarchy). Pass `alwaysRender` to force visibility.

### CommandShortcut

Pure `<span>` with `ml-auto text-xs tracking-widest text-muted-foreground`. Not a cmdk part ; shadcn-only addition. Place inside a CommandItem to render a keyboard-shortcut hint :

```tsx
<CommandItem value="profile">
  Profile
  <CommandShortcut>Ctrl+P</CommandShortcut>
</CommandItem>
```

## a11y : keyboard navigation comes from cmdk

cmdk implements the WAI-ARIA Combobox / Listbox keyboard model :

| Key | Behaviour |
|-----|-----------|
| `ArrowDown` | Move to next item. Wraps if `loop` is true. |
| `ArrowUp` | Move to previous item. Wraps if `loop` is true. |
| `Enter` | Fire `onSelect` on the highlighted item. |
| `Home` / `End` | Jump to first / last item. |
| Typing | Live-narrow the list via cmdk's filter. |

Inside CommandDialog the dialog adds Esc (close) and Tab (focus trap). cmdk handles the listbox roles ; you only need to provide unique item values and visible text.

## Cmd+K global hotkey : the canonical pattern

```tsx
"use client"

import { useEffect, useState } from "react"
import { CommandDialog, CommandInput, CommandList, CommandEmpty,
         CommandGroup, CommandItem } from "@/components/ui/command"

export function CommandMenu() {
  const [open, setOpen] = useState(false)

  useEffect(() => {
    function onKeyDown(e: KeyboardEvent) {
      if (e.key === "k" && (e.metaKey || e.ctrlKey)) {
        e.preventDefault()
        setOpen((current) => !current)
      }
    }
    document.addEventListener("keydown", onKeyDown)
    return () => document.removeEventListener("keydown", onKeyDown)
  }, [])

  return (
    <CommandDialog open={open} onOpenChange={setOpen}>
      <CommandInput placeholder="Type a command or search..." />
      <CommandList>
        <CommandEmpty>No results found.</CommandEmpty>
        <CommandGroup heading="Suggestions">
          <CommandItem value="calendar" onSelect={() => setOpen(false)}>
            Calendar
          </CommandItem>
        </CommandGroup>
      </CommandList>
    </CommandDialog>
  )
}
```

Three details that MUST be present :

1. `e.metaKey || e.ctrlKey` : covers macOS (Cmd) AND Windows / Linux (Ctrl) in one branch.
2. `e.preventDefault()` BEFORE the state toggle : Chrome maps Cmd+K to the URL-bar suggestions ; without preventDefault the browser intercepts the key.
3. `return () => document.removeEventListener("keydown", onKeyDown)` : the cleanup. Without it the listener stacks on every remount, the toggles compound, and React DevTools flags a leak.

NEVER place the listener on `window` and the cleanup on `document` (or vice versa) ; the removal MUST target the same target the registration used.

## Async filtering pattern : shouldFilter={false} + manual fetch

```tsx
const [query, setQuery] = useState("")
const [items, setItems] = useState<Item[]>([])
const [loading, setLoading] = useState(false)

useEffect(() => {
  if (!query) { setItems([]); return }
  const controller = new AbortController()
  setLoading(true)
  fetch(`/api/search?q=${encodeURIComponent(query)}`, { signal: controller.signal })
    .then((r) => r.json())
    .then((data) => { setItems(data); setLoading(false) })
    .catch(() => { /* aborted */ })
  return () => controller.abort()
}, [query])

return (
  <Command shouldFilter={false}>
    <CommandInput value={query} onValueChange={setQuery} />
    <CommandList>
      {loading && <CommandLoading>Searching…</CommandLoading>}
      <CommandEmpty>No results.</CommandEmpty>
      {items.map((it) => (
        <CommandItem key={it.id} value={it.id} onSelect={() => pick(it)}>
          {it.label}
        </CommandItem>
      ))}
    </CommandList>
  </Command>
)
```

Note : `CommandLoading` is imported directly from cmdk, NOT re-exported by shadcn's `command.tsx`. Import it as `import { CommandLoading } from "cmdk"` or wrap your own re-export.

ALWAYS pair `shouldFilter={false}` with a manual fetch + filter ; otherwise the input becomes decorative and the list shows every item.

## Companion Skills

- [shadcn-syntax-selectors](../shadcn-syntax-selectors/SKILL.md) (B5 sibling) : the Select vs Native Select vs Combobox vs Command-as-palette DECISION TREE. Read FIRST when choosing between primitives ; this skill assumes Command was already the right answer.
- [shadcn-syntax-dialog](../shadcn-syntax-dialog/SKILL.md) (B4) : the Dialog primitive that CommandDialog wraps internally. Read for the controlled-state contract (`open` + `onOpenChange`), sr-only DialogTitle pattern, Portal stacking, the `showCloseButton` prop that CommandDialog forwards.
- [shadcn-errors-cmdk-version-drift](../../shadcn-errors/shadcn-errors-cmdk-version-drift/SKILL.md) (B13 forward-pointer) : the three top-15 GitHub issues (#2944 / #2980 / #3051) that all trace to cmdk API drift between releases. Read when you hit "TypeError : undefined is not iterable" or "items unclickable" on an existing project that worked yesterday.
- [shadcn-syntax-drawer](../shadcn-syntax-drawer/SKILL.md) (B5 sibling) : the vaul-backed Drawer ; the Drawer-plus-Command pairing has a known focus quirk forwarded into the anti-patterns of this skill (entry #7 in `references/anti-patterns.md`).
- [shadcn-impl-component-install](../../shadcn-impl/shadcn-impl-component-install/SKILL.md) : the post-install fix-ups for cmdk version pinning ; relevant whenever `shadcn add command` overwrites a working file.

## Reference Links

- [references/methods.md](references/methods.md) : full prop signatures for all nine primitives, the three-arg `filter` signature, `shouldFilter` semantics, `value` + `onValueChange` contracts, `useCommandState` hook, data-attributes.
- [references/examples.md](references/examples.md) : five working recipes (Cmd+K dialog, async server filter with CommandLoading, palette with groups + shortcuts, nested sub-pages, Command inside Popover for Combobox).
- [references/anti-patterns.md](references/anti-patterns.md) : seven anti-patterns with WHY and FIX (stale filter arity, lowercase normalisation gotcha, missing CommandEmpty, leaked keydown listener, shouldFilter without manual filtering, duplicate item values, Drawer + Command focus loss).

## Sources

All claims in this skill trace to URLs in `SOURCES.md` :

- https://ui.shadcn.com/docs/components/radix/command (canonical Command docs, composition example, CommandDialog default props)
- https://cmdk.paco.me (underlying cmdk library docs, three-arg filter signature, shouldFilter / loop / forceMount props, useCommandState hook, async example)
- https://github.com/shadcn-ui/ui (apps/v4/registry/new-york-v4/ui/command.tsx, verbatim v4 source ; CommandDialog sr-only header pattern)
- https://github.com/shadcn-ui/ui/issues/2944 (Command/Combobox TypeError + unclickable items, 117 reactions, cmdk breaking change)
- https://github.com/shadcn-ui/ui/issues/2980 (Combobox "undefined is not iterable", 87 reactions)
- https://github.com/shadcn-ui/ui/issues/3051 (Combobox Array.from requires array-like, 42 reactions)

Verified 2026-05-19.
