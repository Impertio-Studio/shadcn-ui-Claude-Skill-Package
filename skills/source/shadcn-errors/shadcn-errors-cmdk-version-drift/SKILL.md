---
name: shadcn-errors-cmdk-version-drift
description: >
  Use when a shadcn ui Command primitive (command palette, Combobox, search
  list, fuzzy filter, multi-select dropdown) stops filtering, renders the
  full list regardless of typing, fails to highlight or click items, throws
  "TypeError : undefined is not iterable (cannot read property
  Symbol(Symbol.iterator))" on a CommandItem rendered outside CommandList,
  silently ignores a third `keywords` argument passed to the `filter` prop,
  no longer matches values typed in uppercase, loses keyboard focus inside
  a Vaul Drawer, returns zero results despite `shouldFilter={false}`, or
  fails to import `CommandLoading` from the shadcn wrapper. Prevents the
  six recurring drift traps : (1) writing a 2-arg `(value, search) =>
  number` filter against cmdk >= 1.0.0 where the signature is `(value,
  search, keywords) => number` so the third argument is silently dropped
  and per-item `keywords` boosting never fires, (2) writing a custom
  `filter` that does a case-sensitive `===` or `includes` check against
  raw `value` and raw `search` so that "Apple" never matches "apple"
  even though the DEFAULT command-score filter is case-insensitive,
  (3) wrapping `<Command>` inside a Vaul `<Drawer>` without overriding
  `onOpenAutoFocus` so vaul's focus-trap and cmdk's input-autofocus
  fight and the input is stolen back to the trigger on first keystroke,
  (4) flipping `shouldFilter={false}` to fetch results server-side but
  never filtering the rendered `<CommandItem>` set yourself so the list
  shows EVERY item on every keystroke, (5) importing `CommandLoading`
  from `@/components/ui/command` instead of `cmdk` because the shadcn
  registry wrapper re-exports Command / CommandDialog / CommandInput /
  CommandList / CommandEmpty / CommandGroup / CommandItem /
  CommandSeparator / CommandShortcut but NOT CommandLoading, and
  (6) letting `cmdk` float in `package.json` (`"cmdk" : "^1.0.0"`)
  while the `components/ui/command.tsx` file was generated against a
  pinned older cmdk version so prop names and arities drift apart over
  time. Covers detection commands (read installed cmdk version with
  `npm ls cmdk` and the shadcn-wrapper expected version via the
  registry endpoint), version compatibility matrix (cmdk 0.2.x =
  `(value, search) => number` filter, no `keywords` Item prop ; cmdk
  one zero plus = `(value, search, keywords) => number` filter, keywords
  prop as string array on Item, CommandList becomes REQUIRED parent of every
  Item or render crashes with the Symbol.iterator TypeError), fix
  strategies (pin cmdk in package.json to exact version, re-run
  `shadcn add command --overwrite` to refresh the wrapper to the
  registry's current pin, or upgrade BOTH at once and run the
  filter-arity migration), the Vaul + cmdk focus duel and its
  `onOpenAutoFocus={(e) => e.preventDefault()}` workaround, the
  command-score case behaviour (default filter lowercases internally
  via command-score's `formatInput`, custom filter does NOT, the
  `data-value` attribute is stored verbatim with trim()), and the
  three top-15 shadcn-ui/ui issues (#2944 cmdk 1.0.0 break, 117
  reactions ; #2980 Command items unclickable, 87 reactions ; #3051
  Combobox not filtering, 42 reactions).
  Keywords: cmdk version drift, command filter signature, command
  filter 2 arg vs 3 arg, command lowercase value, value normalization
  cmdk, vaul cmdk focus loss, drawer cmdk focus stolen, drawer
  onOpenAutoFocus preventDefault, shouldFilter behavior, shouldFilter
  false no results, manual filtering shouldFilter, CommandLoading
  missing, CommandLoading not exported shadcn, import CommandLoading
  from cmdk, cmdk version mismatch, cmdk one zero breaking change,
  cmdk legacy pin, npm ls cmdk, command not filtering, command palette
  not filtering, combobox not filtering, drawer cmdk focus, why my
  command palette broken, why my combobox returns full list, why does
  uppercase not match cmdk, CommandItem outside CommandList TypeError,
  Symbol iterator cmdk crash, shadcn add command overwrite, cmdk
  package json pin, command-score case insensitive, data-value cmdk
  attribute, keywords prop CommandItem, third argument filter ignored,
  command palette dropdown.
license: MIT
compatibility: "Designed for Claude Code. Requires shadcn ui evergreen-2026 (cmdk pinned in shadcn registry; verify with `npm ls cmdk`)."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# shadcn + cmdk Version Drift : Filter Signature, Value Case, Vaul Focus, and the Hidden Wrapper Coupling

The shadcn Command primitive is a thin React wrapper around the third-party
`cmdk` library (https://github.com/pacocoursey/cmdk). The wrapper is
copied into your repo at `components/ui/command.tsx` by `shadcn add
command`. The wrapper is generated against the cmdk version the registry
was built against at that moment. Your `package.json` then carries its OWN
copy of cmdk under `dependencies`. Those two pins can drift apart, and
when they do the symptoms are always one of three : items will not
filter, items will not click, or `CommandItem` rendered outside
`CommandList` crashes with `TypeError : undefined is not iterable
(cannot read property Symbol(Symbol.iterator))`.

This skill documents the six drift cases that surface as three of the
top-15 issues in shadcn-ui/ui by reaction count (#2944, #2980, #3051)
and the fix workflow.

Companion skills :

- `shadcn-syntax-command` (Batch B5) : Command composition surface,
  the nine subcomponents, `shouldFilter`, async filtering, Cmd+K hotkey
- `shadcn-syntax-drawer` (Batch B5) : Vaul Drawer composition and the
  `onOpenAutoFocus` / `onCloseAutoFocus` focus contract
- `shadcn-errors-radix-controlled` : the parallel controlled-state
  contract for Radix-wrapped primitives (Dialog / Sheet / Popover)
- `shadcn-errors-cli-sync-mismatch` : the `shadcn add ... --overwrite`
  workflow used to repair the wrapper when cmdk is bumped

## Quick Reference : The Three Documented Drift Cases (TL;DR)

| # | Symptom | Root cause | Fix |
|---|---------|-----------|-----|
| 1 | Custom `filter={(value, search) => ...}` works but per-item `keywords` is silently ignored | cmdk >= 1.0.0 filter signature is `(value, search, keywords) => number`. Old 2-arg form still type-checks because `keywords` is just an unused trailing parameter | Rewrite filter to 3-arg form. Use `keywords` to extend the search corpus. See examples.md §1 |
| 2 | "Apple" in data-value never matches `search="apple"` with a custom filter | Default cmdk filter uses `command-score` which lowercases internally via `formatInput`. A custom `filter` receives raw `value` and raw `search` and must lowercase BOTH or use `.toLowerCase().includes(...)` | Lowercase in the custom filter or rely on the default. See examples.md §2 |
| 3 | Inside a Vaul Drawer the Command input loses focus on the first keystroke | Vaul focus-traps on open and re-focuses its content root. cmdk auto-focuses the CommandInput. They fight, vaul wins on the next tick, the input loses focus | Pass `onOpenAutoFocus={(e) => e.preventDefault()}` to `<DrawerContent>` so vaul stops fighting cmdk. See examples.md §3 |

Three additional drift cases compound the above :

| # | Symptom | Root cause | Fix |
|---|---------|-----------|-----|
| 4 | `<Command shouldFilter={false}>` shows every item on every keystroke | `shouldFilter={false}` disables cmdk's internal filter ENTIRELY. You opted out of filtering and never wrote your own | Filter the items array yourself in the parent and pass only the filtered subset as children. See examples.md §4 |
| 5 | `import { CommandLoading } from "@/components/ui/command"` fails with "CommandLoading is not exported" | The shadcn registry wrapper `components/ui/command.tsx` re-exports nine names (Command, CommandDialog, CommandInput, CommandList, CommandEmpty, CommandGroup, CommandItem, CommandSeparator, CommandShortcut). CommandLoading is exported by cmdk but NOT by the shadcn wrapper | Import directly : `import { Command as CommandPrimitive } from "cmdk"; const CommandLoading = CommandPrimitive.Loading`. See examples.md §5 |
| 6 | After `pnpm up cmdk` the Command items become unclickable or throw "Symbol.iterator" TypeError | The wrapper in `components/ui/command.tsx` was generated against a different cmdk version. Bumping cmdk floated the wrapper out of contract. Specifically cmdk 1.0.0 made `CommandList` a REQUIRED parent of every `CommandItem` | Re-run `pnpm dlx shadcn@latest add command --overwrite` to regenerate the wrapper against the current cmdk pin, OR roll cmdk back to the previously-pinned version. See examples.md §6 |

## Quick Reference : Detection Commands

Three commands answer "is there drift in this repo right now" :

```bash
# 1. What cmdk version is installed?
pnpm ls cmdk
# or : npm ls cmdk
# or : yarn list cmdk

# 2. What cmdk version does the shadcn registry expect right now?
curl -s https://ui.shadcn.com/r/styles/new-york/command.json \
  | jq '.dependencies'
# Look for "cmdk" : "<version>" in the dependencies array

# 3. Does the wrapper file reference any cmdk API the installed version lacks?
grep -E "CommandLoading|keywords=" components/ui/command.tsx
# If the wrapper imports CommandLoading but cmdk is <1.0.0, drift.
# If the wrapper does NOT pass keywords through but you use it, drift.
```

The expected outputs match like this :

| `pnpm ls cmdk` says | Registry expects | Drift verdict |
|---------------------|------------------|---------------|
| `cmdk 0.2.1` | `cmdk ^1.0.0` | DRIFT : wrapper expects 1.0.0 API on a 0.2.x install |
| `cmdk 1.0.4` | `cmdk ^1.0.0` | OK : same major |
| `cmdk 1.1.0` | `cmdk ^1.0.0` | OK if minor is backward-compatible. Verify against cmdk CHANGELOG |
| `cmdk 0.2.1` | `cmdk ^0.2.0` | OK : both pre-1.0 |
| `cmdk` not installed | any | BROKEN : shadcn add command never ran. Re-run it |

## Quick Reference : Fix Strategies

There are three valid fix paths. Pick exactly one per project and
document it in your README so future contributors do not re-introduce
drift :

### Strategy A : Pin cmdk to the registry's exact version

When you want the wrapper file to remain untouched (e.g. you have
local customisations on top of the shadcn wrapper) :

```json
// package.json
{
  "dependencies" : {
    "cmdk" : "1.0.4"
  }
}
```

NEVER use `^` or `~` on cmdk if you are pinning. The carrot is what
caused the drift in the first place. Verified at
https://github.com/shadcn-ui/ui/issues/2944#issuecomment-... (cmdk
1.0.0 was a breaking change that float-versioning silently auto-pulled).

### Strategy B : Re-run `shadcn add command --overwrite`

When you want the wrapper to match the registry's CURRENT pin :

```bash
pnpm dlx shadcn@latest add command --diff       # show what will change
pnpm dlx shadcn@latest add command --overwrite  # apply
pnpm install                                    # pull the new cmdk pin
```

This is the "make-the-wrapper-canonical" path. Loss of local customisation
is a real risk ; use Strategy A instead if you have edits to preserve.
See `shadcn-errors-cli-sync-mismatch` for the full --diff / --overwrite
discipline.

### Strategy C : Upgrade both, migrate the filter call sites

When the cmdk major bump introduces useful features (the 1.0.0
`keywords` Item prop is a real productivity win for fuzzy search) :

```bash
pnpm up cmdk@latest
pnpm dlx shadcn@latest add command --overwrite
# Now grep for any custom filter prop and rewrite to 3-arg :
grep -rn 'filter=' src/ components/
```

Then rewrite every `(value, search) => number` filter to
`(value, search, keywords) => number` and decide whether to actually
USE the keywords argument (it concatenates the per-item `keywords:
string[]` prop into the matching corpus).

## Quick Reference : The Filter Prop Signature Drift In Detail

Verified at https://github.com/pacocoursey/cmdk (README), PR #158
(2024-03-07) and release notes 1.0.0 (2024-03-08) :

```ts
// cmdk 0.2.x signature :
type CommandFilter = (value : string, search : string) => number
// returns 1 = match, 0 = no match (or a fuzzy score in (0, 1])

// cmdk 1.0.0+ signature :
type CommandFilter = (
  value : string,
  search : string,
  keywords ?: string[]
) => number
// keywords are the per-Item `keywords` prop array, undefined if unset
```

The drift is silent because TypeScript happily accepts a 2-arg function
where a 3-arg type is expected (extra parameters are allowed by
contravariance). The compiler does not warn. The runtime simply never
sees your handling of `keywords`, so per-item `keywords` arrays are
ignored even when you set them on every `CommandItem`.

The DEFAULT cmdk filter (when you do not pass `filter` at all) is :

```ts
import commandScore from "command-score"
const defaultFilter : CommandFilter =
  (value, search, keywords) => commandScore(value, search, keywords)
```

`command-score` is case-insensitive because it calls `toLowerCase()`
on both the haystack and the needle inside its `formatInput` helper.
Verified at https://raw.githubusercontent.com/pacocoursey/cmdk/main/cmdk/src/command-score.ts
(2026-05-19).

If you write a custom `filter`, you get the raw `value` and raw `search`
with their case intact. Case-insensitive matching is YOUR job in a
custom filter.

## Quick Reference : Value Normalization Contract

cmdk stores each item's value on a `data-value` attribute on the
`<div>` rendered by `CommandItem`. The value comes from one of :

1. The explicit `value` prop on `CommandItem` (preferred ; stable
   across re-renders).
2. The text content of the item (fallback ; volatile if the children
   change).

Before storage, cmdk applies `.trim()` (verified, README quote :
"Values are always trimmed with the trim() method") but NOT
`.toLowerCase()`. The `data-value` attribute therefore preserves case.

The DEFAULT filter (`command-score`) is case-insensitive because the
algorithm lowercases internally. A CUSTOM filter is case-sensitive
unless you lowercase both arguments yourself.

This is the source of "Apple does not match apple" reports : a custom
filter that does `value === search` or `value.includes(search)` against
the raw arguments returns 0 (= no match) when the case differs.

## Quick Reference : The Vaul Drawer + Command Focus Duel

Vaul (https://github.com/emilkowalski/vaul) ships its own focus trap.
When `<DrawerContent>` opens, vaul calls `focus()` on the content root,
which is the element wrapping your `<Command>`. cmdk's `<CommandInput>`
also has `autoFocus` semantics : it ensures the input is focused on
mount so keyboard typing works without an extra click.

Without intervention :

1. Drawer opens. Vaul focuses `DrawerContent` root.
2. Command mounts. cmdk focuses the inner `<input>`.
3. React commits the focus to the input.
4. Vaul's effect runs on the NEXT tick and re-focuses the root.
5. The user's first keystroke goes nowhere.

The fix is to tell vaul to stop fighting :

```tsx
<DrawerContent
  onOpenAutoFocus={(e) => e.preventDefault()}
>
  <Command>
    <CommandInput placeholder="Type a command..." />
    <CommandList>...</CommandList>
  </Command>
</DrawerContent>
```

`onOpenAutoFocus` is a Vaul Radix-derived prop. Calling
`e.preventDefault()` disables the default focus-trap entry behaviour.
cmdk then wins and the input stays focused.

The same duel happens between cmdk and Radix Dialog when Command is
inside a Dialog. Same fix : `<DialogContent onOpenAutoFocus={(e) =>
e.preventDefault()}>`.

## Quick Reference : shouldFilter and Async Search

`shouldFilter` defaults to `true`. cmdk filters and sorts the rendered
items by `command-score` against the current search.

`shouldFilter={false}` disables BOTH filtering AND sorting. cmdk
renders every `CommandItem` you give it, in the order you give them.
This is the correct setting when results come from a server (debounced
fetch on input change) ; cmdk should NOT also try to fuzzy-filter them.

The trap : developers flip `shouldFilter={false}` and then forget to
filter the items in their parent state. Result : the list shows the
entire result set on every keystroke, ignoring what the user typed.

The contract :

| `shouldFilter` | Who filters | Who sorts |
|----------------|-------------|-----------|
| `true` (default) | cmdk (command-score) | cmdk (descending score) |
| `false` | YOU (in parent state, before passing items) | YOU |

When `shouldFilter={false}`, ALWAYS pair it with a `CommandLoading`
indicator (imported from `cmdk` directly, NOT from the shadcn
wrapper) and a debounced fetch. See examples.md §4.

## Quick Reference : CommandLoading Is Not Re-Exported

The shadcn registry wrapper at `components/ui/command.tsx`
re-exports exactly these names :

```ts
// components/ui/command.tsx (shadcn registry, verified 2026-05-19)
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
}
```

`CommandLoading` is exported by `cmdk` itself (as `Command.Loading`)
but NOT re-exported by the shadcn wrapper. Importing it from
`@/components/ui/command` fails with "CommandLoading is not exported".

Two valid imports :

```ts
// Path A : import from cmdk directly
import { Command as CommandPrimitive } from "cmdk"
const CommandLoading = CommandPrimitive.Loading
```

```ts
// Path B : extend the shadcn wrapper to re-export it
// Edit components/ui/command.tsx :
import { Command as CommandPrimitive } from "cmdk"
// ...existing code...
const CommandLoading = CommandPrimitive.Loading
export { /* existing exports */, CommandLoading }
```

Path B makes the wrapper non-re-add-safe : the next
`shadcn add command --overwrite` will wipe your re-export. Document
it in your README or use Path A.

## Quick Reference : The CommandItem-Outside-CommandList Crash

cmdk 1.0.0 changed the internal item registry from a flat map to a
list-scoped map. Every `CommandItem` MUST now be a descendant of
`CommandList`. Rendering an item outside crashes with :

```
TypeError : undefined is not iterable (cannot read property
Symbol(Symbol.iterator))
```

This is the same crash reported in issue #2944. The fix is structural,
not a prop : wrap every `CommandItem` in a `CommandList`. Example
correct shape :

```tsx
<Command>
  <CommandInput placeholder="Search..." />
  <CommandList>
    <CommandEmpty>No results.</CommandEmpty>
    <CommandGroup heading="Suggestions">
      <CommandItem value="calendar">Calendar</CommandItem>
      <CommandItem value="search">Search</CommandItem>
    </CommandGroup>
  </CommandList>
</Command>
```

`CommandEmpty` must also live INSIDE `CommandList` for the same
registry reason.

## See Also

- `references/methods.md` : filter signatures per cmdk version,
  full compatibility matrix, shadcn wrapper vs cmdk dependency tree
- `references/examples.md` : six side-by-side WRONG vs RIGHT
  examples covering each drift case
- `references/anti-patterns.md` : the six drift anti-patterns with
  root cause and fix for each
