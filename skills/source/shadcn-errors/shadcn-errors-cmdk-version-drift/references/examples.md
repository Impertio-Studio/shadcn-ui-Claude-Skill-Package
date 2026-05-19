# cmdk Version Drift : Examples

Each example pairs a WRONG and a RIGHT pattern with a short
explanation. All TypeScript snippets target cmdk 1.0.0+ unless
explicitly stated otherwise. Sources : shadcn registry
(https://ui.shadcn.com/r/styles/new-york/command.json), cmdk README
(https://github.com/pacocoursey/cmdk), issue #2944, issue #2980,
issue #3051, all verified 2026-05-19.

## §1. Filter Prop Signature : 2-arg vs 3-arg

### WRONG : 2-arg filter on cmdk 1.0.0+

```tsx
import { Command, CommandInput, CommandList, CommandItem }
  from "@/components/ui/command"

<Command
  filter={(value, search) => {
    // Only two args. The third (keywords) is silently ignored.
    return value.includes(search) ? 1 : 0
  }}
>
  <CommandInput placeholder="Search..." />
  <CommandList>
    <CommandItem
      value="profile"
      keywords={["account", "settings", "user"]}  // ← ignored at runtime
    >
      Profile
    </CommandItem>
  </CommandList>
</Command>
```

TypeScript does not error : a 2-arg function is structurally assignable
to a 3-arg type because extra parameters are allowed (contravariance).
At runtime, cmdk calls `filter(value, search, keywords)` but your
function body only references the first two. Searching for "user"
does NOT match the Profile item because the keywords array never
participated in the match.

### RIGHT : 3-arg filter that uses keywords

```tsx
<Command
  filter={(value, search, keywords) => {
    const corpus = (value + " " + (keywords ?? []).join(" ")).toLowerCase()
    return corpus.includes(search.toLowerCase()) ? 1 : 0
  }}
>
  <CommandInput placeholder="Search..." />
  <CommandList>
    <CommandItem
      value="profile"
      keywords={["account", "settings", "user"]}
    >
      Profile
    </CommandItem>
  </CommandList>
</Command>
```

Typing "user" now matches the Profile item because the keywords array
is concatenated into the haystack. Also note the `.toLowerCase()` on
both sides for case insensitivity (see §2).

## §2. Value Normalization : Case-Sensitive Match in a Custom Filter

### WRONG : case-sensitive equality in a custom filter

```tsx
<Command
  filter={(value, search) => {
    // Both arguments are raw, original-case strings.
    // value comes from `<CommandItem value="Apple">`.
    // search comes from the user typing "apple" into CommandInput.
    return value === search ? 1 : 0
  }}
>
  <CommandInput placeholder="Type a fruit..." />
  <CommandList>
    <CommandItem value="Apple">Apple</CommandItem>
    <CommandItem value="Banana">Banana</CommandItem>
  </CommandList>
</Command>
```

The user types `apple`. cmdk passes `value="Apple"` and `search="apple"`
to the custom filter. `"Apple" === "apple"` is `false`. Return is 0.
Item is hidden. User sees "No results" even though "apple" is obviously
in the list.

Note that the DEFAULT cmdk filter (no `filter` prop at all) would have
matched correctly because `command-score` lowercases internally. The
bug only appears WITH a custom filter.

### RIGHT : explicit case normalisation in the custom filter

```tsx
<Command
  filter={(value, search, keywords) => {
    const v = value.toLowerCase()
    const s = search.toLowerCase()
    const k = (keywords ?? []).join(" ").toLowerCase()
    return (v + " " + k).includes(s) ? 1 : 0
  }}
>
  <CommandInput placeholder="Type a fruit..." />
  <CommandList>
    <CommandItem value="Apple">Apple</CommandItem>
    <CommandItem value="Banana">Banana</CommandItem>
  </CommandList>
</Command>
```

Or, even simpler, delegate to cmdk's default by NOT passing `filter` :

```tsx
<Command>
  <CommandInput placeholder="Type a fruit..." />
  <CommandList>
    <CommandItem value="Apple">Apple</CommandItem>
    <CommandItem value="Banana">Banana</CommandItem>
  </CommandList>
</Command>
```

The default `command-score` filter handles case insensitivity, fuzzy
matching, and score-based sorting in one line. Custom `filter` is
ONLY worth writing when the default behaviour is wrong for your data.

## §3. Vaul Drawer + Command : Focus Duel and the onOpenAutoFocus Fix

### WRONG : Drawer + Command with no focus override

```tsx
import { Drawer, DrawerContent, DrawerTrigger }
  from "@/components/ui/drawer"
import { Command, CommandInput, CommandList, CommandItem }
  from "@/components/ui/command"

<Drawer>
  <DrawerTrigger>Open palette</DrawerTrigger>
  <DrawerContent>
    <Command>
      <CommandInput placeholder="Search..." />
      <CommandList>
        <CommandItem value="profile">Profile</CommandItem>
      </CommandList>
    </Command>
  </DrawerContent>
</Drawer>
```

Open the drawer. The cursor appears in the input briefly, then vanishes.
Type a key, nothing happens visible in the input ; the keystroke either
goes to `<body>` or to the drawer root. The user reports "the search
field will not accept typing in the drawer".

Root cause : vaul's default `onOpenAutoFocus` focuses the drawer root,
overriding cmdk's input autofocus on the next tick.

### RIGHT : preventDefault on onOpenAutoFocus

```tsx
<Drawer>
  <DrawerTrigger>Open palette</DrawerTrigger>
  <DrawerContent onOpenAutoFocus={(e) => e.preventDefault()}>
    <Command>
      <CommandInput placeholder="Search..." autoFocus />
      <CommandList>
        <CommandItem value="profile">Profile</CommandItem>
      </CommandList>
    </Command>
  </DrawerContent>
</Drawer>
```

The `preventDefault()` cancels vaul's default focus assignment, leaving
cmdk's input focus intact. The `autoFocus` attribute on CommandInput is
optional with most cmdk versions (it autofocuses by default) but it
makes the intent explicit.

The same fix applies to Dialog, Sheet, Popover, DropdownMenu when they
wrap a Command :

```tsx
<DialogContent onOpenAutoFocus={(e) => e.preventDefault()}>
  <Command>...</Command>
</DialogContent>
```

## §4. shouldFilter=false : Async Server-Side Filtering

### WRONG : shouldFilter=false but no manual filter

```tsx
import * as React from "react"
import { Command, CommandInput, CommandList, CommandItem, CommandEmpty }
  from "@/components/ui/command"

function ServerSearch() {
  const [search, setSearch] = React.useState("")
  const [items, setItems] = React.useState<string[]>([])

  React.useEffect(() => {
    fetch(`/api/search?q=${encodeURIComponent(search)}`)
      .then((r) => r.json())
      .then(setItems)
  }, [search])

  return (
    <Command shouldFilter={false}>
      <CommandInput
        value={search}
        onValueChange={setSearch}
        placeholder="Search server..."
      />
      <CommandList>
        <CommandEmpty>No results.</CommandEmpty>
        {/* WRONG : items is the FULL list returned by /api/search.
            cmdk does no filtering because shouldFilter=false.
            On every keystroke, every item from the API renders. */}
        {items.map((item) => (
          <CommandItem key={item} value={item}>{item}</CommandItem>
        ))}
      </CommandList>
    </Command>
  )
}
```

If `/api/search` returns 200 items and you do not filter them in the
parent (or the server does not filter by `q`), every keystroke renders
all 200 items. The user thinks the filter is broken.

### RIGHT : shouldFilter=false WITH manual filter and a loading state

```tsx
import * as React from "react"
import { Command, CommandInput, CommandList, CommandItem, CommandEmpty }
  from "@/components/ui/command"
import { Command as CommandPrimitive } from "cmdk"

const CommandLoading = CommandPrimitive.Loading

function ServerSearch() {
  const [search, setSearch] = React.useState("")
  const [items, setItems] = React.useState<string[]>([])
  const [loading, setLoading] = React.useState(false)

  React.useEffect(() => {
    if (!search) {
      setItems([])
      return
    }
    setLoading(true)
    const ctrl = new AbortController()
    fetch(`/api/search?q=${encodeURIComponent(search)}`, {
      signal : ctrl.signal,
    })
      .then((r) => r.json())
      .then((data) => {
        setItems(data)
        setLoading(false)
      })
      .catch(() => { /* aborted */ })
    return () => ctrl.abort()
  }, [search])

  return (
    <Command shouldFilter={false}>
      <CommandInput
        value={search}
        onValueChange={setSearch}
        placeholder="Search server..."
      />
      <CommandList>
        {loading && <CommandLoading>Searching...</CommandLoading>}
        {!loading && items.length === 0 && (
          <CommandEmpty>No results.</CommandEmpty>
        )}
        {items.map((item) => (
          <CommandItem key={item} value={item}>{item}</CommandItem>
        ))}
      </CommandList>
    </Command>
  )
}
```

Server filters by `q`. Client never re-filters (shouldFilter=false).
CommandLoading is imported from `cmdk` directly because the shadcn
wrapper does not re-export it. AbortController prevents stale results
overwriting newer ones.

## §5. CommandLoading Import : Two Valid Paths

### WRONG : import CommandLoading from the shadcn wrapper

```tsx
import {
  Command,
  CommandInput,
  CommandList,
  CommandItem,
  CommandLoading,        // ← not exported
} from "@/components/ui/command"
```

Compilation error : `"CommandLoading" is not exported from
"@/components/ui/command"`. Or worse, in some bundler configurations,
runtime undefined that produces a blank node where the loading
indicator should be.

### RIGHT (Path A) : import directly from cmdk

```tsx
import {
  Command,
  CommandInput,
  CommandList,
  CommandItem,
} from "@/components/ui/command"
import { Command as CommandPrimitive } from "cmdk"

const CommandLoading = CommandPrimitive.Loading

// usage :
<CommandList>
  {loading && <CommandLoading>Searching...</CommandLoading>}
  ...
</CommandList>
```

This works regardless of whether the shadcn wrapper changes in the
future. Stable across `shadcn add command --overwrite`.

### RIGHT (Path B) : extend the wrapper to re-export it

Edit `components/ui/command.tsx` :

```tsx
// ... existing wrapper code that imports Command, List, Item, ... from cmdk
import { Command as CommandPrimitive } from "cmdk"

// ... existing styled component definitions ...

const CommandLoading = CommandPrimitive.Loading

export {
  Command,
  CommandDialog,
  CommandInput,
  CommandList,
  CommandEmpty,
  CommandGroup,
  CommandItem,
  CommandSeparator,
  CommandShortcut,
  CommandLoading,        // ← added
}
```

Then import from the wrapper as usual :

```tsx
import { CommandLoading } from "@/components/ui/command"
```

Trade-off : Path B makes the wrapper non-re-add-safe. The next
`shadcn add command --overwrite` will wipe the export. Document this in
your project README, or prefer Path A.

## §6. Pinning cmdk : package.json Patterns

### WRONG : floating range on cmdk

```json
{
  "dependencies" : {
    "cmdk" : "^0.2.0"
  }
}
```

`pnpm install` on March 8, 2024 silently upgraded cmdk to 1.0.0 because
1.0.0 was technically a new major and `^0.2.0` matches `>=0.2.0 <1.0.0`
... but many lockfiles were generated when 1.0.0 was already published
and certain pnpm versions resolved `^0.2.0` permissively. Combined
with a CI that ran `pnpm install --frozen-lockfile=false`, the upgrade
was invisible. The wrapper file (generated against 0.2.x) was suddenly
running on 1.0.0 cmdk. Symptom : CommandItem outside CommandList
crash. See issue #2944.

### RIGHT : exact pin matching the shadcn registry

```json
{
  "dependencies" : {
    "cmdk" : "1.0.4"
  }
}
```

No carat, no tilde. Lockfile reflects the exact version. CI uses
`pnpm install --frozen-lockfile=true`. Any future cmdk bump is an
explicit, reviewable PR that pairs with `shadcn add command --overwrite`.

### How to look up the registry's expected version

```bash
curl -s https://ui.shadcn.com/r/styles/new-york/command.json | jq '.dependencies'
```

Output (verified 2026-05-19, exact version may differ on later
verification dates) :

```json
[
  "cmdk"
]
```

The shadcn registry typically lists `cmdk` as a bare dependency name
(version is resolved at install time by the user's pnpm/npm). For the
specific version the registry's wrapper file was authored against,
inspect the wrapper file itself :

```bash
grep -A1 "from \"cmdk\"" components/ui/command.tsx
# Then cross-check against the cmdk CHANGELOG to identify which
# API surface (0.2.x or 1.0.x) the wrapper assumes.
```

Or, after `shadcn add command`, look at the resolved version in your
lockfile :

```bash
grep -A3 "  cmdk@" pnpm-lock.yaml | head -10
```
