# InputOTP : primitive signatures

All signatures match the canonical source at `apps/v4/registry/new-york-v4/ui/input-otp.tsx` in `shadcn-ui/ui` and the upstream `input-otp` types at `https://github.com/guilhermerodz/input-otp`.

## InputOTP

```ts
function InputOTP(
  props: React.ComponentProps<typeof OTPInput> & {
    containerClassName?: string
  }
): JSX.Element
```

Wraps `OTPInput` from `input-otp`. Forwards every `OTPInput` prop and adds explicit typing for `containerClassName` (which `OTPInput` already accepts but the shadcn typings expose for clarity).

Effective prop surface inherited from `OTPInput` :

| Prop | Type | Default | Role |
|------|------|---------|------|
| `maxLength` | `number` | required | Number of characters and number of slots |
| `value` | `string` | uncontrolled | Current value, between `""` and `maxLength` characters |
| `onChange` | `(value: string) => void` | none | Fires on every keystroke and on paste |
| `onComplete` | `(value: string) => void` | none | Fires once when `value.length` reaches `maxLength` |
| `pattern` | `string` | none | Regex source pattern, applied per keystroke and to pastes |
| `autoComplete` | `string` | none | Pass `"one-time-code"` for iOS and Android SMS autofill |
| `disabled` | `boolean` | `false` | Disables the underlying input and dims the container via `has-disabled:opacity-50` |
| `placeholder` | `string` | none | Character shown inside empty slots when using a custom render |
| `inputMode` | `"numeric" \| "text" \| "decimal" \| "tel" \| "search" \| "email" \| "url"` | `"numeric"` | Mobile keyboard hint forwarded to the underlying input |
| `textAlign` | `"left" \| "center" \| "right"` | `"left"` | Caret-position behaviour on focus |
| `pasteTransformer` | `(pasted: string) => string` | none | Transform a clipboard string before slot distribution |
| `pushPasswordManagerStrategy` | `"increase-width" \| "none"` | `"increase-width"` | How to position the hidden password-manager affordance |
| `noScriptCSSFallback` | `string \| null` | library default | CSS injected for no-JavaScript users |
| `containerClassName` | `string` | none | Tailwind classes on the row container (the `div` that holds the slots) |
| `className` | `string` | none | Tailwind classes on the underlying transparent `<input>` element |
| `aria-invalid` | `boolean` | `false` | Drives `aria-invalid:border-destructive` styling on each slot |
| `name` | `string` | none | Standard form field name for native form submission |
| `autoFocus` | `boolean` | `false` | Focus the input on mount |
| `onBlur` | `(e: FocusEvent) => void` | none | Forwarded to the underlying input ; used by `Controller.field.onBlur` |

## InputOTPGroup

```ts
function InputOTPGroup(
  props: React.ComponentProps<"div">
): JSX.Element
```

Plain `div` wrapper with `flex items-center` and `data-slot="input-otp-group"`. Accepts every native `div` attribute including `className`, `children`, `id`, `aria-*`, and event handlers. Contains one or more `InputOTPSlot` children.

## InputOTPSlot

```ts
function InputOTPSlot(
  props: React.ComponentProps<"div"> & { index: number }
): JSX.Element
```

The `index` prop is REQUIRED and MUST match the slot's position in reading order, starting from `0`. The slot reads `{ char, hasFakeCaret, isActive }` from `OTPInputContext` at the given index and renders :

- The current character (from `char`)
- An animated caret div when `hasFakeCaret` is true (only on the active slot when empty)
- The `data-active="true"` attribute when `isActive` is true (focus styling hook)

Default classes : `relative flex h-9 w-9 items-center justify-center border-y border-r border-input text-sm shadow-xs transition-all outline-none first:rounded-l-md first:border-l last:rounded-r-md aria-invalid:border-destructive data-[active=true]:z-10 data-[active=true]:border-ring data-[active=true]:ring-[3px] data-[active=true]:ring-ring/50`.

## InputOTPSeparator

```ts
function InputOTPSeparator(
  props: React.ComponentProps<"div">
): JSX.Element
```

A `div` with `role="separator"` and `data-slot="input-otp-separator"` that renders a `MinusIcon` from `lucide-react`. Place between two `InputOTPGroup` siblings to render a visual divider. Accepts every native `div` attribute ; replace the child icon by passing `children` if you prefer a different separator glyph.

## Exported regex constants

Re-exported from `input-otp` for use as the `pattern` prop value :

```ts
import {
  REGEXP_ONLY_DIGITS,
  REGEXP_ONLY_DIGITS_AND_CHARS,
  REGEXP_ONLY_CHARS,
} from "input-otp"
```

| Constant | Source string | Matches |
|----------|---------------|---------|
| `REGEXP_ONLY_DIGITS` | `^\d+$` | `0` through `9` only |
| `REGEXP_ONLY_DIGITS_AND_CHARS` | `^[a-zA-Z0-9]+$` | Alphanumeric (case-insensitive) |
| `REGEXP_ONLY_CHARS` | `^[a-zA-Z]+$` | Letters only (case-insensitive) |

All three are STRINGS, not `RegExp` objects. Pass them directly to `pattern` :

```tsx
<InputOTP maxLength={6} pattern={REGEXP_ONLY_DIGITS}>...</InputOTP>
```

## Custom pattern signatures

If none of the exported constants fit, supply your own regex source string :

```tsx
// Hex characters only
<InputOTP maxLength={8} pattern="^[0-9a-fA-F]+$">...</InputOTP>

// Uppercase letters and digits only
<InputOTP maxLength={10} pattern="^[A-Z0-9]+$">...</InputOTP>

// Base32 characters (no 0, 1, 8, 9 ; common in TOTP backup codes)
<InputOTP maxLength={16} pattern="^[A-Z2-7]+$">...</InputOTP>
```

The pattern is evaluated per character on keystroke and applied to the entire pasted string on paste. A rejected character or paste is silently dropped ; the `value` does not advance.

## OTPInputContext (advanced)

```ts
import { OTPInputContext } from "input-otp"

type OTPInputContextValue = {
  slots: Array<{
    char: string | null
    placeholderChar: string | null
    isActive: boolean
    hasFakeCaret: boolean
  }>
  isFocused: boolean
  isHovering: boolean
}
```

`InputOTPSlot` consumes this context via `React.useContext(OTPInputContext)`. You generally do NOT need to touch the context directly ; only reach for it when building a fully custom slot component that lives outside the four shadcn primitives.
