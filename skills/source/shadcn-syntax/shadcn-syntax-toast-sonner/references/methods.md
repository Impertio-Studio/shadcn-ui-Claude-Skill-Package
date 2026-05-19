# shadcn-syntax-toast-sonner : Methods Reference

Full signatures for the `<Toaster />` wrapper and every `toast.*` method. All types derived from the upstream `sonner` package (verified against `https://sonner.emilkowal.ski` + `https://github.com/emilkowalski/sonner`) and the shadcn `apps/v4/registry/new-york-v4/ui/sonner.tsx` wrapper.

## Imports

```ts
// Toaster : shadcn wrapper (USE THIS, not the bare sonner one)
import { Toaster } from "@/components/ui/sonner"

// Imperative API : direct from the sonner package
import { toast } from "sonner"

// Type imports (optional)
import type { ToasterProps, ExternalToast } from "sonner"
```

ALWAYS import `Toaster` from `"@/components/ui/sonner"` (the shadcn wrapper with theme integration and Lucide icons). ALWAYS import `toast` from `"sonner"` (the bare imperative API). NEVER expect a `useToast()` hook ; it was removed.

## `<Toaster />` component signature

```ts
function Toaster(props: ToasterProps): JSX.Element
```

The shadcn wrapper accepts ALL of sonner's `ToasterProps` and spreads them after its own defaults. Reference table :

| Prop | Type | Default (shadcn wrapper) | Description |
|------|------|--------------------------|-------------|
| `position` | `"top-left" \| "top-center" \| "top-right" \| "bottom-left" \| "bottom-center" \| "bottom-right"` | `"bottom-right"` | Viewport corner where the stack anchors |
| `duration` | `number` | `4000` (ms) | Global default per-toast lifetime. Pass `Infinity` to make every toast sticky by default. Per-toast overrides via the `duration` option |
| `expand` | `boolean` | `true` | When `true`, hovering the stack expands every toast ; when `false`, only the top toast is visible |
| `richColors` | `boolean` | `false` | When `true`, the four semantic variants (`success` / `error` / `warning` / `info`) render with semantic background colors instead of the neutral popover background |
| `closeButton` | `boolean` | `false` | When `true`, every toast shows a small X close button on hover |
| `visibleToasts` | `number` | `3` | Maximum simultaneously visible toasts ; overflow is queued |
| `theme` | `"light" \| "dark" \| "system"` | wired from `next-themes` `useTheme()` | The shadcn wrapper forwards this from `useTheme()` automatically. Do NOT pass `theme` manually unless you have stripped `next-themes` |
| `className` | `string` | `"toaster group"` | Root element class ; the shadcn wrapper sets a default. Pass your own to extend (NOT replace) |
| `style` | `React.CSSProperties` | bridges `--normal-bg`, `--normal-text`, `--normal-border`, `--border-radius` to shadcn theme vars | The shadcn wrapper sets four CSS variables ; passing your own `style` MERGES |
| `gap` | `number` | `14` (px) | Vertical gap between stacked toasts |
| `offset` | `number \| string` | `32` (px) | Distance from the chosen viewport corner |
| `dir` | `"ltr" \| "rtl" \| "auto"` | `"auto"` | Document direction. Respects `<html dir="rtl">` automatically with `"auto"` |
| `hotkey` | `string[]` | `["altKey", "KeyT"]` | Keyboard combo that focuses the toast region for a11y |
| `invert` | `boolean` | `false` | Invert the toast theme relative to the page theme (dark toasts on light page or vice versa) |
| `toastOptions` | `Partial<ExternalToast>` | `undefined` | Default per-toast options applied to every toast (description, className, duration, etc.) |
| `loadingIcon` / `successIcon` / `errorIcon` / `warningIcon` / `infoIcon` | `ReactNode` | shadcn wrapper sets Lucide icons via the `icons` prop | The shadcn wrapper uses the `icons` prop ; if you also pass individual `*Icon` props they take precedence |
| `icons` | `{ success?, info?, warning?, error?, loading? }` | shadcn wrapper sets all 5 to Lucide | Bulk icon override. Spread will merge ; the shadcn wrapper sets a baseline. Pass your own to override individual entries |
| `pauseWhenPageIsHidden` | `boolean` | `false` | Pause the duration countdown while the tab is backgrounded |
| `containerAriaLabel` | `string` | `"Notifications"` | Accessible name for the toast region |

ALWAYS mount exactly ONE `<Toaster />` per app. NEVER mount it twice ; each instance maintains its own queue.

## The `toast` function : default + variants

```ts
toast(message: string | ReactNode, options?: ExternalToast): string | number
toast.success(message, options?): string | number
toast.error(message, options?): string | number
toast.warning(message, options?): string | number
toast.info(message, options?): string | number
toast.loading(message, options?): string | number
toast.message(message, options?): string | number       // alias for default neutral
```

All variants share the same signature and return type. The return value is the toast `id` (number by default, string if you supplied one via `options.id`). Use the id with `toast.dismiss(id)` or by passing it back as `options.id` on a later call to UPDATE the same toast in place.

Selection rule :

- `toast.success` : completion confirmation
- `toast.error` : failure that the user must notice
- `toast.warning` : soft caution
- `toast.info` : neutral informational notice with a colored info accent
- `toast.loading` : spinner-fronted toast for a long-running operation (manually pair with a later `.success` / `.error` on the same id)
- `toast.message` / plain `toast()` : neutral, no icon, no semantic color

NEVER call `toast()` from a server component, server action, or RSC render path. The function reaches into a client-side store ; calling it from the server is undefined and will throw or no-op depending on bundler.

## `toast.promise` signature

```ts
toast.promise<T>(
  promiseOrFn: Promise<T> | (() => Promise<T>),
  options: {
    loading: string | ReactNode | ((promise: Promise<T>) => ReactNode)
    success: string | ReactNode | ((data: T) => string | ReactNode)
    error:   string | ReactNode | ((error: unknown) => string | ReactNode)
    description?: string | ((data: T) => string)
    finally?: () => void | Promise<void>
    duration?: number
    position?: ToasterProps["position"]
    id?: string | number
  }
): { unwrap(): Promise<T> }
```

- `loading` MUST be present ; it is what shows while the promise is pending.
- `success` MUST be present ; renders when the promise resolves. If a function, it receives the resolved value `T`.
- `error` MUST be present ; renders when the promise rejects. If a function, it receives the thrown value.
- `finally` runs after settle (success or error), useful for cleanup.
- Returns an object with `.unwrap()` that re-throws on rejection if you also need to handle the error in your own code path.

ALWAYS pass all three states. NEVER omit `error` ; if the promise rejects without an `error` handler, the loading toast stays visible indefinitely.

## `toast.custom` signature

```ts
toast.custom(
  jsxFactory: (id: number | string) => ReactNode,
  options?: ExternalToast
): string | number
```

`jsxFactory` receives the toast id so the rendered JSX can wire its own dismiss button via `toast.dismiss(id)`. Use this for compound toasts (avatars, progress bars, action rows) that do not fit the message + description model.

## `toast.dismiss` signature

```ts
toast.dismiss(id?: string | number): void
```

- Pass an `id` to dismiss a specific toast.
- Call without arguments to dismiss EVERY visible toast (and clear the queue).

ALWAYS keep the id from your `toast()` call if you plan to dismiss programmatically.

## `ExternalToast` option fields

The options object passed as the second argument to every `toast.*` method.

| Field | Type | Use |
|-------|------|-----|
| `id` | `string \| number` | Stable id ; if the toast with this id already exists, the call UPDATES it in place (text, icon, type) rather than appending. Critical for the manual loading-pattern : `const id = toast.loading("X"); toast.success("Done", { id })` |
| `description` | `string \| ReactNode` | Secondary line under the main message |
| `duration` | `number` | Override the Toaster's default. Pass `Infinity` for a sticky toast |
| `action` | `{ label: string \| ReactNode, onClick: (event) => void } \| ReactNode` | Primary action button on the right edge. Object form is preferred for keyboard a11y |
| `actionButtonStyle` | `React.CSSProperties` | Style override for the primary action button |
| `cancel` | `{ label, onClick } \| ReactNode` | Secondary cancel-style button rendered to the left of `action` |
| `cancelButtonStyle` | `React.CSSProperties` | Style override for the cancel button |
| `onDismiss` | `(t) => void` | Fires when the user dismisses (swipe, X click, close button) |
| `onAutoClose` | `(t) => void` | Fires when the `duration` timer elapses |
| `position` | `ToasterProps["position"]` | Per-toast position override |
| `dismissible` | `boolean` | Default `true`. Set `false` to prevent the user from dismissing ; still respects `duration` |
| `icon` | `ReactNode` | Override the per-variant icon for this toast |
| `closeButton` | `boolean` | Force the X close button on or off for this toast, overriding the Toaster default |
| `important` | `boolean` | When `true`, sonner uses `role="alert"` instead of `role="status"` so screen readers announce immediately |
| `className` | `string` | Per-toast class override |
| `style` | `React.CSSProperties` | Per-toast inline style |
| `descriptionClassName` | `string` | Class on the description element |
| `unstyled` | `boolean` | Strip sonner's default styling ; you supply everything via `className` / `style` |
| `classNames` | `{ toast?, title?, description?, actionButton?, cancelButton?, closeButton?, error?, success?, warning?, info?, loading? }` | Fine-grained class overrides per inner element + per variant |
| `richColors` | `boolean` | Per-toast override of the Toaster's `richColors` default |

ALWAYS pass `id` when you intend to update a toast later. ALWAYS pass `dismissible: false` for toasts the user must NOT swipe away (e.g., a "saving..." that must hold until completion).

## Return type discipline

```ts
const id: string | number = toast.success("Saved")
// later:
toast.dismiss(id)
```

The return is `string` if you supplied `options.id` as a string, else `number`. Treat it as opaque ; the only valid uses are passing it to `toast.dismiss(id)` or as `options.id` on a follow-up `toast.*` call.

## TypeScript discipline

```ts
import type { ToasterProps, ExternalToast } from "sonner"

// Custom hook that wraps domain-specific toasts :
function useDomainToasts() {
  return {
    saved: (description?: string) =>
      toast.success("Saved", { description } satisfies ExternalToast),
    failed: (description?: string) =>
      toast.error("Failed", { description } satisfies ExternalToast),
  }
}
```

Use `satisfies ExternalToast` to keep type-safety on the options object without widening the inferred type.

## Sources

- https://ui.shadcn.com/docs/components/radix/sonner (shadcn Toaster wrapper + installation)
- https://sonner.emilkowal.ski (full `toast.*` API + ToasterProps + ExternalToast)
- https://github.com/emilkowalski/sonner (upstream source ; type definitions, position values, theme contract)
- https://github.com/shadcn-ui/ui (apps/v4/registry/new-york-v4/ui/sonner.tsx ; wrapper defaults, icon overrides, CSS-variable bridge)

Verified 2026-05-19.
