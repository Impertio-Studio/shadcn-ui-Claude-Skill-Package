---
name: shadcn-agents-form-validator
description: >
  Use when validating a react-hook-form + zod + shadcn Form integration in
  a code review, a draft AI-generated form, or a bug report ("my validation
  does not trigger", "the error message never shows", "the Select does not
  bind", "submit fires but values are undefined", "the field reads as
  uncontrolled then controlled"), when auditing whether a zod schema and
  a shadcn Form composition agree on every field name, when verifying
  Controller-vs-register correctness for custom controls (Select,
  Checkbox, RadioGroup, Switch, DatePicker, Combobox), when confirming
  every FormField has a FormMessage so zod errors are visible to the user,
  when checking that defaultValues covers every schema key (no
  undefined-to-string controlled / uncontrolled warning), when verifying
  the submit handler is wrapped in form.handleSubmit, when checking that
  zodResolver is actually passed to useForm, when ensuring no nested
  FormProvider / shadcn Form context is duplicated, and when emitting
  a structured pass / fail verdict over a form code block.
  Prevents the silent failure modes that make react-hook-form + zod look
  broken : a FormField name prop that does not match the zod schema path
  exactly (typos silently un-validate the field and the input value never
  reaches the schema), a missing FormMessage so zod errors fire but the
  user never sees them, register used on a Radix Select / Checkbox /
  RadioGroup / Switch where Controller is required (the control never
  binds and submit reads undefined), partial defaultValues that leaves
  keys undefined (react warns "input is changing from uncontrolled to
  controlled" the first time the user types), onSubmit passed directly
  to the form element instead of through form.handleSubmit (validation
  is bypassed and onSubmit receives the raw event instead of validated
  values), zodResolver omitted from useForm (the schema is declared but
  never enforced), and a duplicated FormProvider that competes with the
  shadcn Form context (two aria-describedby chains and two formStates).
  Covers the eight-point form validator checklist (schema-to-Form name
  mapping / Controller-vs-register correctness / FormMessage presence per
  field / name prop exact match against zod path / defaultValues
  completeness against schema keys / handleSubmit wrap on submit handler
  / zodResolver presence in useForm options / no nested FormProvider) and
  the validator workflow (parse the form code, classify per FormField,
  flag mismatches, missing FormMessage nodes, wrong-path name props,
  silent-fail Controller-vs-register choices, undefined defaultValues
  keys, then emit a structured verdict that names the failing rule and
  the fix).
  Keywords: form validator, validate react-hook-form, validate zod form,
  FormField name match, FormMessage missing, Controller vs register
  check, validate shadcn form, form code review, how do I verify a form,
  form integration audit, react-hook-form zod check, zodResolver missing,
  defaultValues incomplete, handleSubmit not wrapped, FormProvider
  duplicated, field name mismatch, name prop typo, silent form failure,
  form does not submit, validation does not trigger, error message not
  showing, Select not binding, Checkbox not binding, uncontrolled to
  controlled warning, shadcn form audit, agent form validator.
license: MIT
compatibility: "Designed for Claude Code. Requires shadcn ui evergreen-2026."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# shadcn ui : Form Validator (Agent / Validator)

This skill is an AGENT skill. It does NOT teach the Form API. It validates a react-hook-form + zod + shadcn Form composition against an eight-point checklist and emits a structured verdict.

The shadcn Form layer is a thin wrapper over `FormProvider` from react-hook-form. Most "my form is broken" reports trace to ONE of eight integration mistakes. ALWAYS run the eight-point check in order. NEVER skip a point because "it looks fine" : six of the eight failure modes are silent (no console error, no thrown exception, only wrong behaviour at submit).

## Quick Reference : The Validator Workflow

Given a form code block C (a `useForm` call plus the JSX tree under `<Form>`), execute :

1. **Parse the zod schema** to extract the canonical set of field paths (top-level keys, dotted paths for nested objects, indexed paths for arrays).
2. **Parse the `useForm<...>` call** to confirm `resolver: zodResolver(schema)` is present and `defaultValues` covers every schema path.
3. **Walk every `<FormField name="..." />`** in the JSX tree. For each, record : the `name` value, whether the rendered control uses `register` or `Controller` (via the `field` render-prop), and whether a `<FormMessage />` is present in the FormItem.
4. **Cross-check** the FormField name set against the zod schema path set. Flag every mismatch (typo, missing field, orphan field).
5. **Classify each rendered control** (Input, Textarea, Select, Checkbox, RadioGroup, Switch, Combobox, DatePicker) and verify the correct binding mechanism (register-class vs Controller-class).
6. **Verify the submit handler** is `form.handleSubmit(onValid)` and not a raw `onSubmit`.
7. **Verify the JSX tree has exactly one `<Form>`** (one FormProvider) wrapping the form. Flag duplicated FormProvider or nested `<Form>`.
8. **Emit verdict** with pass / fail per checkpoint and the canonical fix per failure.

ALWAYS report ALL eight checkpoints, even when most pass. NEVER stop at the first failure : multiple silent failures often coexist and the user needs the full picture.

## The Eight-Point Checklist

| # | Check | Pass criterion | Failure signal |
|---|-------|----------------|----------------|
| 1 | Schema-to-Form name mapping | Every zod schema path has a matching `<FormField name="path" />`. Every FormField name matches a schema path. | Field exists in schema but no FormField (uncovered field). FormField exists but no schema path (orphan / typo). |
| 2 | Controller-vs-register correctness | Custom controls (Select, Checkbox, RadioGroup, Switch, Combobox, DatePicker, custom inputs) use the `field` render-prop (Controller). Native `<input>` / `<textarea>` may use `field` (preferred) or `register("name")`. | A Select / Checkbox / Switch / RadioGroup / Combobox is bound via `register("name")` instead of `{...field}` from the FormField render-prop. |
| 3 | FormMessage presence per field | Every `<FormItem>` inside a `<FormField>` contains exactly one `<FormMessage />`. | FormItem has no FormMessage : zod error fires but the user sees nothing. |
| 4 | Name prop exact match | Every FormField `name` prop is character-for-character equal to a schema path. | Typos (`emai`, `usename`), wrong-case (`Email` vs `email`), or wrong dotted path (`user.email` when schema is `userEmail`). |
| 5 | defaultValues completeness | `useForm({ defaultValues: {...} })` lists every schema key with a non-undefined initial value (empty string for strings, `false` for booleans, `[]` for arrays, etc). | A schema key is missing from defaultValues OR set to `undefined`. React fires "input is changing from uncontrolled to controlled" on first keystroke. |
| 6 | handleSubmit wrap | `<form onSubmit={form.handleSubmit(onValid)}>`. Optional second arg : `form.handleSubmit(onValid, onInvalid)`. | `<form onSubmit={onSubmit}>` directly. Validation is bypassed ; onSubmit receives the raw event, not the validated values. |
| 7 | zodResolver in useForm | `useForm({ resolver: zodResolver(formSchema), ... })`. Import : `import { zodResolver } from "@hookform/resolvers/zod"`. | resolver omitted, OR resolver set to a different resolver while the project schema is zod, OR `zodResolver` imported but never wired into useForm. |
| 8 | No nested FormProvider | Exactly one `<Form>` wraps the form. No nested `<Form>` inside it. No external `<FormProvider>` wrapping a `<Form>`. | A second `<Form>` or `<FormProvider>` inside the tree. Two aria-describedby chains. Two competing formStates. |

ALWAYS apply checks 1, 3, 4, 5 by direct string comparison against the zod schema source. NEVER infer "this is probably the same field" : silent un-validation hides exactly here.

## How to Classify a Control (Controller vs register)

The `field` object that FormField passes via its render-prop contains `{ name, value, onChange, onBlur, ref }`. This is the canonical Controller binding. Use it for ALL controls, native or custom.

`register("name")` is a separate, parallel binding API. It only works for native `<input>` / `<textarea>` / `<select>` because it relies on the native onChange firing on the underlying DOM node.

| Control | Binding | Why |
|---------|---------|-----|
| `<Input {...field} />` | Controller (field) | Native input ; field spreads `value` + `onChange` + `onBlur` + `ref`. Preferred so FormControl can wire aria automatically. |
| `<Textarea {...field} />` | Controller (field) | Same as Input. |
| `<Select value={field.value} onValueChange={field.onChange} />` | Controller (field) | Radix Select fires `onValueChange`, NOT `onChange`. `register` cannot wire `onValueChange`. |
| `<Checkbox checked={field.value} onCheckedChange={field.onChange} />` | Controller (field) | Radix Checkbox fires `onCheckedChange`, NOT `onChange`. |
| `<Switch checked={field.value} onCheckedChange={field.onChange} />` | Controller (field) | Same as Checkbox. |
| `<RadioGroup value={field.value} onValueChange={field.onChange}>` | Controller (field) | Radix RadioGroup fires `onValueChange`. |
| `<Combobox value={field.value} onChange={field.onChange} />` | Controller (field) | Composed primitive : Popover + Command. No DOM-level onChange to register against. |
| `<DatePicker selected={field.value} onSelect={field.onChange} />` | Controller (field) | Calendar fires `onSelect`. |
| Native `<input {...register("name")} />` | register (allowed) | Works, but loses the auto-wired FormControl aria. Prefer field. |

ALWAYS bind Radix / shadcn custom controls through the `field` render-prop. NEVER call `register("name")` on a Radix control : the control fires `onValueChange` / `onCheckedChange` / `onSelect`, not `onChange`, so register never receives an update and the form value stays at the defaultValue forever.

## Verdict Output Format

When validating a form code block C, emit :

```
=== Form Validator Verdict ===

Schema paths      : <list of zod schema paths>
FormField names   : <list of name props found in JSX>
defaultValues keys: <list of keys with their initial values>

[1] Schema-to-Form name mapping : PASS / FAIL
    <if FAIL, list uncovered schema paths and orphan FormField names>
[2] Controller-vs-register      : PASS / FAIL
    <if FAIL, list each control + control type + current binding + required binding>
[3] FormMessage presence        : PASS / FAIL
    <if FAIL, list FormField name(s) missing FormMessage>
[4] Name prop exact match       : PASS / FAIL
    <if FAIL, list each (name prop, nearest schema path) with the typo highlighted>
[5] defaultValues completeness  : PASS / FAIL
    <if FAIL, list missing keys + recommended initial value per zod type>
[6] handleSubmit wrap           : PASS / FAIL
    <if FAIL, show current onSubmit and required wrap>
[7] zodResolver in useForm      : PASS / FAIL
    <if FAIL, show how resolver is currently set and the required line>
[8] No nested FormProvider      : PASS / FAIL
    <if FAIL, name the duplicated wrapper>

Overall: PASS / FAIL
Forward-pointer: shadcn-syntax-form (canonical API) | shadcn-impl-form-validation (workflow) | shadcn-errors-form-state (deeper error patterns)
```

ALWAYS emit all eight rows. ALWAYS name the forward-pointer skill. NEVER emit a free-form review : the structured verdict is the contract.

## Canonical Reference Pattern (target shape)

This is the shape every validated form should converge on. Every check above traces back to one rule in this pattern :

```tsx
import { useForm } from "react-hook-form"
import { zodResolver } from "@hookform/resolvers/zod"
import { z } from "zod"

const formSchema = z.object({
  username: z.string().min(2),
  role: z.enum(["admin", "user"]),
  agree: z.boolean(),
})

type FormValues = z.infer<typeof formSchema>

export function ProfileForm() {
  const form = useForm<FormValues>({
    resolver: zodResolver(formSchema),
    defaultValues: { username: "", role: "user", agree: false },
  })

  const onSubmit = (values: FormValues) => { /* ... */ }

  return (
    <Form {...form}>
      <form onSubmit={form.handleSubmit(onSubmit)}>
        <FormField
          control={form.control}
          name="username"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Username</FormLabel>
              <FormControl><Input {...field} /></FormControl>
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
            <FormItem>
              <FormControl>
                <Checkbox checked={field.value} onCheckedChange={field.onChange} />
              </FormControl>
              <FormLabel>I agree</FormLabel>
              <FormMessage />
            </FormItem>
          )}
        />
      </form>
    </Form>
  )
}
```

ALWAYS measure a validated form against this canonical shape. NEVER accept a deviation without naming which checkpoint it violates and why the deviation is acceptable.

## Heuristics : Quick Smoke Tests

These are NOT a substitute for the eight-point check, but flag the most common failures fast :

- **"My Select does not submit"** : 99% of the time, the Select uses `{...field}` spread instead of `value={field.value} onValueChange={field.onChange}`. The spread tries to wire `onChange` which Radix Select does not fire.
- **"Validation does not run"** : check 7 first (zodResolver). Then check 6 (handleSubmit).
- **"Errors never show"** : check 3 (FormMessage). The validation runs ; the UI never renders the error.
- **"My field is silently un-validated"** : check 4 (name prop typo). The schema path is `email` and the FormField is `name="emai"` : zod validates `email`, RHF tracks `emai`, the two never meet.
- **"Uncontrolled to controlled warning"** : check 5 (defaultValues). Add the missing key with `""` / `false` / `[]`.

## Companion Skills

- `shadcn-syntax-form` (B3) : authoritative shadcn Form API : Form / FormField / FormItem / FormLabel / FormControl / FormDescription / FormMessage with zodResolver and the Controller-vs-register decision.
- `shadcn-impl-form-validation` (B9) : end-to-end form validation workflow : zod schema design, async refinements, server-action submit, error rendering, edit-existing-record `values` reset pattern.
- `shadcn-errors-form-state` (B12) : deeper form-state error patterns : Controller-vs-register silent failures, watch-driven re-render storms, defaultValues vs values trap, async validation pending state.
- `shadcn-syntax-field` : new 2026 Field primitive for library-agnostic forms (TanStack Form or no form-state library). Out of scope for this validator : Field has a different a11y wiring.
- `shadcn-agents-component-selector` : sibling agent skill that picks Form vs Field at the family level before this validator runs on a Form composition.

## References

- `references/methods.md` : full eight-point checklist with pass / fail code-pattern matchers (regex-style snippets, AST hints) per checkpoint.
- `references/examples.md` : a broken form with 5+ violations and the per-violation fix, a canonical correct form, an edge case with custom FormField composition.
- `references/anti-patterns.md` : six canonical silent-failure anti-patterns with WRONG / RIGHT / WHY and the corresponding checkpoint.
