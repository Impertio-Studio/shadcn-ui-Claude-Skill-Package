# Anti-Patterns : Radix Controlled-State Traps

Eight recurring traps. Each entry : SYMPTOM (what the user sees) +
CAUSE (root, not surface) + FIX (the correct pattern). Verified
against shadcn ui issues, Radix UI issues, and the Radix docs,
2026-05-19.

## AP-1 : open Without onOpenChange (Read-Only Half-Controlled State)

### Symptom

The dialog opens via the trigger, but the X button, Escape key, and
overlay click all do nothing. Console shows no error. Pressing the
DialogClose element does nothing either.

### WRONG

```tsx
const isOpen = Boolean(selectedUser)
<Dialog open={isOpen}>
  <DialogContent>
    <DialogTitle>Edit</DialogTitle>
    ...
  </DialogContent>
</Dialog>
```

### Why It Breaks

Radix's Dialog runs in controlled mode the instant `open` is a
non-undefined boolean. Internal close handlers (Escape,
PointerDownOutside, DialogClose) call `onOpenChange(false)` instead
of mutating internal state. With `onOpenChange` missing, those
callbacks fire into a void. The `open` prop stays `true` until the
parent's `selectedUser` resets (which it usually never does, because
the dialog was the thing supposed to clear it).

### CORRECT

```tsx
const [open, setOpen] = React.useState(false)

React.useEffect(() => {
  if (selectedUser) setOpen(true)
}, [selectedUser])

<Dialog
  open={open}
  onOpenChange={(next) => {
    setOpen(next)
    if (!next) setSelectedUser(undefined)
  }}
>
  ...
</Dialog>
```

OR drop the controlled state entirely if the parent does not need
to react :

```tsx
<Dialog defaultOpen={Boolean(selectedUser)}>...</Dialog>
```

Applies to : Dialog, AlertDialog, Sheet, Drawer, Popover,
DropdownMenu, ContextMenu, Menubar, Collapsible, HoverCard, Tooltip.
The same trap exists with `value` for Select, Tabs, Accordion,
RadioGroup, ToggleGroup (a `value` without `onValueChange` is
equivalently read-only).

## AP-2 : asChild With Multiple Children (Slot Crash)

### Symptom

Page crashes on render with the error :

> Error: React.Children.only expected to receive a single React element child.

The component does not render at all ; the entire React tree below
unmounts.

### WRONG

```tsx
<DialogTrigger asChild>
  <Link href="/edit"><Icon /></Link>
  <span className="sr-only">Edit</span>
</DialogTrigger>

// OR equivalent with array-children :
<DialogTrigger asChild>
  {[<Icon key="i" />, <span key="t">Edit</span>]}
</DialogTrigger>

// OR a fragment :
<DialogTrigger asChild>
  <>
    <Icon />
    <span>Edit</span>
  </>
</DialogTrigger>
```

### Why It Breaks

The Slot utility (`@radix-ui/react-slot`) calls
`React.Children.only(children)` to enforce that there is exactly one
React element to merge props onto. Zero children, two siblings, or
a fragment-of-multiple all fail. A fragment containing exactly one
element technically also fails because `React.Children.only` does
not unwrap fragments.

### CORRECT

Wrap the multiple inner elements inside the single asChild target :

```tsx
<DialogTrigger asChild>
  <Link href="/edit">
    <Icon />
    <span className="sr-only">Edit</span>
  </Link>
</DialogTrigger>
```

If the design genuinely needs two top-level elements, drop `asChild`
and accept the default `<button>` wrapper :

```tsx
<DialogTrigger>
  <Icon />
  <span>Edit</span>
</DialogTrigger>
```

The trigger then renders a `<button>` containing both children.

## AP-3 : asChild With A Non-forwardRef Child (Silent Focus Loss)

### Symptom

The trigger LOOKS right. The dialog opens. But on close, focus does
not return to the trigger element ; instead it jumps to the document
body. Screen readers announce "blank" after the dialog closes.

### WRONG

```tsx
// A custom button component that does NOT forward refs.
function FancyButton({ children, ...props }: React.ComponentProps<"button">) {
  return <button {...props}>{children}</button>
}

<DialogTrigger asChild>
  <FancyButton>Edit</FancyButton>
</DialogTrigger>
```

### Why It Breaks

Slot composes refs via `composeRefs(forwardedRef, child.ref)`. A
component that does not call `React.forwardRef` drops the ref ;
Radix's focus-restoration code holds a null ref to the trigger and
falls back to document.body on close. The visual click flow still
works because `onClick` is forwarded as a prop, not via the ref.

### CORRECT

```tsx
const FancyButton = React.forwardRef<HTMLButtonElement, React.ComponentProps<"button">>(
  function FancyButton({ children, ...props }, ref) {
    return <button ref={ref} {...props}>{children}</button>
  }
)

<DialogTrigger asChild>
  <FancyButton>Edit</FancyButton>
</DialogTrigger>
```

OR use a native element directly. shadcn's `<Button>` already
forwards refs, so :

```tsx
<DialogTrigger asChild>
  <Button>Edit</Button>
</DialogTrigger>
```

is always safe.

## AP-4 : asChild With <input> (Wrong Semantics + Prop Mismatch)

### Symptom

The dropdown trigger is an `<input>`. The dropdown opens but screen
readers do not announce it as a menu trigger. Keyboard navigation
(ArrowDown to first item) does not work. `aria-haspopup` is set but
on an `<input>`, which is semantically meaningless.

### WRONG

```tsx
<DropdownMenuTrigger asChild>
  <input
    type="text"
    value={query}
    onChange={(e) => setQuery(e.target.value)}
    placeholder="Search..."
  />
</DropdownMenuTrigger>
```

### Why It Breaks

`asChild` merges `aria-haspopup="menu"`, `aria-expanded`,
`data-state`, and `onClick` onto the child. An `<input>` accepts
these as DOM attributes but they have no a11y meaning on inputs.
Worse, `onClick` on an input clashes with text input behavior :
clicking the input to position the caret ALSO toggles the menu.

### CORRECT

Use a Combobox pattern (a button trigger that reveals a popover
containing the input + list) :

```tsx
<Popover open={open} onOpenChange={setOpen}>
  <PopoverTrigger asChild>
    <Button variant="outline" role="combobox" aria-expanded={open}>
      {value ?? "Pick one"}
    </Button>
  </PopoverTrigger>
  <PopoverContent className="p-0">
    <Command>
      <CommandInput value={query} onValueChange={setQuery} />
      <CommandList>
        <CommandEmpty>No results.</CommandEmpty>
        {items.map((item) => (
          <CommandItem key={item} onSelect={() => onPick(item)}>
            {item}
          </CommandItem>
        ))}
      </CommandList>
    </Command>
  </PopoverContent>
</Popover>
```

This pattern is the official shadcn Combobox recipe. See
`shadcn-syntax-command` for the full Command primitive details.

## AP-5 : Portal Default In A Transformed Parent (Stacking-Context Trap)

### Symptom

The dialog opens but is hidden BEHIND the page header, a sticky
sidebar, or a modal-like backdrop from another library. Increasing
the z-index on the dialog's overlay (already at z-50 in shadcn's
default) does nothing.

### WRONG (parent CSS)

```tsx
<div style={{ transform: "translateZ(0)" /* for GPU rasterization */ }}>
  <Dialog>
    <DialogTrigger>Open</DialogTrigger>
    <DialogContent>...</DialogContent>
  </Dialog>
</div>
```

### Why It Breaks

Per the CSS Containing Block spec, ANY non-`none` `transform`,
`filter`, `perspective`, or `will-change` on an ancestor creates a
new containing block for `position: fixed` descendants AND a new
stacking context. Radix Portal lands at `document.body` (bypassing
the local DOM tree) but the visual stacking is governed by the
nearest stacking-context ancestor of where the portal is mounted.

Even though the portal physically lives at body level, its z-index
competes with the OUTER stacking context, which may include a
later-in-DOM-order sibling that wins.

### CORRECT (three options)

1. Remove the offending transform on the ancestor :

```tsx
<div style={{ /* no transform */ }}>...</div>
```

2. Move the transform to a deeper descendant that does not contain
   the Dialog :

```tsx
<div>
  <Dialog>...</Dialog>
  <div style={{ transform: "translateZ(0)" }}>{otherStuff}</div>
</div>
```

3. Scope the portal to a div you control whose stacking context you
   know :

```tsx
const portalHostRef = React.useRef<HTMLDivElement>(null)
return (
  <>
    <div ref={portalHostRef} className="fixed inset-0 z-[200] pointer-events-none" />
    <div style={{ transform: "translateZ(0)" }}>
      <Dialog>
        <DialogTrigger>Open</DialogTrigger>
        <DialogPortal container={portalHostRef.current}>
          <DialogOverlay />
          <DialogContent>...</DialogContent>
        </DialogPortal>
      </Dialog>
    </div>
  </>
)
```

The portal host is OUTSIDE the transformed ancestor, so its
stacking context is the root stacking context. The dialog stacks on
top.

This is verified via shadcn-ui/ui issue #3552 and Radix issue #2233
discussions on transformed parents.

## AP-6 : modal=true Popover With Form Submit (Focus Loss / Click Eaten)

### Symptom

The Popover contains a small form. The user types, presses Tab, and
the popover instantly closes. OR : the user clicks the Submit
button but the click is "eaten" because the popover closed first
and the click landed on a now-different element underneath.

### WRONG

```tsx
// Default Popover : modal={false}
<Popover open={open} onOpenChange={setOpen}>
  <PopoverTrigger asChild><Button>Filter</Button></PopoverTrigger>
  <PopoverContent>
    <form onSubmit={(e) => { e.preventDefault(); applyFilter() }}>
      <Input autoFocus />
      <Button type="submit">Apply</Button>
    </form>
  </PopoverContent>
</Popover>
```

### Why It Breaks

A non-modal Popover does NOT trap focus. The instant focus crosses
the popover boundary (Tab to a sibling DOM element, click on the
overlay-less area outside the popper), `onPointerDownOutside` or
`onFocusOutside` fires and closes the popover. The submit click can
land on a different element after the popover unmounts.

### CORRECT

```tsx
<Popover open={open} onOpenChange={setOpen} modal={true}>
  <PopoverTrigger asChild><Button>Filter</Button></PopoverTrigger>
  <PopoverContent>
    <form
      onSubmit={(e) => {
        e.preventDefault()
        applyFilter()
        setOpen(false)              // close AFTER apply, not before
      }}
    >
      <Input autoFocus />
      <Button type="submit">Apply</Button>
    </form>
  </PopoverContent>
</Popover>
```

With `modal={true}` :

- Focus traps inside the PopoverContent.
- Outside clicks still close via `onPointerDownOutside`, but the
  body is inert so the click cannot also activate an outside
  element.
- The submit handler runs to completion before `setOpen(false)` runs.

If the action is async, close in `onSuccess` of the mutation, NOT in
the synchronous submit branch. See `references/examples.md` Example 2.

## AP-7 : Escape Swallowed By A Custom Keydown Handler

### Symptom

Pressing Escape does NOT close the dialog. Other close paths (X
button, overlay click) work. The dialog appears to ignore Escape
inconsistently : it works on first open, fails on second.

### WRONG

```tsx
<DialogContent
  onKeyDown={(e) => {
    if (e.key === "Escape") {
      // Developer thought they were adding extra Escape behavior.
      // They actually stopped Radix's internal handler from firing.
      e.stopPropagation()
    }
  }}
>
  ...
</DialogContent>
```

OR a parent component that captures Escape :

```tsx
React.useEffect(() => {
  const handler = (e: KeyboardEvent) => {
    if (e.key === "Escape") {
      e.preventDefault()    // global escape eater
      e.stopPropagation()
    }
  }
  window.addEventListener("keydown", handler, true)  // CAPTURE phase
  return () => window.removeEventListener("keydown", handler, true)
}, [])
```

### Why It Breaks

Radix listens for Escape via a document-level keydown handler. A
capture-phase listener on `window` can intercept the event before
Radix sees it. `stopPropagation` on a content keydown handler
prevents the event from bubbling to Radix's listener.

### CORRECT

Use Radix's intended hook : `onEscapeKeyDown` on `DialogContent`.

```tsx
<DialogContent
  onEscapeKeyDown={(e) => {
    if (hasUnsavedChanges) {
      e.preventDefault()    // cancel close
      toast("Save or discard first.")
    }
    // Otherwise, the default close behavior fires.
  }}
>
  ...
</DialogContent>
```

For a global Escape shortcut (e.g., open a command palette), use
the bubble phase NOT capture, and check `e.defaultPrevented`
before firing :

```tsx
React.useEffect(() => {
  const handler = (e: KeyboardEvent) => {
    if (e.defaultPrevented) return    // Radix or another component handled it
    if (e.key === "Escape") {
      // global behavior
    }
  }
  window.addEventListener("keydown", handler)   // BUBBLE phase
  return () => window.removeEventListener("keydown", handler)
}, [])
```

## AP-8 : Calling setOpen(false) Before The Async Mutation Resolves

### Symptom

The user submits a form inside a dialog. The dialog closes
immediately. The submit button never shows its "Saving..." state.
On error, the dialog is already closed and the form errors are
lost.

### WRONG

```tsx
const mutation = useMutation({ mutationFn: api.saveUser })

<form
  onSubmit={async (e) => {
    e.preventDefault()
    setOpen(false)                  // closes BEFORE the request fires
    await mutation.mutateAsync(data)
  }}
>
  ...
</form>
```

### Why It Breaks

`setOpen(false)` triggers the close-and-unmount sequence
immediately. By the time the await on `mutateAsync` resolves, the
DialogContent has been unmounted and any error toast that depends
on the form's state has no anchor.

### CORRECT

```tsx
const mutation = useMutation({
  mutationFn: api.saveUser,
  onSuccess: () => {
    setOpen(false)                  // close ONLY on success
    toast.success("Saved.")
  },
  onError: (err) => {
    // dialog stays open ; user can fix and retry
    toast.error((err as Error).message)
  },
})

<form
  onSubmit={(e) => {
    e.preventDefault()
    mutation.mutate(data)
  }}
>
  <Button type="submit" disabled={mutation.isPending}>
    {mutation.isPending ? "Saving..." : "Save"}
  </Button>
</form>
```

The mutation owns the close decision. Success closes ; error keeps
the dialog open so the user can read the error and retry.

For a Sheet, Drawer, or Popover containing a form, apply the same
rule : close inside `onSuccess`, not inside the submit handler.

## Cross-Reference

- Companion : `shadcn-syntax-dialog` (Dialog composition)
- Companion : `shadcn-syntax-popover-tooltip-hovercard` (modal prop)
- Companion : `shadcn-syntax-drawer` (Vaul controlled state)
- Companion : `shadcn-syntax-menu-primitives` (DropdownMenu /
  ContextMenu / `onSelect.preventDefault`)
- Companion : `shadcn-errors-form-state` (form + controlled dialog
  interaction)
- Verified sources :
  - https://www.radix-ui.com/primitives/docs/components/dialog
  - https://www.radix-ui.com/primitives/docs/components/popover
  - https://www.radix-ui.com/primitives/docs/utilities/slot
  - https://ui.shadcn.com/docs/components/dialog
