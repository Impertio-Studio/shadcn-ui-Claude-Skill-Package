# Methods : Popover, Tooltip, HoverCard API surfaces

All three primitives are Radix-backed thin wrappers shipped by shadcn ui new-york-v4. Source files :

- `apps/v4/registry/new-york-v4/ui/popover.tsx`
- `apps/v4/registry/new-york-v4/ui/tooltip.tsx`
- `apps/v4/registry/new-york-v4/ui/hover-card.tsx`

ALL three files start with `"use client"`. Importing from `@/components/ui/{popover,tooltip,hover-card}` in a React Server Component file requires a parent `"use client"` boundary, OR composing the primitives inside a client child component.

## 1. Popover : composition

```
Popover                  (root, manages open state)
├── PopoverTrigger       (the click target)
├── PopoverAnchor        (optional, decouples anchor from trigger)
└── PopoverContent       (the floating panel, auto-Portalled)
    ├── PopoverHeader    (optional, semantic block)
    │   ├── PopoverTitle (optional)
    │   └── PopoverDescription (optional)
    └── ...your content
```

### Popover (root) props

| Prop | Type | Default | Purpose |
|------|------|---------|---------|
| `open` | `boolean` | `undefined` | Controlled open state. ALWAYS pair with `onOpenChange`. |
| `defaultOpen` | `boolean` | `false` | Uncontrolled initial state. NEVER combine with `open`. |
| `onOpenChange` | `(open: boolean) => void` | `undefined` | Fires on open and on close. |
| `modal` | `boolean` | `false` | `true` traps focus inside Content until close. |

### PopoverTrigger props

| Prop | Type | Default | Purpose |
|------|------|---------|---------|
| `asChild` | `boolean` | `false` | Merge props onto the single child element instead of wrapping in a `<button>`. |

### PopoverAnchor props (optional)

| Prop | Type | Default | Purpose |
|------|------|---------|---------|
| `asChild` | `boolean` | `false` | When the visual anchor must differ from the click trigger. |

Use case : the Trigger is a small icon, but the panel must anchor to the parent row.

### PopoverContent props

| Prop | Type | Default | Purpose |
|------|------|---------|---------|
| `side` | `"top"\|"right"\|"bottom"\|"left"` | `"bottom"` | Side of the trigger to render on. |
| `sideOffset` | `number` | `4` | Pixels between trigger and content. shadcn default override. |
| `align` | `"start"\|"center"\|"end"` | `"center"` | Alignment along the side axis. shadcn default. |
| `alignOffset` | `number` | `0` | Pixels of alignment shift. |
| `avoidCollisions` | `boolean` | `true` | Auto-flip / shift to stay inside the viewport. |
| `collisionBoundary` | `Element \| Element[] \| null` | `null` | Constrain collision detection to a specific scrolling container. |
| `collisionPadding` | `number \| Padding` | `0` | Padding inside the collision boundary. |
| `sticky` | `"partial"\|"always"` | `"partial"` | Behaviour when the anchor scrolls partially out of view. |
| `hideWhenDetached` | `boolean` | `false` | Hide if the trigger has fully scrolled out. |
| `forceMount` | `boolean` | `undefined` | Always render to the DOM (useful for animation libraries). |
| `onOpenAutoFocus` | `(e: Event) => void` | `undefined` | Override the focus-on-open behaviour with `e.preventDefault()`. |
| `onCloseAutoFocus` | `(e: Event) => void` | `undefined` | Override the focus-restore behaviour with `e.preventDefault()`. |
| `onEscapeKeyDown` | `(e: KeyboardEvent) => void` | `undefined` | Cancel close on Escape with `e.preventDefault()`. |
| `onPointerDownOutside` | `(e: Event) => void` | `undefined` | Cancel close on outside click with `e.preventDefault()`. |
| `onFocusOutside` | `(e: Event) => void` | `undefined` | Cancel close on focus leaving with `e.preventDefault()`. |
| `onInteractOutside` | `(e: Event) => void` | `undefined` | Catch-all for outside interactions. |
| `asChild` | `boolean` | `false` | Render content directly on the child element. |
| `className` | `string` | `undefined` | Merged through `cn()` over the shadcn defaults. |

shadcn-applied CSS variables on Content : the content origin is `var(--radix-popover-content-transform-origin)` for animation. shadcn-applied classes : width `w-72`, padding `p-4`, background `bg-popover`, foreground `text-popover-foreground`, border `border`, radius `rounded-md`, shadow `shadow-md`, z-index `z-50`.

## 2. Tooltip : composition

```
TooltipProvider          (REQUIRED at app root, ONCE per tree)
└── Tooltip              (root, manages open state per instance)
    ├── TooltipTrigger   (the hover/focus target)
    └── TooltipContent   (the floating label, auto-Portalled, includes <Arrow/>)
```

### TooltipProvider props

| Prop | Type | Default (shadcn) | Default (Radix) | Purpose |
|------|------|------------------|-----------------|---------|
| `delayDuration` | `number` (ms) | `0` | `700` | Time hover must persist before open. shadcn overrides to `0` for instant tooltips. |
| `skipDelayDuration` | `number` (ms) | `300` | `300` | Time to skip the delay when moving between adjacent triggers. |
| `disableHoverableContent` | `boolean` | `false` | `false` | When `true`, content does NOT receive pointer events (closes faster). |
| `delayDuration` override per-Tooltip | `number` (ms) | inherits Provider | n/a | Per-instance override. |

ALWAYS mount one `<TooltipProvider>` per render tree, at the app root. The shadcn `TooltipProvider` wrapper hard-overrides `delayDuration` to `0` if you do not pass a value; pass `delayDuration={700}` explicitly to restore the Radix default.

### Tooltip (root) props

| Prop | Type | Default | Purpose |
|------|------|---------|---------|
| `open` | `boolean` | `undefined` | Controlled. ALWAYS pair with `onOpenChange`. |
| `defaultOpen` | `boolean` | `false` | Uncontrolled initial state. |
| `onOpenChange` | `(open: boolean) => void` | `undefined` | Fires on open and close. |
| `delayDuration` | `number` (ms) | inherits Provider | Per-instance override. |
| `disableHoverableContent` | `boolean` | inherits Provider | Per-instance override. |

### TooltipTrigger props

| Prop | Type | Default | Purpose |
|------|------|---------|---------|
| `asChild` | `boolean` | `false` | Merge props onto child. ALWAYS true when wrapping a `<Button>`. |

### TooltipContent props

| Prop | Type | Default | Purpose |
|------|------|---------|---------|
| `side` | `"top"\|"right"\|"bottom"\|"left"` | `"top"` | Radix default for Tooltip is `"top"` (Popover and HoverCard default to `"bottom"`). |
| `sideOffset` | `number` | `0` | shadcn default. |
| `align` | `"start"\|"center"\|"end"` | `"center"` | |
| `alignOffset` | `number` | `0` | |
| `avoidCollisions` | `boolean` | `true` | |
| `collisionBoundary` | `Element \| Element[] \| null` | `null` | |
| `collisionPadding` | `number \| Padding` | `0` | |
| `arrowPadding` | `number` | `0` | Distance the arrow stays away from the content corner. |
| `sticky` | `"partial"\|"always"` | `"partial"` | |
| `hideWhenDetached` | `boolean` | `false` | |
| `forceMount` | `boolean` | `undefined` | |
| `onEscapeKeyDown` | `(e: KeyboardEvent) => void` | `undefined` | |
| `onPointerDownOutside` | `(e: Event) => void` | `undefined` | |
| `asChild` | `boolean` | `false` | |
| `className` | `string` | `undefined` | |

shadcn-applied classes on TooltipContent : background `bg-foreground`, text `text-background`, font-size `text-xs`, padding `px-3 py-1.5`, radius `rounded-md`, balanced wrap `text-balance`, z-index `z-50`. shadcn renders `<TooltipPrimitive.Arrow/>` inside the content automatically; arrow uses `bg-foreground fill-foreground`.

## 3. HoverCard : composition

```
HoverCard                (root, manages open state + delays)
├── HoverCardTrigger     (the hover/focus target)
└── HoverCardContent     (the floating panel, auto-Portalled)
```

### HoverCard (root) props

| Prop | Type | Default | Purpose |
|------|------|---------|---------|
| `open` | `boolean` | `undefined` | Controlled open state. |
| `defaultOpen` | `boolean` | `false` | Uncontrolled initial state. |
| `onOpenChange` | `(open: boolean) => void` | `undefined` | Fires on open and close. |
| `openDelay` | `number` (ms) | `700` | Time hover must persist before open. |
| `closeDelay` | `number` (ms) | `300` | Grace period before close after pointer leaves. |

### HoverCardTrigger props

| Prop | Type | Default | Purpose |
|------|------|---------|---------|
| `asChild` | `boolean` | `false` | Merge props onto child. ALWAYS true when wrapping `<a>` or `<button>`. |

The Trigger MUST be a focusable element. HoverCard opens on `focus` of the trigger in addition to `pointerEnter`; if the trigger is a non-focusable div, keyboard users cannot open the content.

### HoverCardContent props

| Prop | Type | Default | Purpose |
|------|------|---------|---------|
| `side` | `"top"\|"right"\|"bottom"\|"left"` | `"bottom"` | |
| `sideOffset` | `number` | `4` | shadcn default. |
| `align` | `"start"\|"center"\|"end"` | `"center"` | shadcn default. |
| `alignOffset` | `number` | `0` | |
| `avoidCollisions` | `boolean` | `true` | |
| `collisionBoundary` | `Element \| Element[] \| null` | `null` | |
| `collisionPadding` | `number \| Padding` | `0` | |
| `sticky` | `"partial"\|"always"` | `"partial"` | |
| `hideWhenDetached` | `boolean` | `false` | |
| `forceMount` | `boolean` | `undefined` | |
| `onEscapeKeyDown` | `(e: KeyboardEvent) => void` | `undefined` | |
| `onPointerDownOutside` | `(e: Event) => void` | `undefined` | |
| `onFocusOutside` | `(e: Event) => void` | `undefined` | |
| `onInteractOutside` | `(e: Event) => void` | `undefined` | |
| `asChild` | `boolean` | `false` | |
| `className` | `string` | `undefined` | |

shadcn-applied classes on HoverCardContent : width `w-64`, padding `p-4`, background `bg-popover`, foreground `text-popover-foreground`, border `border`, radius `rounded-md`, shadow `shadow-md`, z-index `z-50`. Content origin animates from `var(--radix-hover-card-content-transform-origin)`.

## 4. Cross-primitive comparison cheatsheet

| Capability | Popover | Tooltip | HoverCard |
|------------|---------|---------|-----------|
| Provider required | NO | YES (root) | NO |
| Default trigger | click | hover + focus | hover + focus |
| Default delay (ms) open | 0 | 0 (shadcn) / 700 (Radix) | 700 |
| Default delay (ms) close | 0 | 0 | 300 |
| `sideOffset` default | 4 | 0 | 4 |
| `side` default | bottom | top | bottom |
| Built-in Arrow | NO | YES | NO |
| Auto Portal | YES | YES | YES |
| Focus moves into content on open | YES | NO | NO |
| Restores focus to trigger on close | YES | NO | NO |
| `modal` prop | YES | NO | NO |
| Touch behaviour | tap | hidden | tap |
| Interactive content allowed | YES | NO | YES |
| Provider override per-instance | n/a | YES via `delayDuration` prop | n/a (delays are root props) |
| Exported helpers | Trigger, Content, Anchor, Header, Title, Description | Provider, Trigger, Content | Trigger, Content |

## 5. Controlled vs uncontrolled

Pattern is identical for all three roots :

| Mode | Required props |
|------|----------------|
| Uncontrolled | `defaultOpen` (optional, defaults to `false`) only |
| Controlled | BOTH `open` AND `onOpenChange` |

Anti-pattern : `<Popover open={true}>` without `onOpenChange`. The popover is permanently open; nothing can close it. Same trap on Tooltip and HoverCard. See `references/anti-patterns.md` §3.
