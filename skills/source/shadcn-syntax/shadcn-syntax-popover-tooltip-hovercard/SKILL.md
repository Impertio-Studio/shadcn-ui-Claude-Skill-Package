---
name: shadcn-syntax-popover-tooltip-hovercard
description: >
  Use when picking between Popover, Tooltip, and HoverCard for any floating
  panel above the page : a click-to-open form panel, a short hover hint on
  an icon button, a rich preview card that appears when the cursor lingers
  on a user mention, a date picker container, a filter popover, or any
  other "small panel anchored to an element" UX. Use when debugging why a
  Tooltip never appears (missing `TooltipProvider` at the app root), why a
  HoverCard flickers as the cursor passes over an avatar list (zero
  `openDelay`), why a Tooltip works on desktop but is invisible on iPad
  (Radix hides Tooltip on touch by design), why focus jumps to the body
  after a Popover closes (`modal={true}` focus-trap on a non-modal panel),
  or why a Popover refuses to close when controlled (passing `open` without
  `onOpenChange`). Use when choosing which primitive to install for an
  emoji-picker, a colour-picker, a notification preview, a profile-card
  preview on hover, a help-text hint on a form label, or a confirm-action
  popover.
  Prevents the canonical floating-panel mismatches : reaching for Tooltip
  when the content is interactive (links / buttons / inputs inside a
  Tooltip lose keyboard navigation and disappear on touch devices),
  reaching for HoverCard when the trigger is itself interactive AND the
  content must be reachable by keyboard-only users (hover-only is an a11y
  violation), reaching for Popover when a one-line non-interactive hint
  would suffice (Popover forces a click and steals focus on open), forgetting
  to wrap the React tree in `<TooltipProvider>` so every `<Tooltip>` is a
  silent no-op, nesting a Popover inside a Dialog without re-portalling so
  the popover renders behind the Dialog overlay, and setting `modal={true}`
  on a Popover that contains a `<form>` Submit so focus restoration fights
  the form-state machine on close.
  Covers the three Radix-backed primitives shipped in shadcn ui new-york-v4
  (`apps/v4/registry/new-york-v4/ui/{popover,tooltip,hover-card}.tsx`),
  including the full export surface (Popover : Popover / PopoverTrigger /
  PopoverContent / PopoverAnchor / PopoverHeader / PopoverTitle /
  PopoverDescription, Tooltip : TooltipProvider / Tooltip / TooltipTrigger
  / TooltipContent, HoverCard : HoverCard / HoverCardTrigger /
  HoverCardContent), the shadcn-specific default overrides (TooltipProvider
  `delayDuration={0}` shadcn default vs Radix-700 underlying default,
  PopoverContent `sideOffset={4}`, TooltipContent `sideOffset={0}`,
  HoverCardContent `sideOffset={4}`, all three `align="center"`), the
  controlled-state pattern (`open` + `onOpenChange`), the `modal` prop on
  Popover (focus-trap on vs off), the touch-device behaviour matrix
  (Tooltip hidden, HoverCard tap-to-open, Popover always-on), the
  accessibility matrix (`aria-describedby` for Tooltip, `aria-haspopup`
  for HoverCard, focus management for Popover), and the `"use client"`
  requirement that applies to all three.
  Keywords: shadcn popover, shadcn tooltip, shadcn hover card, shadcn
  hovercard, Popover vs Tooltip, Tooltip vs HoverCard, HoverCard vs
  Popover, when to use popover, when to use tooltip, when to use
  hovercard, click popup, hover content, hover popup, floating panel,
  TooltipProvider, TooltipProvider missing, tooltip not showing, tooltip
  not appearing, tooltip silent fail, why is my tooltip blank, tooltip
  not appearing on touch, tooltip on mobile, tooltip on iPad,
  tooltip hidden touch device, delayDuration, openDelay, closeDelay,
  hover-card delay, hover card flicker, hover-card instant, PopoverAnchor,
  Popover modal, modal popover, modal=true Popover, popover focus trap,
  popover form submit focus, popover with form, popover with input,
  popover with date picker, popover with colour picker, popover with
  emoji picker, tooltip on icon button, tooltip on disabled button,
  hover card user profile, hover card profile preview, hover card mention,
  hover card avatar preview, Tooltip with interactive content, HoverCard
  keyboard accessibility, Popover behind Dialog, Popover inside Dialog,
  Popover z-index, Portal Popover, controlled Popover, controlled Tooltip,
  controlled HoverCard, open and onOpenChange, why does my popover not
  close, why does my popover close on submit, how do I show a hint on
  hover, how do I show extra info on hover, plain hover help text,
  how to add hover preview, how to show profile card on hover, what is
  the difference between Popover and Tooltip, use client floating panel,
  Radix Popover, Radix Tooltip, Radix HoverCard, aria-describedby,
  aria-haspopup, floating-ui delay, hover delay popup, popup on click,
  popup on hover.
license: MIT
compatibility: "Designed for Claude Code. Requires shadcn ui evergreen-2026."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# shadcn ui : Popover vs Tooltip vs HoverCard

Three Radix-backed floating-panel primitives ship in shadcn ui new-york-v4. They look similar (a panel anchored to a trigger) but solve three disjoint UX problems. Picking the wrong one breaks keyboard navigation, touch users, screen readers, or all three at once. This skill is a decision tree, not three API surfaces stacked together. The full prop tables live in `references/methods.md`; the working recipes live in `references/examples.md`; the failure modes live in `references/anti-patterns.md`.

## The single decision

```
Is the panel content interactive (inputs, links, buttons, anything
keyboard-focusable)?
├─ NO  -> Is the hint a short, plain-text label (one sentence max)?
│         ├─ YES -> Tooltip
│         └─ NO  -> Popover (open it on click, even for read-only content)
└─ YES -> Should the panel appear on hover, or on click?
          ├─ HOVER (and the trigger is keyboard-reachable too) -> HoverCard
          └─ CLICK                                              -> Popover
```

Three additional gates ALWAYS override the table above :

1. If the trigger is on a touch device and the content is critical : NEVER Tooltip. Tooltip is hidden on touch by Radix design. Use Popover instead.
2. If the content contains a `<form>` with a Submit button : NEVER Tooltip and NEVER HoverCard. Use Popover; both Tooltip and HoverCard close on pointer-leave, which races the form submission.
3. If the content is keyboard-only-reachable via the trigger : NEVER Tooltip and NEVER HoverCard. Tooltip cannot host interactive elements; HoverCard is hover-only and fails WCAG 2.1.1 Keyboard. Use Popover.

## Decision Matrix at a glance

| Need | Primitive | Trigger | Content | a11y |
|------|-----------|---------|---------|------|
| One-line label on icon button | Tooltip | hover / focus | text only | aria-describedby |
| One-line label on disabled button | Tooltip (wrap a `<span>`) | hover / focus | text only | aria-describedby |
| Rich profile preview on mention | HoverCard | hover / focus | text + image + links | aria-haspopup |
| Date picker calendar | Popover | click | Calendar component | focus moves into panel |
| Colour picker | Popover | click | Sliders / inputs | focus moves into panel |
| Emoji picker | Popover | click | grid + search input | focus moves into panel |
| Help text balloon below a form Label | Tooltip | hover / focus | text only | aria-describedby |
| Confirm action panel ("are you sure?") | Popover | click | text + 2 buttons | focus moves into panel |
| Filter popover for a table column | Popover | click | inputs + chips | focus moves into panel |

## The three primitives : compact signatures

ALL three components require `"use client"`. ALL three ship as kebab-case CLI installs and import from `@/components/ui/{popover,tooltip,hover-card}`.

### Popover

```tsx
"use client"

import {
  Popover,
  PopoverTrigger,
  PopoverContent,
} from "@/components/ui/popover"
import { Button } from "@/components/ui/button"

<Popover>
  <PopoverTrigger asChild>
    <Button variant="outline">Open</Button>
  </PopoverTrigger>
  <PopoverContent className="w-80">
    Anything interactive lives here.
  </PopoverContent>
</Popover>
```

Install : `pnpm dlx shadcn@latest add popover`.

`PopoverContent` shadcn defaults : `align="center"`, `sideOffset={4}`. Wraps Radix `PopoverPrimitive.Content` inside `PopoverPrimitive.Portal` automatically. `modal` defaults to `false`; set `modal={true}` ONLY when the popover must trap focus (rare; Dialog is usually a better fit).

Also exported : `PopoverAnchor` (decouple positioning anchor from the trigger), `PopoverHeader`, `PopoverTitle`, `PopoverDescription` (semantic composition helpers, optional).

### Tooltip

```tsx
"use client"

import {
  TooltipProvider,
  Tooltip,
  TooltipTrigger,
  TooltipContent,
} from "@/components/ui/tooltip"

// At app root (layout / _app / main.tsx) :
<TooltipProvider>{children}</TooltipProvider>

// At usage site :
<Tooltip>
  <TooltipTrigger asChild>
    <button aria-label="Save">
      <SaveIcon />
    </button>
  </TooltipTrigger>
  <TooltipContent>Save (Ctrl+S)</TooltipContent>
</Tooltip>
```

Install : `pnpm dlx shadcn@latest add tooltip`.

ALWAYS mount `<TooltipProvider>` at the app root. Every `<Tooltip>` is a silent no-op without an enclosing Provider. The shadcn `TooltipProvider` ships `delayDuration={0}` (immediate tooltips); the underlying Radix default is `700`. Override per-Tooltip with `<Tooltip delayDuration={500}>` when a slower hover-in is appropriate. `TooltipContent` shadcn default : `sideOffset={0}` (Radix default is `0` too) plus a built-in `<TooltipPrimitive.Arrow/>` rendered inside the content. Wraps Radix `TooltipPrimitive.Content` inside a `Portal`.

Tooltip content MUST be plain text. NEVER place buttons, links, inputs, or any focusable element inside `<TooltipContent>`. The Tooltip is hidden on touch devices by Radix design; treat the hint as decorative on mobile.

### HoverCard

```tsx
"use client"

import {
  HoverCard,
  HoverCardTrigger,
  HoverCardContent,
} from "@/components/ui/hover-card"

<HoverCard openDelay={400} closeDelay={200}>
  <HoverCardTrigger asChild>
    <a href="/users/freek">@freek</a>
  </HoverCardTrigger>
  <HoverCardContent className="w-80">
    Rich profile preview, including avatar, bio, follow button.
  </HoverCardContent>
</HoverCard>
```

Install : `pnpm dlx shadcn@latest add hover-card`.

NO provider needed; HoverCard is self-contained. Radix defaults : `openDelay={700}`, `closeDelay={300}`. Override per-instance for hover-flicker-prone UIs (avatar lists, mention lists, link-preview hovers) : `openDelay={400}` is a good baseline. `HoverCardContent` shadcn defaults : `align="center"`, `sideOffset={4}`. Wraps Radix `HoverCardPrimitive.Content` inside a `Portal`.

HoverCard works on touch (tap to open, tap-outside to close), but the trigger MUST be a real focusable element (anchor, button, or `tabIndex={0}` element) so keyboard users can reach the content via focus + Enter.

## TooltipProvider placement rule

The `<TooltipProvider>` is the ONE mandatory wrapper of the three primitives. Place it ONCE at the app root :

- Next.js App Router : in `app/layout.tsx`, wrap `{children}` inside the `<body>`.
- Next.js Pages Router : in `pages/_app.tsx`, wrap `<Component {...pageProps} />`.
- Vite / SPA : in `src/main.tsx` (or wherever you mount `<App/>`), wrap `<App/>`.
- Remix : in `app/root.tsx`, wrap the `<Outlet/>`.

ALWAYS one Provider per render tree. Nested Providers compound `delayDuration` resets and break the `skipDelayDuration` skip-window; nesting is supported by Radix but ALWAYS unnecessary and confusing.

Per-instance overrides go on the `<Tooltip>` itself, not on a new Provider :

```tsx
<Tooltip delayDuration={500} disableHoverableContent>
  <TooltipTrigger asChild>
    <button>?</button>
  </TooltipTrigger>
  <TooltipContent>Slow hint.</TooltipContent>
</Tooltip>
```

HoverCard and Popover do NOT need a provider. There is no `HoverCardProvider` or `PopoverProvider` in the shadcn ui surface.

## Touch-device behaviour matrix

| Primitive | Touch behaviour | Implication |
|-----------|-----------------|-------------|
| Tooltip | Hidden by default on touch (Radix design choice) | NEVER use Tooltip for critical info |
| HoverCard | Opens on tap; closes on tap-outside | Works on mobile but watch z-index |
| Popover | Opens on click (same as tap); no hover dependence | Always works on touch |

Verify this in the official shadcn docs (`/docs/components/tooltip`) : "Tooltips remain hidden on touch devices unless explicitly configured otherwise." The shadcn position is to honour the Radix default; do NOT patch the underlying `Tooltip.Provider` to force-show on touch. Instead, escalate to Popover when the hint is essential.

## Accessibility matrix

| Primitive | ARIA on content | Focus behaviour on open | Keyboard reachable? |
|-----------|-----------------|-------------------------|---------------------|
| Popover | none required; content is generic | focus moves into Content (first focusable element) | YES |
| Tooltip | `aria-describedby` links trigger to content | NO focus move (Tooltip never receives focus) | NO (content text is read via aria-describedby) |
| HoverCard | `aria-haspopup` on the trigger | NO focus move (hover/focus parity, content is content-only) | YES (focus on trigger opens it, Tab moves into content) |

Tooltip MUST receive its label-content via plain text children; screen readers read it via `aria-describedby`. NEVER nest a Button, Link, or focusable element inside `<TooltipContent>` : the screen reader cannot reach those elements through `aria-describedby` and keyboard users have no way to interact with them.

HoverCard's trigger MUST be a real focusable element (button, anchor, or `tabIndex={0}`). The keyboard accessibility for HoverCard is "focus the trigger -> content opens -> Tab moves into content"; if the trigger is a div without `tabIndex`, the HoverCard is keyboard-inaccessible (WCAG 2.1.1 violation).

Popover does its own focus management : focus moves to the first focusable element inside `PopoverContent` on open, and restores to the trigger on close. With `modal={true}`, Popover ALSO traps focus inside the content until close; with `modal={false}` (the default), focus can be tabbed out into the page (a desktop hover-menu UX, rarely what you want).

## Common props that ALL three Content components share

All three `*Content` components render Radix Popper-positioned floating elements and accept the same positioning prop surface : `side` ("top"|"right"|"bottom"|"left", default `"bottom"`), `sideOffset` (defaults : Popover `4`, HoverCard `4`, Tooltip `0`), `align` ("start"|"center"|"end", default `"center"` on Popover and HoverCard, `"center"` on Tooltip too), `alignOffset` (default `0`), `avoidCollisions` (default `true`), `collisionBoundary`, `collisionPadding`, `sticky` ("partial"|"always", default `"partial"`), `hideWhenDetached` (default `false`), `forceMount`. The Tooltip Content additionally renders an `<Arrow/>` element internally. The full per-component prop matrix lives in `references/methods.md`.

## Companion Skills

Read these from `skills/source/shadcn-syntax/` on demand only :

- `shadcn-syntax-dialog` : if the panel must trap focus AND host a multi-step form, use Dialog instead of `Popover modal={true}`. Dialog is the modal primitive; Popover modal is a niche escape hatch.
- `shadcn-syntax-sheet` : side-anchored modal panel; same rule as Dialog but for slide-in side panels.
- `shadcn-syntax-menu-primitives` : DropdownMenu / ContextMenu / Menubar / NavigationMenu. ALWAYS prefer a DropdownMenu over a Popover-containing-buttons; menus have built-in roving-tabindex keyboard navigation that Popovers lack.
- `shadcn-syntax-toast-sonner` : for ephemeral notifications use Sonner toasts, NEVER a Tooltip or HoverCard (toasts are page-level, the three primitives in this skill are anchored to a trigger).
- `shadcn-syntax-calendar-datepicker` : the canonical Popover + Calendar composition for date pickers.
- `shadcn-errors-radix-controlled` : controlled-state pitfalls (`open` without `onOpenChange`, Portal z-index surprises, asChild misuse). Read when debugging a panel that "is stuck open" or "won't close".

## Reference Links

- shadcn Popover docs : https://ui.shadcn.com/docs/components/popover
- shadcn Tooltip docs : https://ui.shadcn.com/docs/components/tooltip
- shadcn HoverCard docs : https://ui.shadcn.com/docs/components/hover-card
- Radix Popover API : https://www.radix-ui.com/primitives/docs/components/popover
- Radix Tooltip API : https://www.radix-ui.com/primitives/docs/components/tooltip
- Radix HoverCard API : https://www.radix-ui.com/primitives/docs/components/hover-card
- WCAG 2.1.1 Keyboard : https://www.w3.org/WAI/WCAG21/Understanding/keyboard.html
- shadcn registry source (popover.tsx) : `apps/v4/registry/new-york-v4/ui/popover.tsx`
- shadcn registry source (tooltip.tsx) : `apps/v4/registry/new-york-v4/ui/tooltip.tsx`
- shadcn registry source (hover-card.tsx) : `apps/v4/registry/new-york-v4/ui/hover-card.tsx`

Detailed prop tables : `references/methods.md`. Working recipes : `references/examples.md`. Failure modes and fixes : `references/anti-patterns.md`.
