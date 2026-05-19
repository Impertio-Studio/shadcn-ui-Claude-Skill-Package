# InputOTP : working examples

Every example begins with `"use client"` because `InputOTP` and `InputOTPSlot` rely on React context and hooks. In a Next.js App Router project, place these components in a client file or import them into a client boundary.

## Example 1 : 6-digit numeric OTP (canonical pattern)

The most common case : a 6-digit code split into two groups of three with a separator. Numeric only, SMS autofill enabled.

```tsx
"use client"

import {
  InputOTP,
  InputOTPGroup,
  InputOTPSlot,
  InputOTPSeparator,
} from "@/components/ui/input-otp"
import { REGEXP_ONLY_DIGITS } from "input-otp"

export function VerificationCodeInput() {
  return (
    <InputOTP
      maxLength={6}
      pattern={REGEXP_ONLY_DIGITS}
      autoComplete="one-time-code"
    >
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

What this gives you : six fixed-width slots, numeric-only input, SMS autofill on iOS and Android, paste of the full 6-digit code in one operation, animated caret on the active slot, accessible to screen readers because the underlying input is a real text field with the right autocomplete hint.

## Example 2 : 4-digit PIN with a single group, no separator

A short PIN entry, kept as one visual group.

```tsx
"use client"

import {
  InputOTP,
  InputOTPGroup,
  InputOTPSlot,
} from "@/components/ui/input-otp"
import { REGEXP_ONLY_DIGITS } from "input-otp"

export function PinInput() {
  return (
    <InputOTP maxLength={4} pattern={REGEXP_ONLY_DIGITS}>
      <InputOTPGroup>
        <InputOTPSlot index={0} />
        <InputOTPSlot index={1} />
        <InputOTPSlot index={2} />
        <InputOTPSlot index={3} />
      </InputOTPGroup>
    </InputOTP>
  )
}
```

Note the absence of `InputOTPSeparator` ; a 4-digit PIN reads more naturally as one block.

## Example 3 : 8-character alphanumeric backup code

A longer code with letters and digits, split into two groups of four.

```tsx
"use client"

import {
  InputOTP,
  InputOTPGroup,
  InputOTPSlot,
  InputOTPSeparator,
} from "@/components/ui/input-otp"
import { REGEXP_ONLY_DIGITS_AND_CHARS } from "input-otp"

export function BackupCodeInput() {
  return (
    <InputOTP
      maxLength={8}
      pattern={REGEXP_ONLY_DIGITS_AND_CHARS}
      inputMode="text"
    >
      <InputOTPGroup>
        <InputOTPSlot index={0} />
        <InputOTPSlot index={1} />
        <InputOTPSlot index={2} />
        <InputOTPSlot index={3} />
      </InputOTPGroup>
      <InputOTPSeparator />
      <InputOTPGroup>
        <InputOTPSlot index={4} />
        <InputOTPSlot index={5} />
        <InputOTPSlot index={6} />
        <InputOTPSlot index={7} />
      </InputOTPGroup>
    </InputOTP>
  )
}
```

Note `inputMode="text"` overrides the default `"numeric"` so the on-screen keyboard offers letters. The `REGEXP_ONLY_DIGITS_AND_CHARS` constant still constrains accepted characters.

## Example 4 : controlled OTP with onChange and onComplete (auto-submit)

A controlled input that submits the moment the user finishes typing the last digit, without requiring a separate submit button.

```tsx
"use client"

import * as React from "react"
import {
  InputOTP,
  InputOTPGroup,
  InputOTPSlot,
  InputOTPSeparator,
} from "@/components/ui/input-otp"
import { REGEXP_ONLY_DIGITS } from "input-otp"

export function AutoSubmitOTP({
  onVerify,
}: {
  onVerify: (code: string) => Promise<void>
}) {
  const [value, setValue] = React.useState("")
  const [isSubmitting, setIsSubmitting] = React.useState(false)
  const [error, setError] = React.useState<string | null>(null)

  async function handleComplete(code: string) {
    setIsSubmitting(true)
    setError(null)
    try {
      await onVerify(code)
    } catch (err) {
      setError("Invalid code. Try again.")
      setValue("")
    } finally {
      setIsSubmitting(false)
    }
  }

  return (
    <div className="flex flex-col gap-2">
      <InputOTP
        maxLength={6}
        pattern={REGEXP_ONLY_DIGITS}
        autoComplete="one-time-code"
        value={value}
        onChange={setValue}
        onComplete={handleComplete}
        disabled={isSubmitting}
        aria-invalid={error !== null}
      >
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
      {error && (
        <p className="text-sm text-destructive" role="alert">
          {error}
        </p>
      )}
    </div>
  )
}
```

What this gives you : the verify request fires automatically the moment the user enters the sixth digit. While the request is in flight, the input is disabled and dimmed. On failure, the value clears, the input goes back to enabled, and `aria-invalid` applies destructive styling.

## Example 5 : OTP inside react-hook-form with Controller

The canonical integration with `react-hook-form` when you want the OTP value to flow through your form state.

```tsx
"use client"

import { Controller, useForm } from "react-hook-form"
import { Button } from "@/components/ui/button"
import {
  InputOTP,
  InputOTPGroup,
  InputOTPSlot,
  InputOTPSeparator,
} from "@/components/ui/input-otp"
import { REGEXP_ONLY_DIGITS } from "input-otp"

type FormValues = {
  code: string
}

export function VerifyForm({
  onSubmit,
}: {
  onSubmit: (values: FormValues) => Promise<void>
}) {
  const {
    control,
    handleSubmit,
    formState: { errors, isSubmitting },
  } = useForm<FormValues>({
    defaultValues: { code: "" },
  })

  return (
    <form
      onSubmit={handleSubmit(onSubmit)}
      className="flex flex-col gap-4"
    >
      <Controller
        control={control}
        name="code"
        rules={{
          required: "Enter the 6-digit code",
          minLength: { value: 6, message: "Code must be 6 digits" },
          maxLength: { value: 6, message: "Code must be 6 digits" },
        }}
        render={({ field, fieldState }) => (
          <div className="flex flex-col gap-2">
            <InputOTP
              maxLength={6}
              pattern={REGEXP_ONLY_DIGITS}
              autoComplete="one-time-code"
              value={field.value}
              onChange={field.onChange}
              onBlur={field.onBlur}
              aria-invalid={fieldState.invalid}
              disabled={isSubmitting}
            >
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
            {fieldState.error && (
              <p className="text-sm text-destructive" role="alert">
                {fieldState.error.message}
              </p>
            )}
          </div>
        )}
      />
      <Button type="submit" disabled={isSubmitting}>
        Verify
      </Button>
    </form>
  )
}
```

Three things to notice :

1. `Controller`, not `register`. The latter cannot drive a controlled component that owns its own `value` and `onChange`.
2. `field.onBlur` is wired through so `react-hook-form` knows when the field has been touched (relevant for `touchedFields` and on-blur validation modes).
3. `aria-invalid={fieldState.invalid}` flows the validation state into the slot styling automatically via the default Tailwind classes.

## Example 6 : full MFA verify flow with a separate verify button

When you do NOT want auto-submit on `onComplete` (for example because the user might want to correct the last digit before submitting), use a controlled value and a separate button.

```tsx
"use client"

import * as React from "react"
import { Button } from "@/components/ui/button"
import {
  InputOTP,
  InputOTPGroup,
  InputOTPSlot,
  InputOTPSeparator,
} from "@/components/ui/input-otp"
import { REGEXP_ONLY_DIGITS } from "input-otp"

export function MfaChallenge({
  onVerify,
  onResend,
}: {
  onVerify: (code: string) => Promise<{ ok: boolean }>
  onResend: () => Promise<void>
}) {
  const [code, setCode] = React.useState("")
  const [status, setStatus] = React.useState<
    "idle" | "verifying" | "error"
  >("idle")
  const [resendCooldown, setResendCooldown] = React.useState(0)

  React.useEffect(() => {
    if (resendCooldown <= 0) return
    const t = setTimeout(() => setResendCooldown((s) => s - 1), 1000)
    return () => clearTimeout(t)
  }, [resendCooldown])

  async function handleVerify() {
    setStatus("verifying")
    const result = await onVerify(code)
    if (result.ok) {
      setStatus("idle")
    } else {
      setStatus("error")
      setCode("")
    }
  }

  async function handleResend() {
    await onResend()
    setResendCooldown(30)
    setCode("")
    setStatus("idle")
  }

  const isComplete = code.length === 6
  const isBusy = status === "verifying"

  return (
    <div className="flex flex-col gap-4">
      <div className="flex flex-col gap-2">
        <label className="text-sm font-medium">
          Enter the 6-digit code we sent you
        </label>
        <InputOTP
          maxLength={6}
          pattern={REGEXP_ONLY_DIGITS}
          autoComplete="one-time-code"
          value={code}
          onChange={setCode}
          disabled={isBusy}
          aria-invalid={status === "error"}
        >
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
        {status === "error" && (
          <p className="text-sm text-destructive" role="alert">
            That code did not match. Try again or resend.
          </p>
        )}
      </div>

      <div className="flex items-center gap-2">
        <Button
          onClick={handleVerify}
          disabled={!isComplete || isBusy}
        >
          {isBusy ? "Verifying..." : "Verify"}
        </Button>
        <Button
          variant="ghost"
          onClick={handleResend}
          disabled={resendCooldown > 0 || isBusy}
        >
          {resendCooldown > 0
            ? `Resend in ${resendCooldown}s`
            : "Resend code"}
        </Button>
      </div>
    </div>
  )
}
```

What this gives you : a complete MFA challenge UI with explicit verify, resend with cooldown, error display, and disabled-during-verify behaviour. The OTP input drops the value on error so the user can retry without manual clearing.
