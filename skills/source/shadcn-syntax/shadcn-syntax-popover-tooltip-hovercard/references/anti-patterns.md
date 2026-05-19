# Anti-patterns : Popover, Tooltip, HoverCard

Eight verified failure modes with WHY each fails and what to do instead. Each entry references the official shadcn / Radix docs or the registry source.

## 1. Missing TooltipProvider : Tooltip silent fail

### Symptom

Tooltips render no content. No console error. The trigger is hoverable, but nothing ever appears.

### Wrong code

```tsx
// app/page.tsx
"use client"

import {
  Tooltip,
  TooltipTrigger,
  TooltipContent,
} from "@/components/ui/tooltip"

export default function Page() {
  return (
    <Tooltip>
      <TooltipTrigger>Hover me</TooltipTrigger>
      <TooltipContent>No provider, no tooltip.</TooltipContent>
    </Tooltip>
  )
}
```

### Why it fails

Radix `Tooltip.Root` depends on a `Tooltip.Provider` context value for its delay machinery. Without the Provider, `Tooltip.Root` short-circuits and renders nothing. The shadcn `TooltipProvider` component is a thin wrapper around `Tooltip.Provider` and MUST be mounted exactly once at the React tree root.

### Right code

```tsx
// app/layout.tsx (Next.js App Router)
import { TooltipProvider } from "@/components/ui/tooltip"

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <TooltipProvider>{children}</TooltipProvider>
      </body>
    </html>
  )
}
```

ALWAYS mount one `<TooltipProvider>` per render tree. The shadcn registry source `apps/v4/registry/new-york-v4/ui/tooltip.tsx` ships `delayDuration={0}` as the Provider override; an absent Provider also means no `delayDuration` resolution and no tooltips.

## 2. Using Tooltip for interactive content : keyboard and touch users locked out

### Symptom

The Tooltip works on desktop with a mouse. Keyboard-only users cannot reach the link or button inside the tooltip. Mobile users see nothing at all.

### Wrong code

```tsx
<Tooltip>
  <TooltipTrigger asChild>
    <button>Help</button>
  </TooltipTrigger>
  <TooltipContent>
    Read our <a href="/docs">documentation</a> or
    <button onClick={contactSupport}>contact support</button>.
  </TooltipContent>
</Tooltip>
```

### Why it fails

Three failure layers compound :

1. `TooltipContent` is wired to the trigger via `aria-describedby`. Screen readers read the content as a label, NOT as a navigable region. The link and the support button are invisible to assistive technology.
2. The Tooltip closes on `pointerLeave` of the trigger. The moment a mouse user tries to move the cursor INTO the Tooltip, the trigger loses hover and the Tooltip closes. Radix mitigates with `disableHoverableContent={false}` but it is still fragile.
3. On touch devices Radix hides Tooltip by default (verified in shadcn docs `/docs/components/tooltip` : "Tooltips remain hidden on touch devices unless explicitly configured otherwise"). Mobile users never see the content.

### Right code

```tsx
<HoverCard>
  <HoverCardTrigger asChild>
    <button type="button">Help</button>
  </HoverCardTrigger>
  <HoverCardContent>
    Read our <a href="/docs">documentation</a> or
    <button onClick={contactSupport}>contact support</button>.
  </HoverCardContent>
</HoverCard>
```

Or, if the trigger is itself a click target (button), prefer Popover :

```tsx
<Popover>
  <PopoverTrigger asChild>
    <Button>Help</Button>
  </PopoverTrigger>
  <PopoverContent>
    Read our <a href="/docs">documentation</a> or
    <Button onClick={contactSupport}>contact support</Button>.
  </PopoverContent>
</Popover>
```

NEVER place focusable elements inside `<TooltipContent>`. Tooltip is text-only by Radix contract.

## 3. Popover `modal={true}` containing a form Submit : focus restore fights the form

### Symptom

After clicking Submit inside a modal Popover, focus jumps to the body or to the trigger, the form state machine looks stale, and the next Tab cycle starts from `<body>` instead of from the trigger.

### Wrong code

```tsx
<Popover modal={true}>
  <PopoverTrigger asChild>
    <Button>Edit</Button>
  </PopoverTrigger>
  <PopoverContent>
    <form onSubmit={handleSubmit}>
      <Input name="title" />
      <Button type="submit">Save</Button>
    </form>
  </PopoverContent>
</Popover>
```

### Why it fails

With `modal={true}`, Radix Popover traps focus inside the Content AND adds an inert overlay to the page. On Submit, the form re-renders (typically clearing the inputs); Radix attempts to restore focus to the trigger on close; React's commit phase races the trigger-restore. Net effect : focus lands inconsistently (sometimes trigger, sometimes body), and screen-reader users hear "Edit button" followed by silence.

Verified pattern across GitHub issues : `modal={true}` is intended for Popovers that are "almost-Dialogs" but ship without a backdrop. As soon as a `<form>` Submit lives inside, Dialog is the right primitive.

### Right code (option A : keep Popover, drop modal)

```tsx
<Popover open={open} onOpenChange={setOpen}>
  <PopoverTrigger asChild>
    <Button>Edit</Button>
  </PopoverTrigger>
  <PopoverContent>
    <form
      onSubmit={async (e) => {
        e.preventDefault()
        await save(...)
        setOpen(false)
      }}
    >
      <Input name="title" />
      <Button type="submit">Save</Button>
    </form>
  </PopoverContent>
</Popover>
```

### Right code (option B : escalate to Dialog)

```tsx
<Dialog open={open} onOpenChange={setOpen}>
  <DialogTrigger asChild>
    <Button>Edit</Button>
  </DialogTrigger>
  <DialogContent>
    <DialogTitle>Edit title</DialogTitle>
    <form onSubmit={...}>...</form>
  </DialogContent>
</Dialog>
```

ALWAYS prefer Dialog for "focus-trapped form panel". Popover `modal={true}` is a niche escape hatch.

## 4. HoverCard without `openDelay` tuning : flicker on avatar / mention lists

### Symptom

Hovering across a list of 20 user mentions opens-and-closes 20 HoverCards in rapid succession. Performance drops; the screen flickers.

### Wrong code

```tsx
{mentions.map((m) => (
  <HoverCard key={m.id}>
    <HoverCardTrigger asChild>
      <a href={`/users/${m.handle}`}>@{m.handle}</a>
    </HoverCardTrigger>
    <HoverCardContent>{/* profile preview */}</HoverCardContent>
  </HoverCard>
))}
```

### Why it fails

Radix `HoverCard` defaults : `openDelay={700}`, `closeDelay={300}`. In a horizontal mention list, the cursor crosses each trigger in roughly 200ms. Each crossing triggers `pointerEnter`; the 700ms delay times out late enough that the next trigger's `pointerEnter` has already fired. The result is each trigger queuing an open-then-immediate-close cycle.

Counter-intuitively, the fix is NOT a longer `openDelay` (delays panel open even more) but a shorter `openDelay` PLUS a shorter `closeDelay` so the open / close cycle resolves cleanly before the next trigger is reached.

### Right code

```tsx
{mentions.map((m) => (
  <HoverCard key={m.id} openDelay={400} closeDelay={150}>
    <HoverCardTrigger asChild>
      <a href={`/users/${m.handle}`}>@{m.handle}</a>
    </HoverCardTrigger>
    <HoverCardContent>{/* profile preview */}</HoverCardContent>
  </HoverCard>
))}
```

`openDelay={400}` matches the empirical "cursor lingers ON-target for >300ms" threshold. `closeDelay={150}` collapses the queue cleanly. Verified pattern : the shadcn docs example uses `openDelay` overrides; the GitHub issue history on Radix HoverCard #1428 documents the flicker case.

## 5. Tooltip-as-Popover replacement : a11y violation across the board

### Symptom

A click-to-open detail panel was built with Tooltip "because it looked like the right component". Keyboard users cannot open it; touch users cannot see it; screen-reader users hear the entire content as the trigger's label.

### Wrong code

```tsx
<Tooltip open={open} onOpenChange={setOpen}>
  <TooltipTrigger asChild>
    <Button onClick={() => setOpen((v) => !v)}>Details</Button>
  </TooltipTrigger>
  <TooltipContent>
    <h3>Item details</h3>
    <p>Long descriptive text...</p>
    <Button>Edit</Button>
  </TooltipContent>
</Tooltip>
```

### Why it fails

- `aria-describedby` floods the screen reader with the entire content as a description of the button.
- Touch devices hide the Tooltip entirely (verified in shadcn docs `/docs/components/tooltip`).
- The interactive Edit button inside is keyboard-unreachable.
- The controlled `open` toggling fights Radix's pointer-based open/close logic; sometimes the Tooltip stays open after pointer-leave, sometimes it does not.

### Right code

```tsx
<Popover open={open} onOpenChange={setOpen}>
  <PopoverTrigger asChild>
    <Button>Details</Button>
  </PopoverTrigger>
  <PopoverContent className="w-80">
    <h3 className="text-sm font-medium">Item details</h3>
    <p className="text-sm">Long descriptive text...</p>
    <Button size="sm" className="mt-2">Edit</Button>
  </PopoverContent>
</Popover>
```

ALWAYS pick the primitive by the UX shape : click + interactive content = Popover. Tooltip is short non-interactive hover-text only. Treat Tooltip as "label", not as "panel".

## 6. Controlled root without `onOpenChange` : stuck-open panel

### Symptom

A Popover, Tooltip, or HoverCard opens but refuses to close. Escape does nothing, outside-click does nothing, the Submit handler closes nothing.

### Wrong code

```tsx
const [open, setOpen] = React.useState(true)

<Popover open={open}>
  <PopoverTrigger asChild>
    <Button>Open</Button>
  </PopoverTrigger>
  <PopoverContent>Stuck open forever.</PopoverContent>
</Popover>
```

### Why it fails

Passing `open` without `onOpenChange` puts Radix into half-controlled mode : the prop value is authoritative, but there is no callback for Radix to request a state change. Every Escape / outside-click / programmatic close attempt fires `onOpenChange` against `undefined` and the state never updates. The panel is effectively read-only.

### Right code

```tsx
const [open, setOpen] = React.useState(false)

<Popover open={open} onOpenChange={setOpen}>
  <PopoverTrigger asChild>
    <Button>Open</Button>
  </PopoverTrigger>
  <PopoverContent>Closable.</PopoverContent>
</Popover>
```

ALWAYS pair `open` with `onOpenChange`. If the state lives in a parent and should not be writable from inside the panel, pass `onOpenChange={() => {}}` only as an explicit decision (rare; usually a bug).

Identical rule for Tooltip and HoverCard roots. See `references/methods.md` §5 controlled-vs-uncontrolled table.

## 7. Popover inside Dialog without Portal scoping : popover renders BEHIND the dialog overlay

### Symptom

A Popover trigger inside a Dialog opens, but the PopoverContent disappears behind the Dialog's dim overlay. Users see the trigger flash but no panel.

### Wrong code

```tsx
<Dialog>
  <DialogTrigger asChild>
    <Button>Open Dialog</Button>
  </DialogTrigger>
  <DialogContent>
    <Popover>
      <PopoverTrigger asChild>
        <Button>Nested</Button>
      </PopoverTrigger>
      <PopoverContent>I am behind the dialog overlay.</PopoverContent>
    </Popover>
  </DialogContent>
</Dialog>
```

### Why it fails

Both Dialog and Popover use `Portal` to render their content at `document.body`. Stacking order is determined by mount order and z-index. The shadcn Dialog `DialogOverlay` and `DialogContent` both use `z-50`. The shadcn Popover `PopoverContent` also uses `z-50`. Same z-index + body Portals = mount-order wins. The Popover mounts AFTER the Dialog opens, but its Portal target is the same body, and the Dialog overlay was already painted on top.

### Right code (option A : bump Popover z-index)

```tsx
<PopoverContent className="z-[60]">
  Now in front of the dialog overlay.
</PopoverContent>
```

### Right code (option B : scope the Portal inside the Dialog)

Radix Popover exposes a `Portal` slot with a `container` prop. shadcn's `PopoverContent` always Portals to `document.body`; for in-Dialog scoping you must use Radix primitives directly :

```tsx
import * as PopoverPrimitive from "@radix-ui/react-popover"

<DialogContent ref={dialogRef}>
  <PopoverPrimitive.Root>
    <PopoverPrimitive.Trigger asChild>
      <Button>Nested</Button>
    </PopoverPrimitive.Trigger>
    <PopoverPrimitive.Portal container={dialogRef.current}>
      <PopoverPrimitive.Content className="z-50 ...">
        Scoped to the Dialog.
      </PopoverPrimitive.Content>
    </PopoverPrimitive.Portal>
  </PopoverPrimitive.Root>
</DialogContent>
```

ALWAYS prefer option A for shadcn-style code; option B is the escape hatch when stacking contexts conflict (e.g. nested portals inside iframes).

## 8. HoverCard trigger as non-focusable div : keyboard users locked out

### Symptom

Mouse users see the HoverCard; keyboard users tabbing through the page never see the preview content.

### Wrong code

```tsx
<HoverCard>
  <HoverCardTrigger asChild>
    <div className="font-medium">@freek</div>
  </HoverCardTrigger>
  <HoverCardContent>Profile preview.</HoverCardContent>
</HoverCard>
```

### Why it fails

HoverCard opens on `focus` of the trigger in addition to `pointerEnter`. A `<div>` is not in the Tab order; keyboard users cannot reach it; the HoverCard never opens for them. This is a WCAG 2.1.1 Keyboard violation : functionality available to mouse users must also be available to keyboard users.

### Right code

```tsx
<HoverCard>
  <HoverCardTrigger asChild>
    <a href="/users/freek" className="font-medium">
      @freek
    </a>
  </HoverCardTrigger>
  <HoverCardContent>Profile preview.</HoverCardContent>
</HoverCard>
```

Or, if there is no `href` target, make the trigger a button :

```tsx
<HoverCard>
  <HoverCardTrigger asChild>
    <button type="button" className="font-medium">
      @freek
    </button>
  </HoverCardTrigger>
  <HoverCardContent>Profile preview.</HoverCardContent>
</HoverCard>
```

ALWAYS use an anchor or a button for HoverCardTrigger. NEVER use a div without `tabIndex={0}`. The `tabIndex={0}` div escape hatch works mechanically but signals nothing to assistive tech about the trigger's purpose; prefer real anchors / buttons whenever possible.
