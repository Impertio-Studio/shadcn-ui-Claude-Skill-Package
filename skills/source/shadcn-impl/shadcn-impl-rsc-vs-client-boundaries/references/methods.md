# Methods : RSC Boundary Reference

Verified against https://ui.shadcn.com/docs/components-json and the
shadcn evergreen-2026 component set. Last verified : 2026-05-19.

## 1. `components.json` : `rsc` Field

The `rsc` field in `components.json` controls what the shadcn CLI emits
when it runs `add`. It is set ONCE at `init` time and rarely changed.

```jsonc
{
  "$schema": "https://ui.shadcn.com/schema.json",
  "style": "new-york",
  "rsc": true,
  "tsx": true,
  "tailwind": {
    "config": "",
    "css": "app/globals.css",
    "baseColor": "neutral",
    "cssVariables": true,
    "prefix": ""
  },
  "aliases": {
    "components": "@/components",
    "utils": "@/lib/utils",
    "ui": "@/components/ui",
    "lib": "@/lib",
    "hooks": "@/hooks"
  },
  "iconLibrary": "lucide"
}
```

### Behaviour matrix

| `rsc` value | What the CLI does                                                                                       | When to choose                                                 |
|-------------|---------------------------------------------------------------------------------------------------------|----------------------------------------------------------------|
| `true`      | Adds `"use client"` ONLY to components that need it (Radix-wrapped, hook-using, browser-API). RSC-safe components ship without the directive. | App Router (Next.js 14+ / 15.x).                                |
| `false`     | Adds `"use client"` to EVERY generated component file, regardless of whether it needs it.               | Pages Router, Vite + React, Astro client island, Remix, TanStack Start (any non-RSC environment). |

### Verification command

```bash
# After "shadcn add dialog" with rsc:true expect this directive :
head -1 components/ui/dialog.tsx        # -> "use client"
head -1 components/ui/card.tsx          # -> import * as React from "react"   (no directive)

# After "shadcn add dialog" with rsc:false expect this directive :
head -1 components/ui/dialog.tsx        # -> "use client"
head -1 components/ui/card.tsx          # -> "use client"
```

### Mutation rule

NEVER hand-edit the directive after `add`. If the directive is missing
or wrong, re-run `shadcn add <name> --overwrite` (use `--diff` first
per `shadcn-impl-component-install`).

If you flip the `rsc` flag in an existing project, re-run `shadcn add`
for every component you want to regenerate. Existing files are not
retroactively re-emitted.

---

## 2. The `"use client"` Directive : Placement Rules

Rule | Detail
---|---
Placement | First non-comment, non-whitespace line of the file.
Quoting   | `"use client"` or `'use client'` (double or single quotes). NEVER backticks (template literal).
Scope     | File-level. Applies to every export in the file.
Granularity | NEVER per-function or per-component within a file.
Bundler signal | Recognised by Turbopack, webpack via `next/dist`, and Vite (`@vitejs/plugin-react-swc`).
Behaviour | The file and every module it imports get bundled for the client. Components imported via dynamic import still respect their own directive (a client file dynamically importing a server file is a build error).

```tsx
"use client"                                       // <-- first line, double-quoted

import * as React from "react"
import { Dialog } from "@/components/ui/dialog"
```

---

## 3. Per-Primitive RSC Compatibility Matrix

The matrix below lists every primitive in the evergreen-2026 catalog,
classifies it as RSC-safe or client-required, and explains why.

Legend :
- `RSC-safe` : no `"use client"`. Renders to HTML on the server, ships zero JavaScript for this component.
- `client` : `"use client"` mandatory. Wraps a Radix primitive, uses React hooks, or accesses browser APIs.
- `mixed` : the primitive has both a server-friendly subcomponent set (HTML containers) and a client subcomponent set (interactive trigger). In practice you import the whole module and it carries `"use client"`.

| # | Primitive            | Classification | Why                                                          |
|---|----------------------|----------------|--------------------------------------------------------------|
| 1 | Accordion            | client         | Radix Accordion, uses `useState` for open state.             |
| 2 | Alert                | RSC-safe       | Pure HTML + Tailwind, no state.                              |
| 3 | AlertDialog          | client         | Radix AlertDialog, focus trap + portal.                      |
| 4 | AspectRatio          | RSC-safe       | Pure CSS aspect-ratio wrapper.                               |
| 5 | Avatar               | RSC-safe       | Pure HTML + Tailwind. Image-load fallback uses no React hooks in the evergreen build. |
| 6 | Badge                | RSC-safe       | Pure HTML span + cva variants.                               |
| 7 | Breadcrumb           | RSC-safe       | Pure HTML nav + Tailwind. (Slash separators are CSS.)        |
| 8 | Button               | RSC-safe       | Pure HTML button + cva variants. Event handlers are added by the consumer file, which must then carry `"use client"`. |
| 9 | Calendar             | client         | Wraps `react-day-picker`, uses `useState` + `Intl.DateTimeFormat`. |
| 10 | Card                | RSC-safe       | Pure HTML div family + Tailwind.                             |
| 11 | Carousel            | client         | Wraps `embla-carousel-react`, uses refs + effect for the slider engine. |
| 12 | Chart               | client         | Wraps Recharts. Recharts depends on browser DOM measurement. |
| 13 | Checkbox            | client         | Radix Checkbox, indeterminate state via ref.                 |
| 14 | Collapsible         | client         | Radix Collapsible, `useState` for open state.                |
| 15 | Combobox            | client         | Composition Popover + Command ; both are client.             |
| 16 | Command             | client         | Wraps `cmdk`, uses internal `useState` + `useReducer`.       |
| 17 | ContextMenu         | client         | Radix ContextMenu, pointer-event handlers + portal.          |
| 18 | Data Table recipe   | client         | Wraps `@tanstack/react-table`, table-state lives in hooks.   |
| 19 | DatePicker recipe   | client         | Composition Popover + Calendar + Input ; client.             |
| 20 | Dialog              | client         | Radix Dialog, focus trap + portal + `useState`.              |
| 21 | Direction           | RSC-safe       | Radix DirectionProvider has no runtime state ; emits a CSS variable. The wrapper may still carry `"use client"` because it is imported by client trees ; mark it client to be safe in next-themes setups. |
| 22 | Drawer              | client         | Wraps `vaul`, gesture handling + portal.                     |
| 23 | DropdownMenu        | client         | Radix DropdownMenu, portal + focus trap.                     |
| 24 | Empty               | RSC-safe       | Pure HTML container with cva variants.                       |
| 25 | Field               | RSC-safe       | Pure HTML form-label composition. (Note : when nested INSIDE Form, the parent Form is client, so the rendered file is client.) |
| 26 | Form (react-hook-form) | client      | Imports `useForm` + `Controller`.                            |
| 27 | HoverCard           | client         | Radix HoverCard, pointer events + portal.                    |
| 28 | Input               | RSC-safe       | Pure HTML input + Tailwind. (Wrapping form is client.)       |
| 29 | InputGroup          | RSC-safe       | Pure HTML flex wrapper.                                      |
| 30 | InputOTP            | client         | Wraps `input-otp`, uses `useState` + keyboard handlers.      |
| 31 | Item                | RSC-safe       | Pure HTML list item primitive.                               |
| 32 | Kbd                 | RSC-safe       | Pure HTML `<kbd>` + Tailwind.                                |
| 33 | Label               | RSC-safe       | Radix Label runtime is HTML-only ; no state. The evergreen build emits it without the directive. |
| 34 | Menubar             | client         | Radix Menubar, keyboard navigation + portal.                 |
| 35 | Native Select       | RSC-safe       | Native HTML `<select>`. Wrap in a client file ONLY if you bind `onChange`. |
| 36 | NavigationMenu      | client         | Radix NavigationMenu, hover + focus state.                   |
| 37 | Pagination          | RSC-safe       | Pure HTML nav + links. Bind `onClick` only inside a client component. |
| 38 | Popover             | client         | Radix Popover, portal + focus trap.                          |
| 39 | Progress            | client         | Radix Progress, animated indicator uses transitions tied to state. |
| 40 | RadioGroup          | client         | Radix RadioGroup, roving tabindex.                           |
| 41 | Resizable           | client         | Wraps `react-resizable-panels`, mouse + touch events.        |
| 42 | ScrollArea          | client         | Radix ScrollArea, ResizeObserver.                            |
| 43 | Select              | client         | Radix Select, portal + keyboard nav.                         |
| 44 | Separator           | RSC-safe       | Radix Separator, pure HTML.                                  |
| 45 | Sheet               | client         | Extends Dialog ; same client requirements.                   |
| 46 | Sidebar             | client         | Uses `useSidebar()` hook + `useContext` + responsive logic.  |
| 47 | Skeleton            | RSC-safe       | Pure CSS animation. No hooks.                                |
| 48 | Slider              | client         | Radix Slider, pointer events.                                |
| 49 | Sonner (Toaster)    | client         | The `<Toaster />` mount uses `useState` and listens to a global event bus. The `toast()` function is callable from anywhere but the mount file must be client. |
| 50 | Spinner             | RSC-safe       | Pure CSS animation.                                          |
| 51 | Switch              | client         | Radix Switch, `useState` for pressed state.                  |
| 52 | Table primitives    | RSC-safe       | Pure HTML table elements.                                    |
| 53 | Tabs                | client         | Radix Tabs, focus management + `useState`.                   |
| 54 | Textarea            | RSC-safe       | Pure HTML textarea.                                          |
| 55 | Toggle              | client         | Radix Toggle, `useState` for pressed state.                  |
| 56 | ToggleGroup         | client         | Radix ToggleGroup, roving tabindex.                          |
| 57 | Tooltip             | client         | Radix Tooltip, pointer + focus events.                       |
| 58 | Typography          | RSC-safe       | Pure HTML + Tailwind. (Documentation page, not a component.) |
| 59 | InputGroup          | RSC-safe       | Pure HTML flex wrapper. (Duplicate row removed in tooling.)  |
| 60 | NativeSelect        | RSC-safe       | See row 35.                                                  |

### Quick mental model

```
Anything that wraps a Radix package -> client
Anything that wraps a third-party state library
  (cmdk, embla, vaul, react-day-picker, react-resizable-panels,
   react-hook-form, sonner, @tanstack/react-table) -> client
Anything that is pure HTML + Tailwind + cva variants -> RSC-safe
Anything you add an onClick to inside a parent file -> the PARENT is client
```

---

## 4. Provider Wrappers : Mandatory Client Files

These libraries provide React context and MUST be wrapped in a thin
client file before being imported into `app/layout.tsx`.

| Library                       | Reason                                       | Wrapper filename                    |
|-------------------------------|----------------------------------------------|-------------------------------------|
| `next-themes`                 | `useTheme()` hook + class mutation on `<html>` | `components/theme-provider.tsx`     |
| `@tanstack/react-query`       | `QueryClient` instance + `QueryClientProvider` context | `components/providers/query-provider.tsx` |
| `jotai`                       | `Provider` with atom store ; uses `useStore` hook | `components/providers/jotai-provider.tsx` |
| `zustand` (with `Provider`)   | Optional ; only when using `createStore` per-tree | `components/providers/zustand-provider.tsx` |
| `react-aria` `OverlayProvider`| Browser focus management                    | `components/providers/overlay-provider.tsx` |
| `nuqs` `NuqsAdapter`          | Reads/writes URL search params on the client | `components/providers/nuqs-adapter.tsx` |

Every wrapper follows the same shape :

```tsx
"use client"
import { ProviderFromLib } from "lib-name"

export function MyProvider({ children, ...props }) {
  return <ProviderFromLib {...props}>{children}</ProviderFromLib>
}
```

ALWAYS make the wrapper a one-line passthrough. NEVER mix unrelated
client logic into a Provider wrapper ; it becomes a magnet for hooks
that don't belong.

---

## 5. Boundary Crossing Rules

| Direction              | Allowed                              | Disallowed                                              |
|------------------------|--------------------------------------|---------------------------------------------------------|
| Server -> Client props | Serialisable data (string, number, boolean, plain object, plain array, Date, Map, Set, null, undefined) | Functions (unless `"use server"`), class instances, JSX with event handlers, Promises that should be awaited |
| Server -> Client `children` | Any server JSX                  | -                                                       |
| Client -> Server import | Server components imported into client files become CLIENT components silently. AVOID this pattern ; use the `children` slot instead. | - |
| Client -> Server props | Not allowed                          | -                                                       |
| Server Action -> Client | Returned as a callable from a server module (`"use server"`) | -                                                       |

---

## 6. Verification Recipe

```bash
# 1. Check that components.json declares rsc:true (App Router projects)
jq '.rsc' components.json                    # -> true

# 2. List components that SHOULD carry the directive
grep -L '^"use client"' components/ui/{dialog,sheet,drawer,popover,tooltip,select,form,sonner,sidebar,tabs,accordion,carousel,calendar,combobox,command}.tsx 2>/dev/null

# Expected output : (empty). Any filename printed is MISSING the directive.

# 3. List components that should NOT carry the directive (false positives = over-clientified)
grep -l '^"use client"' components/ui/{card,badge,button,alert,skeleton,separator,avatar,aspect-ratio,table,label,input,textarea,kbd}.tsx 2>/dev/null

# Expected output : (empty). Any filename printed is an over-clientified RSC-safe primitive.

# 4. Check that layout.tsx is NOT marked client
head -1 app/layout.tsx                       # -> import ...   NOT "use client"
```

---

## 7. References

- https://ui.shadcn.com/docs/components-json : the `rsc` field.
- https://ui.shadcn.com/docs/dark-mode/next : ThemeProvider wrapper pattern.
- https://nextjs.org/docs/app/building-your-application/rendering/composition-patterns : server/client composition rules.
- https://react.dev/reference/rsc/use-client : the directive contract.
