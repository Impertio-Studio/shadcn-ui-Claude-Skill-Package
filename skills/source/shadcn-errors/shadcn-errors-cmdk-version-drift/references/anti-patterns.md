# cmdk Version Drift : Anti-Patterns

Each entry pairs a concrete WRONG pattern with the WHY (root cause)
and the right thing to do instead. Sources cross-referenced in
SKILL.md and methods.md. Verified against shadcn registry
(https://ui.shadcn.com/), cmdk repo (https://github.com/pacocoursey/cmdk),
and shadcn-ui/ui issues #2944, #2980, #3051.

## 1. The 2-Arg Filter on cmdk 1.0.0+ (Third Argument Silently Dropped)

WRONG :

```tsx
<Command filter={(value, search) => value.includes(search) ? 1 : 0}>
  <CommandList>
    <CommandItem value="profile" keywords={["account", "user"]}>
      Profile
    </CommandItem>
  </CommandList>
</Command>
```

Why this fails : the cmdk 1.0.0 filter signature is
`(value, search, keywords) => number`. TypeScript allows a 2-arg
function in a 3-arg position because of structural-subtyping
contravariance on parameter lists. At runtime cmdk passes the
`keywords` array as the third argument, but the function body never
references it. The per-item `keywords` prop is silently ignored. Users
searching for "user" or "account" never match the Profile item.

The compiler does not warn. The lint does not warn. The only signal is
"my keywords-based search does not work".

Right thing : ALWAYS write filters as 3-arg on cmdk 1.0.0+ even when
you do not currently use keywords, so that adding `keywords` later
"just works" :

```tsx
<Command
  filter={(value, search, keywords) => {
    const corpus = value + " " + (keywords ?? []).join(" ")
    return corpus.toLowerCase().includes(search.toLowerCase()) ? 1 : 0
  }}
>
```

If you are on cmdk 0.2.x, the 2-arg form is correct but the keywords
prop on CommandItem does not exist there either. Verify with
`pnpm ls cmdk`.

## 2. Case-Sensitive Comparison in a Custom Filter (Silent Miss)

WRONG :

```tsx
<Command filter={(value, search) => value === search ? 1 : 0}>
  <CommandList>
    <CommandItem value="Apple">Apple</CommandItem>
  </CommandList>
</Command>
```

Why this fails : cmdk trims `data-value` but does NOT lowercase it.
The DEFAULT filter (command-score) is case-insensitive because the
algorithm lowercases internally inside its `formatInput` helper. A
CUSTOM filter receives raw, original-case `value` and `search`.

User types `apple`. cmdk calls `filter("Apple", "apple", undefined)`.
The strict equality is false. The item is hidden. The bug looks like
"the filter is broken" but is in fact "the custom filter does what
you literally wrote".

Right thing : either delete the custom `filter` prop and rely on the
case-insensitive default, or explicitly lowercase both sides in the
custom filter :

```tsx
<Command
  filter={(value, search) => {
    return value.toLowerCase().includes(search.toLowerCase()) ? 1 : 0
  }}
>
```

If the data is already canonical (e.g. machine IDs, slugs), keep
case-sensitive equality but document the assumption. Most consumer-
facing palette use cases want case-insensitive.

## 3. Vaul Drawer Wrapping Command Without onOpenAutoFocus Override (Focus Loss)

WRONG :

```tsx
<Drawer>
  <DrawerTrigger>Open palette</DrawerTrigger>
  <DrawerContent>
    <Command>
      <CommandInput placeholder="Search..." />
      <CommandList>...</CommandList>
    </Command>
  </DrawerContent>
</Drawer>
```

Why this fails : vaul applies its own focus trap on Drawer open and
focuses `<DrawerContent>` root on mount. cmdk's `<CommandInput>` also
auto-focuses on mount. Two effects fire in the same commit cycle, vaul
wins on the next tick because its effect is registered later in the
content lifecycle. The input loses focus immediately after gaining it.
First keystroke lands on `<body>` or the drawer root, not the input.

User reports : "I open the drawer but cannot type in the search field".
The bug is layout-position-dependent and reproduces only on the FIRST
open because subsequent re-opens hit a cached focus state in some
browsers.

Right thing : tell vaul not to fight cmdk :

```tsx
<DrawerContent onOpenAutoFocus={(e) => e.preventDefault()}>
  <Command>
    <CommandInput placeholder="Search..." />
    ...
  </Command>
</DrawerContent>
```

The same fix applies to any Radix overlay wrapping a Command : Dialog,
Sheet, Popover, DropdownMenu, ContextMenu. The
`onOpenAutoFocus={(e) => e.preventDefault()}` idiom is the universal
"yield to inner autofocus" signal.

## 4. shouldFilter=false Without Manual Filtering (Full List on Every Keystroke)

WRONG :

```tsx
const [search, setSearch] = useState("")
const [items, setItems] = useState<string[]>([])

useEffect(() => {
  fetch(`/api/search?q=${search}`).then(r => r.json()).then(setItems)
}, [search])

return (
  <Command shouldFilter={false}>
    <CommandInput value={search} onValueChange={setSearch} />
    <CommandList>
      {items.map(i => <CommandItem key={i} value={i}>{i}</CommandItem>)}
    </CommandList>
  </Command>
)
```

Why this fails : `shouldFilter={false}` disables cmdk's filter ENTIRELY.
cmdk renders every CommandItem you give it. The intent of the code is
clearly "let the server filter", but if the `/api/search` endpoint
ignores `q` (or returns the full table because the param is missing,
or returns stale data because of a race), the user sees the entire
catalog on every keystroke.

The bug is amplified by the fact that the server might return 5 items
on a slow query and 500 items on a fast one, so the list flickers
between scoped and full results unpredictably.

Right thing :

1. Verify the server actually filters by `q`. Use a curl test :
   `curl '/api/search?q=appl' | jq length`. If it returns the same
   count as `curl '/api/search?q=' | jq length`, the server is the bug.
2. If client-side filtering is needed, do it in the parent component
   BEFORE passing to CommandList :

```tsx
const filtered = items.filter(i =>
  i.toLowerCase().includes(search.toLowerCase())
)
return (
  <Command shouldFilter={false}>
    <CommandInput value={search} onValueChange={setSearch} />
    <CommandList>
      {filtered.map(i => <CommandItem key={i} value={i}>{i}</CommandItem>)}
    </CommandList>
  </Command>
)
```

3. ALWAYS pair `shouldFilter={false}` with a `<CommandLoading>` indicator
   so users get feedback during the async fetch. See examples.md §4 for
   the complete pattern including AbortController.

## 5. Importing CommandLoading from the shadcn Wrapper (Not Re-Exported)

WRONG :

```tsx
import {
  Command,
  CommandInput,
  CommandList,
  CommandLoading,
} from "@/components/ui/command"
```

Why this fails : the shadcn registry's wrapper at
`components/ui/command.tsx` re-exports exactly nine names (Command,
CommandDialog, CommandInput, CommandList, CommandEmpty, CommandGroup,
CommandItem, CommandSeparator, CommandShortcut). `CommandLoading` is
NOT in that list, even though cmdk itself exports `Command.Loading`.

Symptoms vary by bundler :

- Vite + tsc strict : "Module has no exported member 'CommandLoading'"
- Webpack with non-strict imports : silent `undefined` at runtime,
  the component renders nothing where the loading indicator should be
- esbuild : same as Vite

The bug is misdiagnosed as "cmdk is broken" or "shadcn dropped the
loading component" when in fact the wrapper simply does not re-export it.

Right thing (preferred) : import from cmdk directly.

```tsx
import { Command as CommandPrimitive } from "cmdk"
const CommandLoading = CommandPrimitive.Loading
```

Right thing (alternative) : extend the wrapper to re-export. Edit
`components/ui/command.tsx` to add `CommandLoading` to the export
list. Trade-off : the wrapper is no longer re-add-safe ; the next
`shadcn add command --overwrite` removes your edit. Document the
extension in a code comment :

```tsx
// LOCAL EXTENSION : re-export CommandLoading.
// The shadcn registry does NOT include this. After every
// `shadcn add command --overwrite`, re-apply this export.
const CommandLoading = CommandPrimitive.Loading
export { /* ... */, CommandLoading }
```

## 6. Floating cmdk Version in package.json (Unpredictable Filter Signature)

WRONG :

```json
{
  "dependencies" : {
    "cmdk" : "^0.2.0"
  }
}
```

Why this fails : `^0.2.0` matches `>=0.2.0 <1.0.0` in strict semver but
pre-1.0 semver is by-convention "anything goes". Some package managers
(pnpm older minor versions, certain yarn versions) and some lockfile
states permit auto-upgrade across the 0.2.x to 1.0.0 boundary on
`pnpm install` without `--frozen-lockfile`. The wrapper file in
`components/ui/command.tsx` was generated against a specific cmdk
version. After the silent upgrade, the wrapper and the installed cmdk
disagree on : filter arity, the presence of `keywords` Item prop, the
list-scoped registry requirement, and the export surface of
`Command.*`.

The drift manifests three ways :

1. "TypeError : undefined is not iterable (cannot read property
   Symbol(Symbol.iterator))" on first render (CommandItem outside
   CommandList because the wrapper renders an old shape, but cmdk
   1.0.0 enforces the new structural rule).
2. "Items are unclickable / disabled" because cmdk 1.0.0 changed the
   internal selected-state mechanism and the wrapper's class names no
   longer match.
3. Per-item `keywords` array is undefined at runtime because the
   wrapper does not pass it.

This is the literal sequence reported in issue #2944 (117 reactions,
the top cmdk drift issue), issue #2980 (87 reactions, "CommandItem
not clickable"), and issue #3051 (42 reactions, "Combobox not
filtering after update").

Right thing : pin cmdk to an exact version and align the wrapper :

```json
{
  "dependencies" : {
    "cmdk" : "1.0.4"
  }
}
```

```bash
pnpm install --frozen-lockfile=true
pnpm dlx shadcn@latest add command --diff
pnpm dlx shadcn@latest add command --overwrite
```

Document the cmdk pin in your project README. Treat any cmdk bump as
a coordinated change : version bump + wrapper regenerate + filter
arity audit + smoke test on Command, Combobox, Drawer-wrapping-Command,
Dialog-wrapping-Command. Never bump cmdk in isolation.

## 7. CommandItem Rendered Outside CommandList (Crash on cmdk 1.0.0+)

WRONG :

```tsx
<Command>
  <CommandInput placeholder="Search..." />
  <CommandItem value="profile">Profile</CommandItem>
  <CommandItem value="settings">Settings</CommandItem>
</Command>
```

Why this fails : cmdk 1.0.0 introduced a list-scoped item registry.
The internal `useCmdk()` hook iterates `commandListRef.current.items`.
If `CommandList` is not the parent of the items, that ref is never
populated, and the iteration crashes :

```
TypeError : undefined is not iterable (cannot read property
Symbol(Symbol.iterator))
```

The crash is unconditional : it happens on first render, blocking the
entire route. Reported in issue #2944 as the canonical sign of the
cmdk 0.2.x to 1.0.0 break, often surfacing AFTER a `pnpm install` that
silently upgraded cmdk while the wrapper still rendered the old shape.

Right thing : wrap every CommandItem in CommandList. CommandEmpty,
CommandGroup, and CommandSeparator must also live inside CommandList.
Only CommandInput is allowed (and required) to be a sibling of
CommandList, not a child of it.

```tsx
<Command>
  <CommandInput placeholder="Search..." />
  <CommandList>
    <CommandEmpty>No results.</CommandEmpty>
    <CommandItem value="profile">Profile</CommandItem>
    <CommandItem value="settings">Settings</CommandItem>
  </CommandList>
</Command>
```

If you see this crash AFTER an upgrade, it means cmdk drifted but
the wrapper did not. Run `shadcn add command --overwrite` to refresh
the wrapper, OR roll cmdk back to the pre-1.0 version your wrapper
was generated against.
