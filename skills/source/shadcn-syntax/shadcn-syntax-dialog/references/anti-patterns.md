# Dialog : Anti-Patterns

Six concrete failures that block real shadcn ui Dialog usage. Each entry follows : WRONG code, WHY it fails, FIX.

Sources : `apps/v4/registry/new-york-v4/ui/dialog.tsx`, https://www.radix-ui.com/primitives/docs/components/dialog, https://github.com/shadcn-ui/ui/issues (issues #5746, vooronderzoek §9 entries 3-4 + 18). Verified 2026-05-19.

## 1. Passing `open` without `onOpenChange`

### WRONG

```tsx
const [open, setOpen] = useState(true)

<Dialog open={open}>
  <DialogContent>
    <DialogTitle>Help</DialogTitle>
    <DialogClose>Close</DialogClose>
  </DialogContent>
</Dialog>
```

### WHY it fails

Radix treats `open` as the authoritative prop. With no `onOpenChange`, Radix has nowhere to report state-change intent. Clicking DialogClose, pressing Esc, and clicking outside all fire internal change-events that get DROPPED. The dialog appears stuck open and uncloseable. The reverse failure : passing `onOpenChange` without `open` causes the callback to fire but the dialog still uses its uncontrolled internal state, so the parent never reflects reality.

Verified in vooronderzoek §9 entry 3 : "passing `open={true}` without `onOpenChange` makes the Dialog read-only and uncloseable".

### FIX

Pair them, every time :

```tsx
const [open, setOpen] = useState(true)

<Dialog open={open} onOpenChange={setOpen}>
  <DialogContent>
    <DialogTitle>Help</DialogTitle>
    <DialogClose>Close</DialogClose>
  </DialogContent>
</Dialog>
```

If you don't need controlled state, pass NEITHER prop. Use `defaultOpen` for a one-shot initial-open with no ongoing control.

## 2. Missing DialogTitle

### WRONG

```tsx
<Dialog>
  <DialogTrigger asChild><Button>Open</Button></DialogTrigger>
  <DialogContent>
    <DialogDescription>Choose a colour theme.</DialogDescription>
    <RadioGroup>…</RadioGroup>
  </DialogContent>
</Dialog>
```

### WHY it fails

Radix Dialog implements the WAI-ARIA Dialog pattern, which requires every modal dialog to have an accessible name via `aria-labelledby`. shadcn's Dialog auto-wires this to `<DialogTitle>`. Without DialogTitle :

1. Radix logs to the console : `Warning: DialogContent requires a DialogTitle for the component to be accessible for screen reader users.`
2. axe-core / Lighthouse audits flag the dialog as critical : "Element does not have an accessible name" (rule `aria-dialog-name`).
3. Screen-reader users hear only "dialog" with no further context.

Issue #5746 shows the same failure mode in the mobile Sidebar variant.

### FIX

Always include a DialogTitle. If it must be visually hidden, use `sr-only` :

```tsx
<Dialog>
  <DialogTrigger asChild><Button>Open</Button></DialogTrigger>
  <DialogContent>
    <DialogTitle className="sr-only">Theme picker</DialogTitle>
    <DialogDescription>Choose a colour theme.</DialogDescription>
    <RadioGroup>…</RadioGroup>
  </DialogContent>
</Dialog>
```

Or wrap with the Radix VisuallyHidden primitive : `<VisuallyHidden asChild><DialogTitle>Theme picker</DialogTitle></VisuallyHidden>`. NEVER drop DialogTitle to silence the warning ; the a11y violation persists.

## 3. DialogContent outside DialogPortal (z-index conflicts)

### WRONG

```tsx
"use client"

import { Dialog as DialogPrimitive } from "radix-ui"

function CustomDialogContent({ children }) {
  // Reaches into Radix directly, skipping the Portal.
  return (
    <div className="relative z-10">
      <DialogPrimitive.Overlay className="fixed inset-0 bg-black/50" />
      <DialogPrimitive.Content className="fixed top-1/2 left-1/2 z-20">
        {children}
      </DialogPrimitive.Content>
    </div>
  )
}
```

### WHY it fails

By skipping `DialogPortal`, the dialog renders INSIDE the parent's stacking context. Any of these in an ancestor breaks z-index ordering :

- `transform: translate*`, `scale`, `rotate` (creates a new stacking context)
- `filter`, `backdrop-filter`
- `will-change: transform`
- `isolation: isolate`
- A modern CSS `container` query host

The dialog ends up beneath the page chrome it should overlay, or beneath a navigation bar's transformed shadow, or clipped by an `overflow: hidden` ancestor. The shadcn `z-50` token can't escape a stacking context whose root is at `z-0`. Verified in vooronderzoek §9 entry 18 : "Radix Portals render at `document.body`. If a parent uses `transform`, `filter`, or `will-change` the popper anchor reference can become wrong."

### FIX

Use `<DialogContent>` directly. It internally wraps in `<DialogPortal>` + `<DialogOverlay>` :

```tsx
<Dialog>
  <DialogTrigger asChild><Button>Open</Button></DialogTrigger>
  <DialogContent>
    <DialogTitle>OK</DialogTitle>
  </DialogContent>
</Dialog>
```

If you must retarget the portal (rare), use the explicit form with `container` :

```tsx
<DialogPortal container={hostRef.current}>
  <DialogOverlay />
  <DialogContent>…</DialogContent>
</DialogPortal>
```

NEVER hand-render `DialogPrimitive.Content` outside a Portal context.

## 4. asChild DialogClose with non-button child (Slot crashes)

### WRONG

```tsx
<DialogClose asChild>
  <Button variant="outline">Cancel</Button>
  <span className="ml-2 text-muted-foreground">(Esc)</span>
</DialogClose>
```

### WHY it fails

Radix `<DialogClose asChild>` renders via `Slot.Root`, which calls `React.Children.only`. It accepts EXACTLY one React element child. Two siblings throw at render time :

```
Error: React.Children.only expected to receive a single React element child.
```

The error originates in Radix Slot, not in DialogClose ; the stack trace points to `Slot.tsx` which is confusing the first time you see it.

A subtler variant : a child that ignores forwarded refs and props silently swallows the merged `onClick` :

```tsx
<DialogClose asChild>
  <div className="cursor-pointer">Cancel</div>
</DialogClose>
```

`<div>` does not forward refs and ignores most ARIA props. The dialog renders but clicking the div doesn't close it ; the merged onClick lands on a target that doesn't propagate.

Verified in vooronderzoek §9 entries 4-5.

### FIX

For multi-child layouts : compose INSIDE one element :

```tsx
<DialogClose asChild>
  <Button variant="outline">
    Cancel
    <span className="ml-2 text-muted-foreground">(Esc)</span>
  </Button>
</DialogClose>
```

For custom triggers : use a forwardRef-compatible component (Button, an `<a>`, a Radix-aware component). NEVER use a raw `<div>`.

## 5. Controlled state not synced with form-submit close

### WRONG

```tsx
const [open, setOpen] = useState(false)

return (
  <Dialog open={open} onOpenChange={setOpen}>
    <DialogTrigger asChild><Button>Edit</Button></DialogTrigger>
    <DialogContent>
      <DialogTitle>Edit</DialogTitle>
      <form onSubmit={handleSubmit(onSubmit)}>
        <Input {...register("name")} />
        <DialogFooter>
          <DialogClose asChild>
            <Button type="submit">Save</Button>
          </DialogClose>
        </DialogFooter>
      </form>
    </DialogContent>
  </Dialog>
)
```

### WHY it fails

`<DialogClose asChild><Button type="submit">Save</Button></DialogClose>` runs the close BEFORE the form's submit handler. The sequence is :

1. User clicks Save.
2. DialogClose fires its onClick : `onOpenChange(false)`.
3. The dialog unmounts (or animates closed).
4. The form's `onSubmit` fires on the unmounted form ; values may be stale or never reach the handler.
5. Validation errors, if any, never display because the dialog is already closing.

The dialog also closes on validation failure, leaving the user with no idea what went wrong.

### FIX

Plain Submit button. Close in the success handler.

```tsx
const [open, setOpen] = useState(false)

async function onSubmit(values) {
  await save(values)
  setOpen(false)   // ALWAYS after the await, in the success branch
}

return (
  <Dialog open={open} onOpenChange={setOpen}>
    <DialogTrigger asChild><Button>Edit</Button></DialogTrigger>
    <DialogContent>
      <DialogTitle>Edit</DialogTitle>
      <form onSubmit={handleSubmit(onSubmit)}>
        <Input {...register("name")} />
        <DialogFooter>
          <DialogClose asChild>
            <Button type="button" variant="outline">Cancel</Button>
          </DialogClose>
          <Button type="submit">Save</Button>
        </DialogFooter>
      </form>
    </DialogContent>
  </Dialog>
)
```

Submit is plain. Cancel uses DialogClose with `type="button"` so it can't accidentally submit. The dialog stays open on validation failure and closes only on success.

## 6. Nested Dialogs without separate portal scoping

### WRONG

```tsx
function Outer() {
  return (
    <Dialog>
      <DialogTrigger asChild><Button>Open outer</Button></DialogTrigger>
      <DialogContent>
        <DialogTitle>Outer</DialogTitle>
        <Inner />
      </DialogContent>
    </Dialog>
  )
}

function Inner() {
  return (
    <Dialog>
      <DialogTrigger asChild><Button>Open inner</Button></DialogTrigger>
      <DialogPortal container={document.querySelector(".outer-host")}>
        <DialogOverlay />
        <DialogContent>
          <DialogTitle>Inner</DialogTitle>
        </DialogContent>
      </DialogPortal>
    </Dialog>
  )
}
```

### WHY it fails

The outer dialog auto-portals to `document.body` with `z-50`. The inner dialog is told to render INSIDE the outer dialog's content host (`.outer-host`), so its overlay+content live in the outer's stacking context. The inner overlay ends up :

- Beneath the outer overlay (wrong DOM order)
- Constrained by the outer dialog's grid layout (visual misalignment)
- Trapped behind any later-portaled element with `z-50`

Additionally, both dialogs share Esc-key handling : pressing Esc closes BOTH, because Radix's pointer-down-outside + Esc handlers fire on whichever Dialog Content was last entered.

### FIX

Let both dialogs use the default portal. Stacking order follows DOM insertion order, and `document.body` ends each portal at the end of the document.

```tsx
function Inner() {
  return (
    <Dialog>
      <DialogTrigger asChild><Button>Open inner</Button></DialogTrigger>
      <DialogContent>
        <DialogTitle>Inner</DialogTitle>
      </DialogContent>
    </Dialog>
  )
}
```

The inner dialog now portals to `document.body` AFTER the outer dialog already mounted there. The inner overlay sits ABOVE the outer overlay. Esc closes the inner first (Radix tracks focus and routes Esc to the topmost open dialog), the outer second.

For confirm-within-confirm patterns, prefer `AlertDialog` for the inner ; it's purpose-built for the irreversible-action prompt and reuses the same a11y contract. See `shadcn-syntax-alert-dialog` (planned) or use Dialog twice with default portals as above.
