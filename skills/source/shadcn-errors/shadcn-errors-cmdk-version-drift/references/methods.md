# cmdk Version Drift : Methods Reference

This document captures the verbatim API contracts you need when
diagnosing cmdk drift in a shadcn project. Sources are
https://github.com/pacocoursey/cmdk (README, source, releases),
https://github.com/shadcn-ui/ui/blob/main/apps/v4/registry/new-york-v4/ui/command.tsx
(shadcn wrapper), and https://github.com/shadcn-ui/ui/issues/2944
(the canonical drift incident report). All quotes verified
2026-05-19.

## 1. The Filter Prop Signature Across cmdk Versions

The shape of the `filter` prop on `<Command>` (Command.Root) changed
exactly once, between cmdk 0.2.x and cmdk 1.0.0. There is no
intermediate signature.

### cmdk 0.2.x (released 2022 to early 2024)

```ts
type CommandFilter = (value : string, search : string) => number
```

- `value` : the trimmed value of the CommandItem currently being
  considered. Sourced from the `value` prop on `<CommandItem>` or,
  if absent, from `String(children)`.
- `search` : the current text inside `<CommandInput>`.
- Return : a number. Convention is `1` for match, `0` for no match,
  or a fuzzy score in the range `(0, 1]`. cmdk sorts the rendered
  items by descending return value.

There is no `keywords` argument. Per-item `keywords` boosting did
not exist.

### cmdk 1.0.0 (released 2024-03-08, PR #158)

```ts
type CommandFilter = (
  value : string,
  search : string,
  keywords ?: string[]
) => number
```

- `value` and `search` unchanged.
- `keywords` : the value of the `keywords : string[]` prop on
  the `<CommandItem>` currently being considered. `undefined` if
  the item did not declare a `keywords` prop.

PR #158 ("Add keywords prop to Item") introduced this in a single
breaking-but-back-compat-adjacent commit. The signature change is
backward-compatible in TypeScript (a 2-arg function is structurally
assignable to a 3-arg type because extra parameters are allowed by
contravariance) but semantically breaking : 2-arg filter functions
silently ignore the keywords corpus.

### How to read your installed version

```bash
pnpm ls cmdk           # pnpm
npm ls cmdk            # npm
yarn list cmdk         # yarn
```

Output examples :

```
$ pnpm ls cmdk
my-app@0.1.0
└── cmdk 1.0.4
```

```
$ pnpm ls cmdk
my-app@0.1.0
└── cmdk 0.2.1
```

If you see `0.2.x`, you are pre-1.0.0. The 3-arg filter is wrong
for that install ; the keywords array will be `undefined` at
runtime even if you wrote a 3-arg signature.

## 2. The shouldFilter Prop

```ts
type CommandProps = {
  // ...
  shouldFilter ?: boolean  // default true
  // ...
}
```

Behaviour table :

| `shouldFilter` | Effect |
|----------------|--------|
| `true` (default) | cmdk applies `filter` (custom or default) to every CommandItem, hides items returning 0, sorts the remaining items by descending return value |
| `false` | cmdk applies NO filter. Every CommandItem renders in source order. `filter` is ignored. `CommandEmpty` STILL renders if zero items are children |

There is no "filter but do not sort" mode. If you need stable order,
either pre-sort the items in your parent and pass `shouldFilter={false}`
(doing your own filter in the parent too), or accept cmdk's
score-descending order.

## 3. Value Normalization Inside cmdk

cmdk applies exactly one normalisation to a CommandItem's value before
storing it on the `data-value` attribute of the rendered `<div>` :

```
String.prototype.trim()
```

Source : the cmdk README, verbatim quote :

> "Values are always trimmed with the trim() method."

`toLowerCase()` is NOT applied by cmdk on the value side. Whatever
case you wrote, that is what ends up on `data-value`.

### The default filter (command-score) IS case-insensitive

cmdk's default filter (when no `filter` prop is passed) is :

```ts
import commandScore from "command-score"
const defaultFilter : CommandFilter =
  (value, search, keywords) => commandScore(value, search, keywords)
```

`command-score` lowercases both arguments inside its `formatInput`
helper :

```ts
// from command-score.ts, lines verified 2026-05-19
function formatInput(string : string) : string {
  return string.toLowerCase().replace(COUNT_SPACE_REGEXP, ' ')
}
```

Therefore : the DEFAULT cmdk behaviour is case-insensitive fuzzy match.
A CUSTOM filter receives the raw, unmodified `value` and `search`.

### Implication for custom filters

A custom filter that does :

```ts
filter={(value, search) => value === search ? 1 : 0}
```

will return 0 for `value="Apple"`, `search="apple"`. The user typed
all lowercase, the data is title case, no match. The user reports
"the filter is broken" but the input is in fact processed correctly.

The right shape for a custom case-insensitive filter :

```ts
filter={(value, search, keywords) => {
  const v = value.toLowerCase()
  const s = search.toLowerCase()
  const k = (keywords ?? []).join(' ').toLowerCase()
  return (v + ' ' + k).includes(s) ? 1 : 0
}}
```

## 4. The shadcn Wrapper Export Surface

The shadcn registry generates `components/ui/command.tsx` with this
export block (verified 2026-05-19 against
https://ui.shadcn.com/r/styles/new-york/command.json) :

```ts
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

That is nine names. `CommandLoading` is NOT among them.

cmdk itself exports (verified against
https://raw.githubusercontent.com/pacocoursey/cmdk/main/cmdk/src/index.tsx) :

```ts
const Command = ...
Command.List = List
Command.Item = Item
Command.Group = Group
Command.Input = Input
Command.Separator = Separator
Command.Dialog = Dialog
Command.Empty = Empty
Command.Loading = Loading      // <-- present in cmdk
export { Command, ... }
```

The shadcn wrapper picks up Command, List, Item, Group, Input,
Separator, Dialog, Empty but NOT Loading. The Loading subcomponent
must be imported from cmdk directly. See SKILL.md §"CommandLoading
Is Not Re-Exported" and examples.md §5.

## 5. The CommandList Parent Requirement (cmdk 1.0.0+)

cmdk 1.0.0 introduced a list-scoped item registry. Every
`<CommandItem>` MUST be a descendant of a `<CommandList>`. Same for
`<CommandEmpty>`, `<CommandGroup>`, and `<CommandSeparator>`.

Rendering an item outside the list crashes with :

```
TypeError : undefined is not iterable (cannot read property
Symbol(Symbol.iterator))
```

This trace appears in `cmdk/index.tsx` inside the internal `useCmdk()`
hook when it iterates the list registry that does not exist. Reported
in issue #2944 (117 reactions) as the canonical symptom of the cmdk
0.2.x to 1.0.0 break.

The contract is structural :

```
<Command>
  <CommandInput />
  <CommandList>
    <CommandEmpty />?
    <CommandGroup>?
      <CommandItem />*
      <CommandSeparator />?
    </CommandGroup>*
    <CommandItem />*           ← items outside a group are fine
                                 as long as they are inside the list
  </CommandList>
</Command>
```

`<CommandInput>` is the ONLY child that may live outside
`<CommandList>` (and must, in fact, be a sibling of it because cmdk
wires keyboard navigation between them at the Command root level).

## 6. The shadcn Wrapper vs cmdk Version Dependency Tree

The dependency relationship :

```
your-project/package.json
  └── dependencies
        └── cmdk : <some-version>           ← (A) the runtime install

your-project/components/ui/command.tsx       ← (B) the wrapper file
  └── import { Command as CommandPrimitive } from "cmdk"
  └── re-exports based on cmdk API surface AT THE TIME shadcn registry
      generated this file
```

Drift happens when (A) and (B) reference different cmdk API surfaces.
Three concrete drift scenarios :

| (B) wrapper was generated for | (A) installed cmdk | Symptom |
|------------------------------|--------------------|---------|
| cmdk 1.0.x | cmdk 0.2.x | "CommandLoading is not exported" if wrapper extended ; or no per-item keywords boosting ; or Symbol.iterator crash if cmdk wrapper code paths assume list-scoped registry |
| cmdk 0.2.x | cmdk 1.0.x | filter prop wrapper-passes only 2 args even though cmdk passes 3 ; per-item keywords ignored ; CommandItem outside CommandList crash if wrapper renders the old flat-list shape |
| cmdk 1.0.x | cmdk 1.1.x (minor bump) | Usually fine, but verify the cmdk CHANGELOG for any item-attribute renames |

The repair workflow is in SKILL.md §"Quick Reference : Fix Strategies"
(Strategies A, B, C).

## 7. The Vaul Drawer Focus API

Vaul re-uses Radix's focus prop names via its Drawer wrapping a
Radix Dialog. The relevant props on `<DrawerContent>` :

```ts
type DrawerContentProps = {
  onOpenAutoFocus ?: (event : Event) => void
  onCloseAutoFocus ?: (event : Event) => void
  // ...
}
```

- `onOpenAutoFocus` : fires immediately after the Drawer opens, on the
  element that would receive default focus (the content root). Calling
  `event.preventDefault()` cancels the default focus assignment.
- `onCloseAutoFocus` : fires when the Drawer closes, on the trigger
  element that would receive returned focus. Calling
  `event.preventDefault()` keeps focus where it was.

For a Drawer wrapping a Command, ALWAYS preventDefault on
`onOpenAutoFocus` so cmdk's CommandInput focus wins. See examples.md
§3.

The same prop pair exists on Radix's `DialogContent`, `SheetContent`,
`PopoverContent`, `DropdownMenuContent`, `ContextMenuContent`. The
Command-inside-Radix-overlay duel is identical for each. Apply
`onOpenAutoFocus={(e) => e.preventDefault()}` to every overlay that
contains a Command primitive.

## 8. Default Filter Score Interpretation

`command-score` returns a number in `[0, 1]` :

| Range | Meaning |
|-------|---------|
| `0` | No match. Item is hidden. |
| `(0, 0.5)` | Weak match (lots of skipped characters). Item is shown, sorted low. |
| `[0.5, 0.9)` | Reasonable match. |
| `[0.9, 1)` | Strong match (word boundaries, prefix matches). |
| `1` | Exact equality after lowercase + trim. |

If you implement a custom filter that returns boolean-style 0 and 1
only, you lose the sorted-by-relevance behaviour cmdk gives you for
free. Return a graded score whenever the matching logic admits one
(e.g. `value.startsWith(search) ? 1 : value.includes(search) ? 0.5 : 0`).
