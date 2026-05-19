# shadcn ui Selectors : Anti-Patterns

Seven anti-patterns with the canonical failure mode, the root cause, and the fix. Each is observed in real shadcn-ui/ui GitHub issues, the Radix Select / Popover docs, or the cmdk error catalogue.

## Anti-Pattern 1 : binding shadcn Select with `register("field")` instead of Controller

### Bad

```tsx
import { Select, SelectTrigger, SelectValue, SelectContent, SelectItem } from "@/components/ui/select"

const { register } = useForm()

<Select {...register("framework")}>
  <SelectTrigger><SelectValue placeholder="Pick" /></SelectTrigger>
  <SelectContent>
    <SelectItem value="next">Next.js</SelectItem>
  </SelectContent>
</Select>
```

### Why it breaks

Radix Select is a controlled component owned by `onValueChange`. `register("framework")` returns `{ name, ref, onChange, onBlur }` and tries to spread `onChange` onto `<Select>` ; but Radix Select does NOT call `onChange` (it calls `onValueChange`). The form silently captures the initial empty string forever, even though the visual trigger updates correctly. Submit produces `{ framework: undefined }` or the default value.

This is issue #shadcn-ui/ui repeated across the form-Select integration questions ; vooronderzoek §10 documents 117+72+42 reactions on cmdk-related Select/Combobox breakages where the same pattern surfaces.

### Fix

ALWAYS use `Controller` :

```tsx
<Controller
  control={control}
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

`field.onChange` is wired to `onValueChange` directly. The form captures the real selected value.

## Anti-Pattern 2 : Combobox Popover without `modal={true}`

### Bad

```tsx
<Popover open={open} onOpenChange={setOpen}>
  <PopoverTrigger asChild>...</PopoverTrigger>
  <PopoverContent>
    <Command>
      <CommandInput />
      <CommandList>
        <CommandItem onSelect={...}>...</CommandItem>
      </CommandList>
    </Command>
  </PopoverContent>
</Popover>
```

### Why it breaks

Popover defaults to `modal={false}`. When the user clicks a `<CommandItem>` :

1. Click fires on the item.
2. Focus moves OUT of the popover (because non-modal Popover does not trap focus).
3. The `onPointerDownOutside` handler fires from the click landing outside the focus-trapped region.
4. Race condition : `setOpen(false)` and `onSelect(value)` fire in unpredictable order. The popover may stay open, items may not register the click, and keyboard navigation breaks.

Symptom : "Combobox items not clickable" (issue #2944, 117 reactions, listed in vooronderzoek §10 as one of the top-15 cmdk pain points).

### Fix

ALWAYS pass `modal` on the Popover root for Combobox recipes :

```tsx
<Popover open={open} onOpenChange={setOpen} modal>
  ...
</Popover>
```

Focus is trapped inside the popover ; clicks on items fire predictably ; `onSelect` runs synchronously before the popover closes.

## Anti-Pattern 3 : Native `<select>` with 30+ options on mobile

### Bad

```tsx
<select {...register("country")}>
  {countries.map(c => <option key={c.code} value={c.code}>{c.name}</option>))}
</select>
```

Where `countries.length === 250`.

### Why it breaks

iOS renders a `<select>` with 250 options as a full-screen wheel picker. Scrolling through 250 items is a scroll trap : no search, no jump, the user must wheel through all entries. Android renders a bottom sheet with the same issue. Desktop rendering shows a 250-row dropdown that overflows the viewport with no scroll-up/scroll-down affordances on some browsers.

A "country picker" with 250 options is the canonical anti-example.

### Fix

Reach for Combobox (Recipe 3 or 4). The Combobox provides :

- A search input that filters as the user types.
- Keyboard navigation with `aria-activedescendant`.
- A scrollable list with consistent height.
- An empty state when the filter matches nothing.

ALWAYS use a Combobox for lists with > ~30 items. NEVER use native `<select>` for country, language, framework, or tag pickers with large catalogues.

## Anti-Pattern 4 : Command palette opened via plain `<Command>` instead of `<CommandDialog>`

### Bad

```tsx
const [open, setOpen] = React.useState(false)

return (
  <>
    <Button onClick={() => setOpen(true)}>Open</Button>
    {open && (
      <div className="fixed inset-0 z-50 bg-black/50">
        <div className="absolute left-1/2 top-1/4 w-[400px] -translate-x-1/2 rounded-lg bg-popover p-4">
          <Command>
            <CommandInput placeholder="Search..." />
            <CommandList>
              <CommandItem onSelect={...}>...</CommandItem>
            </CommandList>
          </Command>
        </div>
      </div>
    )}
  </>
)
```

### Why it breaks

The hand-rolled overlay :

- Is NOT portalled. If an ancestor has `transform`, `filter`, or `will-change`, the palette renders inside that stacking context and may be clipped or stacked incorrectly.
- Does NOT trap focus. Tab moves out of the palette into the page underneath.
- Has NO accessible name. Screen readers announce "dialog" with no further context. Radix would log a critical a11y warning ; here the warning is missing because Radix is not in the loop.
- Cannot be closed by `Esc` automatically.
- Does NOT release `aria-hidden` on the page underneath when closed.

axe-core flags this as critical (`aria-dialog-name` rule, severity critical).

### Fix

Use `<CommandDialog>` :

```tsx
<CommandDialog open={open} onOpenChange={setOpen} title="Command Palette">
  <CommandInput placeholder="Search..." />
  <CommandList>...</CommandList>
</CommandDialog>
```

Portal, overlay, focus trap, sr-only DialogTitle from the `title` prop, `Esc`-to-close, all wired automatically.

## Anti-Pattern 5 : async Combobox shows `<CommandEmpty>` while request is in-flight

### Bad

```tsx
const { data } = useQuery({ queryKey: ["users", q], queryFn: () => searchUsers(q) })

<CommandList>
  {data?.length === 0 && <CommandEmpty>No users found.</CommandEmpty>}
  {data?.map(u => <CommandItem ...>...</CommandItem>)}
</CommandList>
```

### Why it breaks

While the request is loading, `data` is `undefined`, so `data?.length === 0` is `false` and `<CommandEmpty>` doesn't render. Good so far. But on the FIRST successful response with zero matches, `data === []` and `data.length === 0` is `true`, so `<CommandEmpty>` paints. So far still correct.

The failure mode : on a second query that triggers a refetch, React Query keeps `data` as the PREVIOUS result set during the in-flight refetch (default `keepPreviousData` behaviour, or just stale-while-revalidate). If the previous result was `[]`, `<CommandEmpty>` paints "No users found" during a query the user just typed, even though the server may return matches in 200ms. User reads "No results", abandons the input, never sees the actual matches.

### Fix

ALWAYS gate the three states explicitly on `isLoading` AND `isError` AND `data.length` :

```tsx
{isLoading && <CommandLoading>Searching...</CommandLoading>}
{isError && <div role="alert">Failed to load</div>}
{!isLoading && !isError && data && data.length === 0 && (
  <CommandEmpty>No users found.</CommandEmpty>
)}
{!isLoading && !isError && data && data.length > 0 && (
  <CommandGroup>{data.map(...)}</CommandGroup>
)}
```

The four states (typing-too-short, loading, error, empty, results) are mutually exclusive ; show exactly one. NEVER let `<CommandEmpty>` paint while `isLoading` is true.

## Anti-Pattern 6 : SelectTrigger without SelectValue

### Bad

```tsx
<Select value={value} onValueChange={setValue}>
  <SelectTrigger>Pick a framework</SelectTrigger>
  <SelectContent>
    <SelectItem value="next">Next.js</SelectItem>
  </SelectContent>
</Select>
```

### Why it breaks

`<SelectTrigger>` renders whatever children it receives, plus the `ChevronDownIcon`. It does NOT inspect `value` and does NOT render the selected label by default. Without a `<SelectValue>` slot inside the trigger, the trigger ALWAYS shows the literal string "Pick a framework" ; the selected label never paints. The user picks "Next.js", the dropdown closes, the trigger still reads "Pick a framework".

Symptom : "Select shows placeholder after pick" (Stack Overflow recurring question, plus shadcn-ui/ui Q&A discussion threads).

### Fix

ALWAYS render `<SelectValue placeholder="...">` inside `<SelectTrigger>` :

```tsx
<SelectTrigger>
  <SelectValue placeholder="Pick a framework" />
</SelectTrigger>
```

`<SelectValue>` is Radix's binding to the root `value` ; it renders the selected item's label, falling back to the `placeholder` prop when no value is set. NEVER omit it.

## Anti-Pattern 7 : empty-string `value` on `<SelectItem>`

### Bad

```tsx
<SelectContent>
  <SelectItem value="">Any framework</SelectItem>
  <SelectItem value="next">Next.js</SelectItem>
</SelectContent>
```

### Why it breaks

Radix Select treats empty string as "no value selected" internally. Passing `value=""` on a SelectItem creates a value collision : the item exists, but its selection is indistinguishable from "no selection". Radix throws a runtime error in development : `A <Select.Item /> must have a value prop that is not an empty string.`

### Fix

ALWAYS use a non-empty sentinel for an "Any" / "All" / "None" option :

```tsx
<SelectContent>
  <SelectItem value="__any">Any framework</SelectItem>
  <SelectItem value="next">Next.js</SelectItem>
</SelectContent>
```

Then map the sentinel back to "no filter" in your consumer code (`value === "__any" ? undefined : value`). NEVER use empty string.

## Verification

All anti-patterns are observable in :

- shadcn-ui/ui issues #2944 (117 reactions), #2980 (87), #3051 (42), #9393 (39) for cmdk + Combobox failures.
- Radix Select primitive docs (`radix-ui.com/primitives/docs/components/select`) for the empty-string SelectItem error.
- W3C WAI-ARIA Dialog pattern for the missing-DialogTitle a11y critical failure.
- React Hook Form Controller docs for the controlled-input contract.
- Vooronderzoek §2 (Combobox/Select/Command), §3.10 (Combobox/Command cmdk break), §3.17 (Form Controller vs register confusion).
