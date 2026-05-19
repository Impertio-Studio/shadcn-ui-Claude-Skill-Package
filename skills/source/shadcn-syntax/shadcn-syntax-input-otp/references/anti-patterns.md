# InputOTP : anti-patterns

Every anti-pattern below is sourced from real failure modes documented in the `guilhermerodz/input-otp` issue tracker, the shadcn ui registry source, or the official docs. Each entry shows the broken pattern, why it breaks, and the fix.

## 1. Missing `maxLength` : no slots rendered, empty box

### Broken

```tsx
"use client"

import {
  InputOTP,
  InputOTPGroup,
  InputOTPSlot,
} from "@/components/ui/input-otp"

export function BrokenOTP() {
  return (
    <InputOTP>
      <InputOTPGroup>
        <InputOTPSlot index={0} />
        <InputOTPSlot index={1} />
        <InputOTPSlot index={2} />
        <InputOTPSlot index={3} />
        <InputOTPSlot index={4} />
        <InputOTPSlot index={5} />
      </InputOTPGroup>
    </InputOTP>
  )
}
```

### Why it breaks

`OTPInput` requires `maxLength` to know how many characters the user may type and to initialize its internal `slots` array. Without `maxLength`, the underlying input has no length cap, the `slots` array is empty, and every `InputOTPSlot` reads `undefined` from context. You see six empty boxes that never fill, and pasting or typing puts characters into the hidden input without visible feedback.

### Fix

ALWAYS pass `maxLength` and make it equal to the number of `InputOTPSlot` elements you render.

```tsx
<InputOTP maxLength={6}>
  <InputOTPGroup>
    <InputOTPSlot index={0} />
    <InputOTPSlot index={1} />
    <InputOTPSlot index={2} />
    <InputOTPSlot index={3} />
    <InputOTPSlot index={4} />
    <InputOTPSlot index={5} />
  </InputOTPGroup>
</InputOTP>
```

## 2. Missing or duplicate `index` prop on `InputOTPSlot`

### Broken

```tsx
<InputOTP maxLength={4}>
  <InputOTPGroup>
    <InputOTPSlot />
    <InputOTPSlot />
    <InputOTPSlot />
    <InputOTPSlot />
  </InputOTPGroup>
</InputOTP>
```

or :

```tsx
<InputOTP maxLength={4}>
  <InputOTPGroup>
    <InputOTPSlot index={0} />
    <InputOTPSlot index={0} />
    <InputOTPSlot index={0} />
    <InputOTPSlot index={0} />
  </InputOTPGroup>
</InputOTP>
```

### Why it breaks

`InputOTPSlot` uses its `index` prop to read `{ char, hasFakeCaret, isActive }` from `OTPInputContext.slots[index]`. The first version (`index` omitted) is a TypeScript error AND a runtime failure : `slots[undefined]` yields `undefined`, every slot renders empty, the active-slot ring never moves. The second version (every slot has `index={0}`) renders four slots that all show the first character and all light up simultaneously when any slot is focused.

### Fix

Each slot's `index` MUST match its position in reading order, starting from `0`. For long codes, generate slots with a map :

```tsx
<InputOTP maxLength={6}>
  <InputOTPGroup>
    {Array.from({ length: 6 }, (_, i) => (
      <InputOTPSlot key={i} index={i} />
    ))}
  </InputOTPGroup>
</InputOTP>
```

## 3. Passing a `RegExp` object to `pattern` instead of a source string

### Broken

```tsx
<InputOTP maxLength={6} pattern={/^[0-9]+$/}>
  ...
</InputOTP>
```

### Why it breaks

The `OTPInput` library expects `pattern` to be a regex source string and internally reconstructs the regex with `new RegExp(pattern)`. When you pass a `RegExp` object, JavaScript coerces it to a string via `toString()`, which yields `"/^[0-9]+$/"` (with the leading and trailing slashes). The library then tries to construct `new RegExp("/^[0-9]+$/")`, which produces a regex that literally matches the slash character at start and end ; nothing the user types will pass validation, so every keystroke is silently rejected and the input appears frozen.

### Fix

Pass either one of the exported constants or a plain regex source string :

```tsx
import { REGEXP_ONLY_DIGITS } from "input-otp"

// Best : use the exported constant
<InputOTP maxLength={6} pattern={REGEXP_ONLY_DIGITS}>...</InputOTP>

// Acceptable : a hand-written source string
<InputOTP maxLength={6} pattern="^[0-9]+$">...</InputOTP>

// WRONG : a RegExp object literal
<InputOTP maxLength={6} pattern={/^[0-9]+$/}>...</InputOTP>
```

## 4. Trying to drive `InputOTP` with react-hook-form `register()`

### Broken

```tsx
import { useForm } from "react-hook-form"

const { register, handleSubmit } = useForm<{ code: string }>()

return (
  <form onSubmit={handleSubmit(onSubmit)}>
    <InputOTP maxLength={6} {...register("code")}>
      <InputOTPGroup>
        <InputOTPSlot index={0} />
        <InputOTPSlot index={1} />
        <InputOTPSlot index={2} />
        <InputOTPSlot index={3} />
        <InputOTPSlot index={4} />
        <InputOTPSlot index={5} />
      </InputOTPGroup>
    </InputOTP>
  </form>
)
```

### Why it breaks

`register("code")` returns `{ onChange, onBlur, name, ref }` designed for native `<input>` elements that emit `change` events with `event.target.value`. `InputOTP` (via `OTPInput`) calls `onChange` with the raw VALUE STRING, not a synthetic event, so the `register`-provided onChange tries to read `event.target.value` on a plain string and crashes (or silently stores the string itself as the form value with the wrong shape). The `ref` spread also fails because `InputOTP` does not forward a ref to a native input.

### Fix

ALWAYS use `Controller` to bridge `InputOTP` into react-hook-form :

```tsx
import { Controller, useForm } from "react-hook-form"

const { control, handleSubmit } = useForm<{ code: string }>({
  defaultValues: { code: "" },
})

return (
  <form onSubmit={handleSubmit(onSubmit)}>
    <Controller
      control={control}
      name="code"
      render={({ field }) => (
        <InputOTP
          maxLength={6}
          value={field.value}
          onChange={field.onChange}
          onBlur={field.onBlur}
        >
          <InputOTPGroup>
            {Array.from({ length: 6 }, (_, i) => (
              <InputOTPSlot key={i} index={i} />
            ))}
          </InputOTPGroup>
        </InputOTP>
      )}
    />
  </form>
)
```

The same rule applies when using `FormField` from `shadcn-syntax-form` : `FormField` uses `Controller` under the hood, and the `render={({ field }) => ...}` prop gives you the same `field` shape.

## 5. Omitting `autoComplete="one-time-code"` : SMS autofill never triggers

### Broken

```tsx
<InputOTP maxLength={6}>
  <InputOTPGroup>
    <InputOTPSlot index={0} />
    <InputOTPSlot index={1} />
    <InputOTPSlot index={2} />
    <InputOTPSlot index={3} />
    <InputOTPSlot index={4} />
    <InputOTPSlot index={5} />
  </InputOTPGroup>
</InputOTP>
```

### Why it breaks

iOS Safari and Android Chrome only offer the SMS-code suggestion chip (and only allow programmatic autofill) when the underlying input carries `autocomplete="one-time-code"`. Without it, your verification flow forces the user to read the code from their notification, switch back to the browser, and type six digits manually, even though the platform could have surfaced a one-tap suggestion. This is one of the most common failure modes in shipped MFA flows.

### Fix

ALWAYS set `autoComplete="one-time-code"` on `InputOTP` for any flow where the code arrives via SMS or push notification :

```tsx
<InputOTP maxLength={6} autoComplete="one-time-code">
  ...
</InputOTP>
```

`autoComplete` flows through to the underlying single hidden input that backs all the slots. No further configuration is required.

## 6. Slot count does not match `maxLength`

### Broken

```tsx
// Six characters allowed, but only four slots rendered
<InputOTP maxLength={6}>
  <InputOTPGroup>
    <InputOTPSlot index={0} />
    <InputOTPSlot index={1} />
    <InputOTPSlot index={2} />
    <InputOTPSlot index={3} />
  </InputOTPGroup>
</InputOTP>
```

or :

```tsx
// Four characters allowed, but six slots rendered
<InputOTP maxLength={4}>
  <InputOTPGroup>
    <InputOTPSlot index={0} />
    <InputOTPSlot index={1} />
    <InputOTPSlot index={2} />
    <InputOTPSlot index={3} />
    <InputOTPSlot index={4} />
    <InputOTPSlot index={5} />
  </InputOTPGroup>
</InputOTP>
```

### Why it breaks

First variant : the user can type six characters but only sees the first four ; characters five and six are invisible and `onComplete` fires after six, surprising the user who only sees four full slots. Second variant : slots four and five read `OTPInputContext.slots[4]` and `slots[5]` which are `undefined`, so they render as permanently empty boxes and never become active.

### Fix

The number of `InputOTPSlot` instances rendered inside `InputOTP` MUST equal `maxLength`. Use a map to avoid drift :

```tsx
const LENGTH = 6

<InputOTP maxLength={LENGTH}>
  <InputOTPGroup>
    {Array.from({ length: LENGTH }, (_, i) => (
      <InputOTPSlot key={i} index={i} />
    ))}
  </InputOTPGroup>
</InputOTP>
```

## 7. Dropping the `'use client'` directive

### Broken

A consuming page that places `InputOTP` directly inside a Server Component file without re-exporting it through a client boundary :

```tsx
// app/verify/page.tsx (Server Component by default in Next.js App Router)
import { InputOTP, InputOTPSlot, InputOTPGroup } from "@/components/ui/input-otp"

export default function Page() {
  // ...
}
```

This is actually fine if `components/ui/input-otp.tsx` retains its `"use client"` directive : the directive marks every export from that module as client-only, and importing into a Server Component creates the client boundary automatically.

What breaks is this :

```tsx
// components/ui/input-otp.tsx (after customization)
// "use client"  <-- removed
import * as React from "react"
import { OTPInput, OTPInputContext } from "input-otp"
// ...
```

### Why it breaks

`InputOTPSlot` calls `React.useContext(OTPInputContext)` and `OTPInput` uses hooks for focus, caret animation, and paste handling. Without `"use client"` at the top of the file that exports these components, the Next.js App Router compiler treats the module as a Server Component and rejects the React hook calls at build time with `useContext only works in Client Components`.

### Fix

The shadcn registry source ships with `"use client"` at the top of `input-otp.tsx`. NEVER remove this directive when customizing the file. If you create a derivative file (for example a styled wrapper), put `"use client"` at the top of that file too.

```tsx
"use client"

import * as React from "react"
import { OTPInput, OTPInputContext } from "input-otp"
// ...
```
