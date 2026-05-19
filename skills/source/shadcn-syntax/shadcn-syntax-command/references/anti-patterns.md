# Command : Anti-Patterns

Seven concrete failures that block real shadcn ui Command usage. Each entry follows : WRONG code, WHY it fails, FIX.

Sources : `apps/v4/registry/new-york-v4/ui/command.tsx`, https://cmdk.paco.me, https://ui.shadcn.com/docs/components/radix/command, vooronderzoek §9 entry 10 + §10 issues #2944 / #2980 / #3051, and the cmdk issue tracker. Verified 2026-05-19.

The first three anti-patterns are forward-pointers to `shadcn-errors-cmdk-version-drift` ; the deep dive into the three top-15 GitHub issues lives there. The remaining four are Command-specific composition failures.

## 1. `filter` prop written with the stale two-argument signature

### WRONG

```tsx
<Command
  filter={(value, search) => {
    return value.toLowerCase().includes(search.toLowerCase()) ? 1 : 0
  }}
>
  <CommandInput placeholder="Search..." />
  <CommandList>
    <CommandItem value="apple" keywords={["fruit", "red"]}>Apple</CommandItem>
    <CommandItem value="banana" keywords={["fruit", "yellow"]}>Banana</CommandItem>
  </CommandList>
</Command>
```

### WHY it fails

The latest cmdk filter signature accepts THREE arguments : `(value, search, keywords) => number`. The two-argument form still compiles (TypeScript happily widens the function type), but the `keywords` array NEVER reaches the filter callback ; cmdk passes the third argument and your function silently discards it. Typing "red" matches nothing because the filter only inspects `value` ("apple", "banana"), never the aliases.

AI training data routinely emits the older two-arg form because that was the documented signature from 2022 to early 2024. Snippets copied from old blog posts replicate the bug.

Issue #2944 (117 reactions) and the related #2980 / #3051 are all version-drift symptoms ; the filter-signature change is one cause among three (see `shadcn-errors-cmdk-version-drift`).

### FIX

ALWAYS use the three-argument signature ; fold the keywords into the comparison :

```tsx
<Command
  filter={(value, search, keywords) => {
    const haystack = (value + " " + (keywords ?? []).join(" ")).toLowerCase()
    return haystack.includes(search.toLowerCase()) ? 1 : 0
  }}
>
```

Note that cmdk has ALREADY lowercased and trimmed `value` and `search` by the time the callback runs ; the explicit `.toLowerCase()` on the right side of the includes is defensive but harmless. `keywords` is the raw array as you passed it on each CommandItem ; trim and lowercase it yourself if you want consistency with `value`.

## 2. Expecting case-sensitive matching (value-normalisation gotcha)

### WRONG

```tsx
const products = [
  { id: "iPhone", label: "iPhone 16 Pro" },
  { id: "iPad", label: "iPad Air" },
]

<Command>
  <CommandInput />
  <CommandList>
    {products.map((p) => (
      <CommandItem
        key={p.id}
        value={p.id}
        onSelect={(v) => {
          // BUG : v is "iphone", not "iPhone"
          const product = products.find((it) => it.id === v)
          openProduct(product) // crashes : product is undefined
        }}
      >
        {p.label}
      </CommandItem>
    ))}
  </CommandList>
</Command>
```

### WHY it fails

cmdk normalises every `value` (on Command root, on CommandInput, AND on CommandItem) to its TRIMMED, LOWERCASED form before any internal comparison. The `value` argument passed to `onSelect` is therefore "iphone", not "iPhone". The strict equality `it.id === v` evaluates false for every product, `find` returns undefined, and the next line crashes.

This is documented in the cmdk README ("Values are always trimmed with the trim() method") but the lowercase part is only implied by the case-insensitive filter behaviour. AI-generated code routinely treats `onSelect`'s argument as the verbatim `value` prop.

### FIX

Either (A) store the lookup key in a separate field that the value already represents :

```tsx
<CommandItem
  value={p.id} // already lowercase in your data
  onSelect={(v) => openProduct(products.find((it) => it.id === v))}
>
```

Or (B) compare case-insensitively :

```tsx
onSelect={(v) => {
  const product = products.find((it) => it.id.toLowerCase() === v)
  openProduct(product)
}}
```

Or (C) close over the loop-local variable :

```tsx
{products.map((p) => (
  <CommandItem
    key={p.id}
    value={p.id}
    onSelect={() => openProduct(p)}  // p closed over, no lookup needed
  >
    {p.label}
  </CommandItem>
))}
```

Option (C) is the cleanest for static lists. Option (A) requires lowercase data. Option (B) is the safest retrofit for legacy data.

## 3. `shouldFilter={false}` without doing the manual filter

### WRONG

```tsx
const [query, setQuery] = useState("")

return (
  <Command shouldFilter={false}>
    <CommandInput value={query} onValueChange={setQuery} />
    <CommandList>
      <CommandEmpty>No results.</CommandEmpty>
      {ALL_ITEMS.map((it) => (
        <CommandItem key={it.id} value={it.id}>
          {it.label}
        </CommandItem>
      ))}
    </CommandList>
  </Command>
)
```

### WHY it fails

`shouldFilter={false}` disables cmdk's built-in filter. With the flag off, cmdk renders every CommandItem verbatim regardless of what the user types. The search input becomes decorative : every keystroke updates `query` state but the visible list never narrows. CommandEmpty never renders because there are always items in the list.

This is the most common cause of the "command not filtering" bug report. Users see the flag in tutorials about server-side search and copy it without reading the surrounding context.

### FIX

Either (A) keep cmdk's default filter (drop the `shouldFilter` prop entirely) :

```tsx
return (
  <Command>
    <CommandInput />
    <CommandList>
      <CommandEmpty>No results.</CommandEmpty>
      {ALL_ITEMS.map((it) => (
        <CommandItem key={it.id} value={it.id}>{it.label}</CommandItem>
      ))}
    </CommandList>
  </Command>
)
```

Or (B) own the filter yourself in JavaScript :

```tsx
const filtered = ALL_ITEMS.filter((it) =>
  it.label.toLowerCase().includes(query.toLowerCase())
)

return (
  <Command shouldFilter={false}>
    <CommandInput value={query} onValueChange={setQuery} />
    <CommandList>
      {filtered.length === 0 && <CommandEmpty>No results.</CommandEmpty>}
      {filtered.map((it) => (
        <CommandItem key={it.id} value={it.id}>{it.label}</CommandItem>
      ))}
    </CommandList>
  </Command>
)
```

ALWAYS pick one. NEVER set `shouldFilter={false}` and rely on cmdk to still narrow the list.

## 4. Missing CommandEmpty (silent empty list)

### WRONG

```tsx
<Command>
  <CommandInput placeholder="Search..." />
  <CommandList>
    <CommandGroup heading="Fruits">
      <CommandItem value="apple">Apple</CommandItem>
      <CommandItem value="banana">Banana</CommandItem>
    </CommandGroup>
  </CommandList>
</Command>
```

### WHY it fails

When the user types "xyz" and zero items match, cmdk hides the group via the `hidden` HTML attribute. The list area collapses to its padding. There is no visible message ; the user sees a near-empty box and assumes the component is broken or that they need to keep typing. Screen-reader users hear nothing at all.

cmdk auto-renders CommandEmpty only IF the element is present in the tree. If it is not, the empty state is silent.

### FIX

Always include CommandEmpty inside CommandList :

```tsx
<Command>
  <CommandInput placeholder="Search..." />
  <CommandList>
    <CommandEmpty>No results found.</CommandEmpty>
    <CommandGroup heading="Fruits">
      <CommandItem value="apple">Apple</CommandItem>
      <CommandItem value="banana">Banana</CommandItem>
    </CommandGroup>
  </CommandList>
</Command>
```

For a dynamic message that shows the failed query, use `useCommandState` from cmdk :

```tsx
import { useCommandState } from "cmdk"

function EmptyWithQuery() {
  const search = useCommandState((state) => state.search)
  return <CommandEmpty>No results for "{search}".</CommandEmpty>
}
```

CommandEmpty must be a DIRECT or NESTED child of CommandList ; rendering it as a sibling of CommandList (outside the list) breaks cmdk's auto-toggle.

## 5. Cmd+K listener registered without removeEventListener cleanup

### WRONG

```tsx
export function CommandMenu() {
  const [open, setOpen] = useState(false)

  useEffect(() => {
    document.addEventListener("keydown", (e) => {
      if (e.key === "k" && (e.metaKey || e.ctrlKey)) {
        e.preventDefault()
        setOpen((current) => !current)
      }
    })
    // NO return ! The cleanup is missing.
  }, [])

  return <CommandDialog open={open} onOpenChange={setOpen}>...</CommandDialog>
}
```

### WHY it fails

The effect adds a listener on every mount. The component remounts on hot reload, on parent state changes that re-key the tree, and (most importantly) every time React StrictMode runs the effect twice in development. Without `removeEventListener`, the listener accumulates : after three mounts there are three listeners, each toggling `open` independently. The first Cmd+K opens AND closes AND opens again, depending on parity, all in a single key press.

Additional consequence : the inline arrow function is a NEW reference on every render, so even when you DO add a cleanup, it cannot remove the same function ; the cleanup silently fails. React DevTools / Chrome's Performance tab flag the leak.

### FIX

Hoist the handler to a NAMED function so cleanup can target the same reference, and return the cleanup :

```tsx
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
```

The cleanup MUST target the same event target (`document`) and the same function reference (`onKeyDown`). NEVER mix `window.addEventListener` with `document.removeEventListener` ; the removal is silently a no-op.

## 6. Duplicate `value` props collapse items and break keyboard nav

### WRONG

```tsx
<Command>
  <CommandInput />
  <CommandList>
    <CommandEmpty>No results.</CommandEmpty>
    <CommandGroup heading="Recent">
      <CommandItem value="open">Open file</CommandItem>
      <CommandItem value="open">Open recent</CommandItem>
      <CommandItem value="open">Open folder</CommandItem>
    </CommandGroup>
  </CommandList>
</Command>
```

### WHY it fails

cmdk uses `value` as the identity key for filtering, scoring, and keyboard-navigation tracking. Two CommandItem elements with the same `value` collide in cmdk's internal store. Symptoms vary by version : ArrowDown stops mid-list ; only the first duplicate is reachable ; `onSelect` fires on the wrong item ; or React logs a duplicate-key warning if cmdk also uses `value` as a React key.

This is also a common cause of "items unclickable" bug reports (issue #2944) when the duplicate values arise from auto-inferred `textContent` (two items with identical visible text, no explicit `value`).

### FIX

Always give every CommandItem a unique `value`. Use a stable identifier from your data model :

```tsx
<CommandGroup heading="Recent">
  <CommandItem value="open-file">Open file</CommandItem>
  <CommandItem value="open-recent">Open recent</CommandItem>
  <CommandItem value="open-folder">Open folder</CommandItem>
</CommandGroup>
```

When the items come from a list, derive `value` from the item ID :

```tsx
{recents.map((r) => (
  <CommandItem key={r.id} value={r.id} onSelect={() => openFile(r.path)}>
    {r.label}
  </CommandItem>
))}
```

NEVER rely on auto-inferred `textContent` for items that share visible text. ALWAYS provide an explicit `value` when items contain only icons + tooltips, where the inferred text would be empty.

## 7. Command inside a vaul Drawer loses input focus

### WRONG

```tsx
<Drawer open={open} onOpenChange={setOpen}>
  <DrawerContent>
    <Command>
      <CommandInput placeholder="Search..." />
      <CommandList>
        <CommandEmpty>No results.</CommandEmpty>
        <CommandItem value="apple">Apple</CommandItem>
      </CommandList>
    </Command>
  </DrawerContent>
</Drawer>
```

### WHY it fails

vaul (the Drawer library) manages its own focus trap and runs `onOpenAutoFocus` to move focus to the first focusable element inside `DrawerContent`. cmdk's CommandInput is inside a wrapper `<div>` ; depending on vaul's focus-discovery order, vaul may land on the wrapper instead of the input, or move focus AWAY from the input when the Drawer's height animation completes. The user sees the Drawer open, types, and nothing happens because the input never received focus.

This is the third member of the cmdk-version-drift family : it is not a cmdk bug per se, but a focus-management collision between vaul and cmdk that the shadcn registry does not paper over. See `shadcn-errors-cmdk-version-drift` for the full trace.

### FIX

Override vaul's auto-focus and target the CommandInput explicitly :

```tsx
import { useRef, useEffect } from "react"

const inputRef = useRef<HTMLInputElement>(null)

<Drawer open={open} onOpenChange={setOpen}>
  <DrawerContent
    onOpenAutoFocus={(e) => {
      e.preventDefault()
      // Wait for the drawer height animation, then focus the input.
      setTimeout(() => inputRef.current?.focus(), 100)
    }}
  >
    <Command>
      <CommandInput ref={inputRef} placeholder="Search..." />
      <CommandList>
        <CommandEmpty>No results.</CommandEmpty>
        <CommandItem value="apple">Apple</CommandItem>
      </CommandList>
    </Command>
  </DrawerContent>
</Drawer>
```

Three details that make this work :

1. `e.preventDefault()` on `onOpenAutoFocus` stops vaul from picking its own focus target.
2. The `setTimeout` of ~100ms waits for vaul's slide-in animation to finish ; focusing before the transition completes triggers a re-blur on transition-end.
3. The `ref` lives on `CommandInput` (which forwards to the underlying input) ; the wrapper `<div>` is NOT focusable.

For the equivalent Drawer-on-mobile / Dialog-on-desktop responsive pattern, see `shadcn-impl-responsive-dialog-drawer`. The same focus-forwarding fix applies inside Drawer ; Dialog handles auto-focus correctly out of the box because shadcn's CommandDialog already wraps Command in Dialog.
