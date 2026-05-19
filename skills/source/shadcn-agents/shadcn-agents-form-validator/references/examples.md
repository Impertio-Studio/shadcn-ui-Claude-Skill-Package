# Form Validator : Worked Examples

Three worked examples : a broken form with seven distinct violations and a per-violation fix, a canonical correct form for cross-reference, and an edge case where a custom FormField composition extends the canonical shape.

## Example 1 : The Broken Form (Seven Violations)

This is a realistic AI-generated draft that compiles, renders, and looks correct in the browser. None of the seven violations throws a runtime error. All seven cause silent wrong behaviour at submit time.

```tsx
import { useForm } from "react-hook-form"
// note : zodResolver imported but not used
import { zodResolver } from "@hookform/resolvers/zod"
import { z } from "zod"

const formSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
  role: z.enum(["admin", "user"]),
  agree: z.boolean(),
  tags: z.array(z.string()),
})

export function SignUpForm() {
  const form = useForm({
    defaultValues: {
      email: "",
      // missing : password, role, agree, tags
    },
  })

  const onSubmit = (values: any) => { console.log(values) }

  return (
    <Form {...form}>
      <FormProvider {...form}>
        <form onSubmit={onSubmit}>
          <FormField
            control={form.control}
            name="emai"
            render={({ field }) => (
              <FormItem>
                <FormLabel>Email</FormLabel>
                <FormControl><Input {...field} /></FormControl>
              </FormItem>
            )}
          />
          <FormField
            control={form.control}
            name="password"
            render={({ field }) => (
              <FormItem>
                <FormLabel>Password</FormLabel>
                <FormControl><Input type="password" {...field} /></FormControl>
                <FormMessage />
              </FormItem>
            )}
          />
          <FormField
            control={form.control}
            name="role"
            render={({ field }) => (
              <FormItem>
                <FormLabel>Role</FormLabel>
                <FormControl>
                  <Select {...register("role")}>
                    <SelectTrigger><SelectValue /></SelectTrigger>
                    <SelectContent>
                      <SelectItem value="admin">Admin</SelectItem>
                      <SelectItem value="user">User</SelectItem>
                    </SelectContent>
                  </Select>
                </FormControl>
                <FormMessage />
              </FormItem>
            )}
          />
          <FormField
            control={form.control}
            name="agree"
            render={({ field }) => (
              <FormItem>
                <Checkbox {...field} />
                <FormLabel>I agree</FormLabel>
                <FormMessage />
              </FormItem>
            )}
          />
        </form>
      </FormProvider>
    </Form>
  )
}
```

### Validator Verdict

```
=== Form Validator Verdict ===

Schema paths      : [email, password, role, agree, tags]
FormField names   : [emai, password, role, agree]
defaultValues keys: [email]

[1] Schema-to-Form name mapping : FAIL
    - uncovered schema path: tags (no FormField bound to it)
    - orphan FormField name: emai (no matching schema path : Levenshtein 1 to "email")

[2] Controller-vs-register : FAIL
    - role: Select bound via `{...register("role")}`. Radix Select fires onValueChange, not onChange. register never receives an update. Required: value={field.value} onValueChange={field.onChange}.
    - agree: Checkbox bound via `{...field}` spread. Checkbox fires onCheckedChange, not onChange. The spread tries to set onChange. Required: checked={field.value} onCheckedChange={field.onChange}.

[3] FormMessage presence : FAIL
    - missing on: emai (the FormItem has no <FormMessage />, so zod errors fire but the user sees nothing)

[4] Name prop exact match : FAIL
    - "emai" → did you mean "email" ? (Levenshtein 1, only this FormField is the orphan)

[5] defaultValues completeness : FAIL
    - missing: password (→ "")
    - missing: role (→ "user" or one of the enum members)
    - missing: agree (→ false). Checkbox renders checked={undefined} → uncontrolled-to-controlled warning on first click
    - missing: tags (→ []). Tags has no FormField, so even with defaultValues the array is unreachable through the UI

[6] handleSubmit wrap : FAIL
    - current: <form onSubmit={onSubmit}>
    - required: <form onSubmit={form.handleSubmit(onSubmit)}>
    - effect: zod validation is bypassed entirely ; onSubmit receives the raw SyntheticEvent, not the validated values

[7] zodResolver : FAIL
    - zodResolver imported but not wired into useForm options
    - current: useForm({ defaultValues: { ... } })
    - required: useForm({ resolver: zodResolver(formSchema), defaultValues: { ... } })

[8] No nested FormProvider : FAIL
    - <Form {...form}> wraps a <FormProvider {...form}> wraps the form. The shadcn <Form> IS already a FormProvider. The duplication creates two competing react-hook-form contexts.

Overall: FAIL (8/8 fail)
Forward-pointer: shadcn-syntax-form (canonical API), shadcn-impl-form-validation (workflow), shadcn-errors-form-state (Controller-vs-register deep dive)
```

### Per-Violation Fix

1. Remove the `<FormProvider {...form}>` wrapper. The shadcn `<Form>` is already a FormProvider.
2. Wire `resolver: zodResolver(formSchema)` into `useForm`.
3. Wrap `onSubmit` in `form.handleSubmit` : `onSubmit={form.handleSubmit(onSubmit)}`.
4. Fix the typo : `name="emai"` → `name="email"`.
5. Add `<FormMessage />` to the email FormItem.
6. Change the Select binding to `value={field.value} onValueChange={field.onChange}`.
7. Change the Checkbox binding to `checked={field.value} onCheckedChange={field.onChange}`.
8. Add missing defaultValues : `password: ""`, `role: "user"`, `agree: false`, `tags: []`.
9. Add the missing FormField for `tags` (or remove `tags` from the schema if it is genuinely not user-input).

## Example 2 : The Canonical Correct Form

This is the cross-reference shape every fix in Example 1 converges on.

```tsx
import { useForm } from "react-hook-form"
import { zodResolver } from "@hookform/resolvers/zod"
import { z } from "zod"
import { Form, FormField, FormItem, FormLabel, FormControl, FormMessage } from "@/components/ui/form"
import { Input } from "@/components/ui/input"
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from "@/components/ui/select"
import { Checkbox } from "@/components/ui/checkbox"
import { Button } from "@/components/ui/button"

const formSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
  role: z.enum(["admin", "user"]),
  agree: z.boolean(),
})

type FormValues = z.infer<typeof formSchema>

export function SignUpForm() {
  const form = useForm<FormValues>({
    resolver: zodResolver(formSchema),
    defaultValues: {
      email: "",
      password: "",
      role: "user",
      agree: false,
    },
  })

  const onSubmit = (values: FormValues) => { console.log(values) }

  return (
    <Form {...form}>
      <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-4">
        <FormField
          control={form.control}
          name="email"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Email</FormLabel>
              <FormControl><Input type="email" {...field} /></FormControl>
              <FormMessage />
            </FormItem>
          )}
        />
        <FormField
          control={form.control}
          name="password"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Password</FormLabel>
              <FormControl><Input type="password" {...field} /></FormControl>
              <FormMessage />
            </FormItem>
          )}
        />
        <FormField
          control={form.control}
          name="role"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Role</FormLabel>
              <FormControl>
                <Select value={field.value} onValueChange={field.onChange}>
                  <SelectTrigger><SelectValue /></SelectTrigger>
                  <SelectContent>
                    <SelectItem value="admin">Admin</SelectItem>
                    <SelectItem value="user">User</SelectItem>
                  </SelectContent>
                </Select>
              </FormControl>
              <FormMessage />
            </FormItem>
          )}
        />
        <FormField
          control={form.control}
          name="agree"
          render={({ field }) => (
            <FormItem className="flex items-center gap-2">
              <FormControl>
                <Checkbox checked={field.value} onCheckedChange={field.onChange} />
              </FormControl>
              <FormLabel>I agree to the terms</FormLabel>
              <FormMessage />
            </FormItem>
          )}
        />
        <Button type="submit">Sign up</Button>
      </form>
    </Form>
  )
}
```

### Validator Verdict

```
=== Form Validator Verdict ===

Schema paths      : [email, password, role, agree]
FormField names   : [email, password, role, agree]
defaultValues keys: [email="", password="", role="user", agree=false]

[1] Schema-to-Form name mapping : PASS
[2] Controller-vs-register      : PASS
[3] FormMessage presence        : PASS (all four fields)
[4] Name prop exact match       : PASS
[5] defaultValues completeness  : PASS
[6] handleSubmit wrap           : PASS
[7] zodResolver                 : PASS
[8] No nested FormProvider      : PASS

Overall: PASS
Forward-pointer: shadcn-syntax-form (canonical API)
```

## Example 3 : Edge Case : Custom FormField Composition

Some projects extract a reusable wrapper that composes FormField + FormItem + FormLabel + FormControl + FormMessage into a single component, parametrised by the input renderer. This is a legitimate composition, and the validator must NOT flag it as a violation when the underlying contract is preserved.

```tsx
// Reusable wrapper
function TextField({ name, label, type = "text" }: { name: string; label: string; type?: string }) {
  const { control } = useFormContext<any>()
  return (
    <FormField
      control={control}
      name={name}
      render={({ field }) => (
        <FormItem>
          <FormLabel>{label}</FormLabel>
          <FormControl><Input type={type} {...field} /></FormControl>
          <FormMessage />
        </FormItem>
      )}
    />
  )
}

// Usage
const formSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
})

export function LoginForm() {
  const form = useForm<z.infer<typeof formSchema>>({
    resolver: zodResolver(formSchema),
    defaultValues: { email: "", password: "" },
  })

  return (
    <Form {...form}>
      <form onSubmit={form.handleSubmit((v) => console.log(v))}>
        <TextField name="email" label="Email" type="email" />
        <TextField name="password" label="Password" type="password" />
        <Button type="submit">Login</Button>
      </form>
    </Form>
  )
}
```

### Validator Verdict

```
=== Form Validator Verdict ===

Schema paths      : [email, password]
FormField names   : [email, password]   (resolved via TextField composition)
defaultValues keys: [email="", password=""]

[1] Schema-to-Form name mapping : PASS (via TextField composition ; the wrapper passes `name` through to FormField)
[2] Controller-vs-register      : PASS (Input + {...field} inside the wrapper)
[3] FormMessage presence        : PASS (FormMessage inside the wrapper, applies to every TextField usage)
[4] Name prop exact match       : PASS (the `name` prop of TextField forwards verbatim to FormField)
[5] defaultValues completeness  : PASS
[6] handleSubmit wrap           : PASS
[7] zodResolver                 : PASS
[8] No nested FormProvider      : PASS

Overall: PASS
Note: TextField is a legitimate composition wrapper. The validator MUST trace through one level of composition to confirm the wrapper itself contains a FormField + FormItem + FormControl + FormMessage shape and forwards `name` verbatim.
Forward-pointer: shadcn-syntax-form (canonical API)
```

### Composition Rules the Validator Must Trace

When the user uses a wrapper component instead of FormField directly :

1. Resolve the wrapper definition in the same file or the same module path. If it cannot be resolved, emit `[1] : INCONCLUSIVE, wrapper TextField could not be traced` rather than a false FAIL.
2. Verify the wrapper forwards the `name` prop unchanged to the inner FormField (no string concatenation, no template-literal mutation).
3. Verify the wrapper contains exactly one FormItem + FormControl + FormMessage subtree.
4. Verify the inner control uses the `field` render-prop correctly per the Controller-vs-register table (`{...field}` for Input/Textarea ; explicit prop mapping for Select/Checkbox/Switch/RadioGroup).
5. If the wrapper renders multiple FormFields internally (rare ; e.g., a paired first-name + last-name field), each must have its own FormMessage and its own name path. Treat them as siblings during the check.

ALWAYS trace one level. NEVER trace recursively beyond one level : the validator is not a type-checker. If the user wraps a wrapper inside a wrapper, ask the user to validate the inner wrapper independently.
