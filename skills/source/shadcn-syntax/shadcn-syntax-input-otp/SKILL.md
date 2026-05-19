---
name: shadcn-syntax-input-otp
description: >
  Use when building a one-time-password, verification-code, MFA, two-factor,
  SMS-code, or PIN-entry input in shadcn ui evergreen-2026, when composing
  InputOTP, InputOTPGroup, InputOTPSlot, and InputOTPSeparator, when wiring
  a 6-digit numeric code, a 4-digit PIN, or an alphanumeric token, when
  enabling iOS and Android SMS autofill via autoComplete one-time-code,
  when handling paste of a full code in one operation, when constraining
  allowed characters via the pattern prop, when reacting to a fully entered
  code via onComplete, when integrating an OTP input with react-hook-form
  Controller, or when an OTP renders as a single empty input with no slots
  visible.
  Prevents the common Input OTP failures : omitting maxLength so the
  underlying OTPInput renders zero slots and the user sees an empty box,
  forgetting the index prop on InputOTPSlot so every slot reads from the
  same context entry, passing a JavaScript RegExp object to the pattern
  prop instead of a regex source string or one of the exported constants,
  trying to register InputOTP with react-hook-form register() instead of
  wrapping it in Controller, omitting autoComplete one-time-code so iOS
  and Android cannot suggest the SMS code, and dropping the use client
  directive in a Next.js App Router file so the underlying React hooks
  fail at build time.
  Covers the four exported primitives, the maxLength contract, the pattern
  prop with the three exported regex source constants (REGEXP_ONLY_DIGITS,
  REGEXP_ONLY_DIGITS_AND_CHARS, REGEXP_ONLY_CHARS), the value plus onChange
  controlled pattern, the onComplete callback, autoComplete one-time-code
  for SMS autofill, the multi-character paste behaviour that fills every
  slot from a single clipboard event, the data-active attribute that drives
  focus-ring styling, aria-invalid wiring for error states, the disabled
  prop and has-disabled container styling, the InputOTPSlot index prop
  contract, and the canonical Controller integration with react-hook-form.
  This skill covers the InputOTP primitive only. For the broader Form
  composition with FormField, FormItem, FormLabel, FormControl, and
  FormMessage, see shadcn-syntax-form. For end-to-end form validation
  recipes with zod and onSubmit handlers, see shadcn-impl-form-validation.
  Keywords: shadcn input otp, InputOTP, InputOTPGroup, InputOTPSlot,
  InputOTPSeparator, one time password, OTP code, verification code,
  MFA input, two-factor code, 2FA code, 6-digit code input, 4-digit PIN,
  SMS code autofill, autoComplete one-time-code, REGEXP_ONLY_DIGITS,
  REGEXP_ONLY_DIGITS_AND_CHARS, REGEXP_ONLY_CHARS, onComplete callback,
  pattern prop, controlled OTP input, Controller with InputOTP,
  react-hook-form OTP, paste OTP code, alphanumeric OTP, numeric OTP,
  data-active slot, has-disabled, how do I make an OTP input,
  how to build a verification code input, otp empty slots not rendering,
  otp paste not working, otp SMS autofill not triggering.
license: MIT
compatibility: "Designed for Claude Code. Requires shadcn ui evergreen-2026."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# shadcn ui : Input OTP primitive

The `InputOTP` family wraps the `input-otp` library by guilhermerodz and renders a one-time-password input as a row of fixed-width slots that mimic individual character boxes while remaining a single accessible text input under the hood. Every claim in this skill traces to the canonical source at `apps/v4/registry/new-york-v4/ui/input-otp.tsx` in `shadcn-ui/ui`, the official docs at `https://ui.shadcn.com/docs/components/radix/input-otp`, and the upstream library at `https://github.com/guilhermerodz/input-otp`.

## Quick Reference

| Primitive | Wraps | Role |
|-----------|-------|------|
| `InputOTP` | `OTPInput` from `input-otp` | Root container, owns `maxLength`, `value`, `onChange`, `pattern`, `onComplete`, `autoComplete`, `disabled` |
| `InputOTPGroup` | plain `div` | Visual group of consecutive slots ; combine multiple groups with `InputOTPSeparator` between them |
| `InputOTPSlot` | plain `div` reading from `OTPInputContext` | One visible character box, requires `index: number` matching its position 0 to `maxLength - 1` |
| `InputOTPSeparator` | plain `div` with `role="separator"` and a `MinusIcon` | Visual divider between two `InputOTPGroup` instances |

Install : `pnpm dlx shadcn@latest add input-otp` (CLI copies `components/ui/input-otp.tsx` and installs the `input-otp` peer dependency).

## When to use this skill

Use `InputOTP` for any flow where the user must enter a short, fixed-length code character-by-character with visible per-character boxes :

- Email verification codes (typically 6 digits)
- SMS-delivered second-factor codes (typically 6 digits, SMS autofill expected)
- Authenticator-app TOTP codes (6 digits)
- Backup-code entry (often 8 to 10 alphanumeric)
- PIN entry on payment or unlock flows (typically 4 to 6 digits)

Do NOT use `InputOTP` for free-text fields, passwords, or any input where the length is unknown ; use `Input` from `shadcn-syntax-form` or the underlying `<input>` element instead.

## The four primitives and how they compose

```tsx
"use client"

import {
  InputOTP,
  InputOTPGroup,
  InputOTPSlot,
  InputOTPSeparator,
} from "@/components/ui/input-otp"
import { REGEXP_ONLY_DIGITS } from "input-otp"

export function SixDigitCode() {
  return (
    <InputOTP maxLength={6} pattern={REGEXP_ONLY_DIGITS}>
      <InputOTPGroup>
        <InputOTPSlot index={0} />
        <InputOTPSlot index={1} />
        <InputOTPSlot index={2} />
      </InputOTPGroup>
      <InputOTPSeparator />
      <InputOTPGroup>
        <InputOTPSlot index={3} />
        <InputOTPSlot index={4} />
        <InputOTPSlot index={5} />
      </InputOTPGroup>
    </InputOTP>
  )
}
```

Three rules govern composition :

1. The number of `InputOTPSlot` instances rendered inside `InputOTP` MUST equal `maxLength`. If you render fewer slots, characters typed beyond the last visible slot still enter the underlying input but the user cannot see them. If you render more, the extra slots read `undefined` from context and render empty forever.
2. Each `InputOTPSlot` MUST receive a unique `index` prop matching its position. `index={0}` is the leftmost slot, `index={maxLength - 1}` is the rightmost. The slot uses this index to read `{ char, hasFakeCaret, isActive }` from `OTPInputContext`.
3. `InputOTPSeparator` belongs BETWEEN two `InputOTPGroup` elements, not inside one. It is a sibling of groups, not a sibling of slots.

## The `'use client'` directive

The shadcn `input-otp.tsx` file begins with `"use client"` because `InputOTPSlot` calls `React.useContext(OTPInputContext)` and `OTPInput` itself uses hooks for caret animation. In a Next.js App Router project the file is already a client component ; any consuming page or layout that imports it stays a Server Component, which is fine. If you re-export `InputOTP` from a barrel file in a Server Component, the directive on the original file still applies. Do NOT remove the directive when customizing the file.

## `maxLength` : the slot contract

`maxLength` is the single most important prop. It tells the underlying `OTPInput` how many characters the user may type and it implicitly defines how many `InputOTPSlot` instances you MUST render. Common values :

- `4` : short PIN
- `6` : standard email or SMS verification code
- `8` : backup code
- `10` : long backup code or recovery code

If you omit `maxLength`, `OTPInput` renders nothing usable and the user sees an empty box with no slots ; this is the single most common cause of "the OTP component is broken" bug reports.

## `pattern` : restricting allowed characters

`pattern` accepts a regex source string (not a JS `RegExp` object) that defines which characters the user may type. The library exports three pre-built constants you should prefer over hand-written patterns :

| Constant | Source pattern | Use for |
|----------|----------------|---------|
| `REGEXP_ONLY_DIGITS` | `^\d+$` | Numeric codes (SMS, email, TOTP, PIN) |
| `REGEXP_ONLY_DIGITS_AND_CHARS` | `^[a-zA-Z0-9]+$` | Alphanumeric backup codes, license keys |
| `REGEXP_ONLY_CHARS` | `^[a-zA-Z]+$` | Letter-only codes (rare) |

Import from the library : `import { REGEXP_ONLY_DIGITS } from "input-otp"`. The pattern is applied PER KEYSTROKE : a character that does not match is silently rejected. The pattern also applies to pasted strings : a paste containing any rejected character is rejected as a whole.

If you must hand-write a pattern, supply the regex source as a string :

```tsx
<InputOTP maxLength={6} pattern="^[0-9]+$">...</InputOTP>
```

Do NOT pass a `RegExp` object literal. `pattern={/^[0-9]+$/}` will not behave as you expect because the library re-constructs the regex from a string internally.

## `value` and `onChange` : controlled pattern

The canonical controlled pattern :

```tsx
const [value, setValue] = React.useState("")

return (
  <InputOTP maxLength={6} value={value} onChange={setValue}>
    ...
  </InputOTP>
)
```

`value` is always a string with length between `0` and `maxLength`. `onChange` fires with the new string on every keystroke. The component is also valid uncontrolled (drop `value` and `onChange`) and exposes the value through the underlying input's standard form submission.

## `onComplete` : reacting to a finished code

`onComplete` fires exactly once when the user types or pastes the final character that brings the value's length to `maxLength`. Use it to auto-submit the form or fire the verification request without requiring a separate submit click :

```tsx
<InputOTP
  maxLength={6}
  value={value}
  onChange={setValue}
  onComplete={(code) => verifyCode(code)}
>
  ...
</InputOTP>
```

`onComplete` receives the full code string. It does NOT fire on every value change ; only on the transition to `value.length === maxLength`.

## `autoComplete="one-time-code"` : SMS autofill

On iOS Safari and Android Chrome, setting `autoComplete="one-time-code"` on an input that the system detects as an OTP field unlocks two behaviours :

1. The keyboard offers a suggestion chip with the code parsed from a recently received SMS.
2. The browser may auto-fill the field without user interaction in some flows.

`OTPInput` forwards `autoComplete` to the underlying single hidden `<input>`. Without this prop, neither autofill path triggers and your users will type the code manually even when their phone received it 200 milliseconds ago.

```tsx
<InputOTP maxLength={6} autoComplete="one-time-code">
  ...
</InputOTP>
```

## Paste : a full code in one operation

`input-otp` rewrites the underlying paste handler to accept a clipboard string of up to `maxLength` characters and distribute it across slots in one operation. The user CAN paste a full 6-digit code from their clipboard onto the first slot (or any slot) and every slot fills correctly. The `pattern` constraint applies to the pasted string as a whole : a paste that contains any rejected character is rejected entirely.

No additional configuration is required. Paste works out of the box on desktop and mobile.

## `data-active` and aria-invalid : styling hooks

Two attributes drive the default Tailwind classes on `InputOTPSlot` :

- `data-active="true"` : applied to the slot that is currently focused. Drives the ring and z-index lift in the default classes.
- `aria-invalid="true"` : when you place this on `InputOTP` (or the underlying input via custom render), the destructive border and ring colours apply automatically.

```tsx
<InputOTP maxLength={6} aria-invalid={hasError}>
  ...
</InputOTP>
```

The default styles use `aria-invalid:border-destructive`, `data-[active=true]:border-ring`, and `data-[active=true]:ring-ring/50` ; customize by passing `className` to individual `InputOTPSlot` instances.

## `disabled` and `has-disabled`

`disabled` on `InputOTP` forwards to the underlying input and triggers two style hooks :

- `disabled:cursor-not-allowed` on the input itself
- `has-disabled:opacity-50` on the container (the `has-` variant requires Tailwind v3.4 or v4)

```tsx
<InputOTP maxLength={6} disabled={isVerifying}>
  ...
</InputOTP>
```

## Form integration : use Controller, not register

`InputOTP` is a controlled component that exposes `value` and `onChange`, not a raw `<input>`. The react-hook-form `register()` API cannot drive it correctly because `register` expects a native input ref and target.value flow. ALWAYS use the `Controller` component to bridge :

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
      rules={{ minLength: 6, maxLength: 6 }}
      render={({ field }) => (
        <InputOTP
          maxLength={6}
          value={field.value}
          onChange={field.onChange}
          onBlur={field.onBlur}
        >
          <InputOTPGroup>
            {[0, 1, 2, 3, 4, 5].map((i) => (
              <InputOTPSlot key={i} index={i} />
            ))}
          </InputOTPGroup>
        </InputOTP>
      )}
    />
  </form>
)
```

When wrapping `InputOTP` with the higher-level `FormField` from `shadcn-syntax-form`, the same Controller wiring happens for you ; you pass `render={({ field }) => <InputOTP {...field}>...</InputOTP>}` and `FormField` injects Controller behind the scenes.

## What this skill does NOT cover

- The `FormField` + `FormItem` + `FormControl` + `FormMessage` composition that adds label and error wiring around `InputOTP`. See `shadcn-syntax-form`.
- End-to-end form validation with zod resolvers, async server validation, and submit handlers. See `shadcn-impl-form-validation`.
- The full underlying `input-otp` `render` prop (used only when fully replacing the shadcn slot rendering). The shadcn registry already wraps `OTPInput` with `data-slot` markup, so the `render` prop is not exposed at the `InputOTP` layer.

## References

- Primitive signatures : `references/methods.md`
- Working examples (6-digit numeric, 4-digit PIN with separator, alphanumeric, controlled with onComplete, Controller integration, full MFA verify flow) : `references/examples.md`
- Anti-patterns and how to fix them : `references/anti-patterns.md`

## Companion skills

- `shadcn-syntax-form` : the Form composition layer (FormField, FormItem, FormLabel, FormControl, FormMessage) that wraps `InputOTP` with react-hook-form context and label or error wiring.
- `shadcn-impl-form-validation` : end-to-end recipes for form validation with zod, submit handlers, server actions, and async verification flows that consume the code from `InputOTP`.
