# Form Validator : Methods Reference

This is the operational reference for the eight-point form validator workflow. It defines the pass / fail criteria, the code-pattern matchers per checkpoint, and the order of operations.

## 0. Inputs

The validator receives a code block C containing :

- A zod schema (top-level `z.object({...})` or composed from sub-schemas)
- A `useForm<...>` call with the resolver and defaultValues
- A JSX tree under `<Form {...form}>` with one or more `<FormField name="..." />`

ALWAYS require all three pieces. If any is missing, ABORT with a precondition failure : "validator requires schema + useForm + Form JSX tree".

## 1. Schema-to-Form Name Mapping

**Goal** : every zod schema path has exactly one matching FormField, and every FormField has exactly one matching schema path.

**Pass criterion** :
- The set `S` of zod schema paths equals the set `F` of FormField name props.
- A "schema path" is the dotted notation for nested objects (`profile.bio.url`) and the indexed notation for arrays (`tags.0`, `items.2.id`). Top-level keys count as their bare key (`username`).

**Code-pattern matcher** :

```regex
# Extract schema paths : top-level
z\.object\(\s*\{\s*(\w+)\s*:
# Extract schema paths : nested
(\w+)\s*:\s*z\.object\(
# Extract FormField names from JSX
<FormField[^>]*\bname=["']([^"']+)["']
```

**Pass** :
```ts
const schema = z.object({ username: z.string(), email: z.string() })
// JSX
<FormField name="username" .../>
<FormField name="email" .../>
```

**Fail (uncovered)** :
```ts
const schema = z.object({ username: z.string(), email: z.string() })
<FormField name="username" .../>  // email is uncovered → submit value for email is undefined
```

**Fail (orphan)** :
```ts
const schema = z.object({ username: z.string() })
<FormField name="username" .../>
<FormField name="bio" .../>  // bio is orphan → zod schema does not include it, FormField path tracked by RHF only
```

**Output line** : `[1] Schema-to-Form name mapping : FAIL. uncovered: [email]  orphan: [bio]`

## 2. Controller-vs-register Correctness

**Goal** : custom Radix-based controls (Select, Checkbox, RadioGroup, Switch, Combobox, DatePicker) bind through the FormField `field` render-prop ; native `<input>` and `<textarea>` may use either `field` or `register("name")`.

**Pass criterion** :

| Control component | Required binding |
|-------------------|------------------|
| `<Input>`, `<Textarea>` | `{...field}` (preferred) or `{...register("name")}` (allowed) |
| `<Select>` (Radix) | `value={field.value} onValueChange={field.onChange}` |
| `<Checkbox>` | `checked={field.value} onCheckedChange={field.onChange}` |
| `<Switch>` | `checked={field.value} onCheckedChange={field.onChange}` |
| `<RadioGroup>` | `value={field.value} onValueChange={field.onChange}` |
| `<Combobox>` | `value={field.value} onChange={field.onChange}` (composed) |
| `<DatePicker>` / `<Calendar mode="single">` | `selected={field.value} onSelect={field.onChange}` |

**Code-pattern matcher** :

```regex
# Wrong : register on a Radix control
<Select[^>]*\{\s*\.\.\.register\(
<Checkbox[^>]*\{\s*\.\.\.register\(
<Switch[^>]*\{\s*\.\.\.register\(
<RadioGroup[^>]*\{\s*\.\.\.register\(

# Wrong : spreading {...field} on a Radix control (Select fires onValueChange not onChange)
<Select[^>]*\{\s*\.\.\.field\s*\}
<Checkbox[^>]*\{\s*\.\.\.field\s*\}
<Switch[^>]*\{\s*\.\.\.field\s*\}
```

**Fail** :
```tsx
<Select {...register("role")}>          // wrong : register cannot wire onValueChange
<Checkbox {...field} />                 // wrong : spread tries to set onChange, Checkbox fires onCheckedChange
```

**Pass** :
```tsx
<Select value={field.value} onValueChange={field.onChange}>...</Select>
<Checkbox checked={field.value} onCheckedChange={field.onChange} />
```

**Output line** : `[2] Controller-vs-register : FAIL. role: Select bound via register, expected value/onValueChange`

## 3. FormMessage Presence per Field

**Goal** : every `<FormField>` render-prop returns a `<FormItem>` that contains exactly one `<FormMessage />`. Without it, zod errors fire into the formState but the user never sees them.

**Pass criterion** :
- For every FormField in the tree, its `render={({ field }) => ( ... )}` output contains `<FormMessage />` somewhere inside the FormItem subtree.

**Code-pattern matcher** :

```regex
# For each FormField block, scan the render return for <FormMessage />
<FormField[^>]*name=["'](\w+)["'][\s\S]*?render=\{\s*\(\s*\{\s*field[\s\S]*?\}\s*\)\s*=>\s*\(?\s*<FormItem[\s\S]*?<FormMessage\s*/?>[\s\S]*?</FormItem>
```

If the inner match for `<FormMessage />` is absent, FAIL.

**Fail** :
```tsx
<FormField name="email" render={({ field }) => (
  <FormItem>
    <FormLabel>Email</FormLabel>
    <FormControl><Input {...field} /></FormControl>
    {/* zod will set an error here on invalid input, but no FormMessage means user sees nothing */}
  </FormItem>
)}/>
```

**Pass** :
```tsx
<FormField name="email" render={({ field }) => (
  <FormItem>
    <FormLabel>Email</FormLabel>
    <FormControl><Input {...field} /></FormControl>
    <FormMessage />
  </FormItem>
)}/>
```

**Output line** : `[3] FormMessage presence : FAIL. missing on: [email, role]`

## 4. Name Prop Exact Match

**Goal** : every FormField `name` prop is character-for-character equal to a zod schema path. Subtle typos (`emai`, `useName`, `user_email`, `userEmail` vs `user.email`) are the most common silent un-validation.

**Pass criterion** :
- For each FormField name `n`, there exists a schema path `s` with `n === s` (strict equality, case-sensitive, no whitespace, no synonym mapping).

**Code-pattern matcher** :

```python
schema_paths = extract_zod_paths(schema_source)        # exact strings
form_field_names = extract_form_field_names(jsx)       # exact strings
typos = [(n, nearest(s, n)) for n in form_field_names if n not in schema_paths]
```

The "nearest" match is computed with Levenshtein distance ≤ 2 against the schema path set to suggest the likely intended schema path.

**Fail** :
```ts
const schema = z.object({ email: z.string() })
<FormField name="emai" .../>  // typo : Levenshtein 1 from "email"
```

**Fail (case)** :
```ts
const schema = z.object({ userEmail: z.string() })
<FormField name="useremail" .../>  // wrong case ; JS object lookup fails
```

**Fail (path shape)** :
```ts
const schema = z.object({ user: z.object({ email: z.string() }) })
<FormField name="userEmail" .../>  // wrong : the schema path is "user.email"
```

**Output line** : `[4] Name prop exact match : FAIL. "emai" → did you mean "email" ?`

## 5. defaultValues Completeness

**Goal** : `useForm({ defaultValues: {...} })` has a non-undefined initial value for every top-level schema key (and for nested object keys, a non-undefined nested object). Without it, React fires "input is changing from uncontrolled to controlled" on the first keystroke.

**Pass criterion (per zod type)** :

| zod type | Required initial value |
|----------|------------------------|
| `z.string()`, `z.string().email()`, `z.string().min(n)` | `""` |
| `z.string().optional()` | `""` (preferred) or `undefined` is technically allowed but warns ; document the choice |
| `z.number()` | `0` (or null if `z.number().nullable()`) |
| `z.boolean()` | `false` |
| `z.enum([...])` | one of the enum values (or `""` ONLY if the field is required and Select has a placeholder ; warn) |
| `z.array(...)` | `[]` |
| `z.object({...})` | `{ ...nested defaults }` |
| `z.date()` | `undefined` (date pickers handle this) or a default Date |

**Code-pattern matcher** :

```regex
useForm\([\s\S]*?defaultValues\s*:\s*\{([\s\S]*?)\}
```

Extract the defaultValues object literal keys, compare to schema paths.

**Fail** :
```ts
const schema = z.object({ username: z.string(), agree: z.boolean() })
useForm({ resolver: zodResolver(schema), defaultValues: { username: "" } })
// missing: agree → Checkbox renders with checked={undefined} → uncontrolled → "input is changing from uncontrolled to controlled" warning the first time the user clicks it
```

**Pass** :
```ts
useForm({ resolver: zodResolver(schema), defaultValues: { username: "", agree: false } })
```

**Output line** : `[5] defaultValues completeness : FAIL. missing: [agree → false], [tags → []]`

## 6. handleSubmit Wrap

**Goal** : `<form onSubmit={form.handleSubmit(onValid)}>`. Optional : `form.handleSubmit(onValid, onInvalid)`.

**Pass criterion** :
- The `<form>` element inside `<Form {...form}>` has `onSubmit={form.handleSubmit(...)}`.
- The argument(s) to `handleSubmit` are function references (named or arrow), not a manually-invoked function.

**Code-pattern matcher** :

```regex
<form[^>]*onSubmit=\{form\.handleSubmit\(\s*(\w+)\s*(?:,\s*\w+\s*)?\)\}
```

**Fail (missing wrap)** :
```tsx
<form onSubmit={onSubmit}>     // direct call : validation bypassed, onSubmit gets the raw event
```

**Fail (immediate invocation)** :
```tsx
<form onSubmit={form.handleSubmit(onSubmit())}>  // calls onSubmit at render-time, passes the return value
```

**Pass** :
```tsx
<form onSubmit={form.handleSubmit(onSubmit)}>
```

**Output line** : `[6] handleSubmit wrap : FAIL. current: onSubmit={onSubmit}, required: onSubmit={form.handleSubmit(onSubmit)}`

## 7. zodResolver in useForm

**Goal** : `useForm({ resolver: zodResolver(formSchema), ... })` AND `import { zodResolver } from "@hookform/resolvers/zod"`.

**Pass criterion** :
- `zodResolver` is imported from `@hookform/resolvers/zod`.
- `useForm` receives `resolver: zodResolver(<the same schema declared in the file>)`.

**Code-pattern matcher** :

```regex
import\s+\{[^}]*zodResolver[^}]*\}\s+from\s+["']@hookform/resolvers/zod["']
useForm[^(]*\([\s\S]*?resolver\s*:\s*zodResolver\(\s*(\w+)\s*\)
```

Capture the schema variable name and confirm it matches the declared `const <name> = z.object(...)`.

**Fail (resolver missing)** :
```ts
const formSchema = z.object({ ... })
useForm({ defaultValues: { ... } })   // schema declared but never enforced
```

**Fail (wrong resolver)** :
```ts
useForm({ resolver: yupResolver(formSchema), ... })  // project is zod ; using yupResolver against a zod schema throws at submit
```

**Fail (import only)** :
```ts
import { zodResolver } from "@hookform/resolvers/zod"
useForm({ defaultValues: { ... } })  // imported but not wired
```

**Pass** :
```ts
import { zodResolver } from "@hookform/resolvers/zod"
useForm({ resolver: zodResolver(formSchema), defaultValues: { ... } })
```

**Output line** : `[7] zodResolver : FAIL. formSchema declared but resolver missing from useForm options`

## 8. No Nested FormProvider

**Goal** : exactly one `<Form>` (one FormProvider) wraps the form. No nested `<Form>`. No external `<FormProvider>` wrapping a `<Form>`.

**Pass criterion** :
- The JSX tree contains exactly one `<Form ...>` element along any root-to-leaf path.
- No `<FormProvider>` from `react-hook-form` directly used inside the tree (the shadcn `<Form>` already IS the FormProvider).

**Code-pattern matcher** :

```regex
# Count Form open tags on the path
<Form[\s>]
# Find FormProvider direct usage
<FormProvider
```

A simple heuristic : if more than one `<Form` token appears in the file (other than the import statement and inside comments), inspect for nesting.

**Fail** :
```tsx
<Form {...form}>
  <form onSubmit={form.handleSubmit(onSubmit)}>
    <Form {...form}>            // nested : two FormProvider contexts compete
      <FormField .../>
    </Form>
  </form>
</Form>
```

**Fail (external FormProvider)** :
```tsx
<FormProvider {...form}>
  <Form {...form}>              // double-wrap
    <FormField .../>
  </Form>
</FormProvider>
```

**Pass** :
```tsx
<Form {...form}>
  <form onSubmit={form.handleSubmit(onSubmit)}>
    <FormField .../>
  </form>
</Form>
```

**Output line** : `[8] No nested FormProvider : FAIL. nested <Form> at line 42`

## Order of Operations

ALWAYS execute the checks in this order :

1. Parse schema → extract schema paths.
2. Parse `useForm` → record resolver, defaultValues, schema-variable identity.
3. Walk JSX → record FormField names, control types per field, FormMessage presence per field.
4. Run checks 7, 6, 8 first (whole-form structural). A FAIL on 7 or 6 means the form does not validate or does not submit-with-validation at all : worth reporting before per-field detail.
5. Run checks 1, 4 (name set + name match) : these define the field-level integrity.
6. Run checks 2, 3, 5 per field : these are the per-field silent failures.
7. Emit the structured verdict with all eight rows even when most pass.

NEVER skip checks. NEVER reorder beyond this : checks 7 and 6 are global enablement, the rest are per-field.

## Verdict Aggregation

- Overall PASS if all eight checks pass.
- Overall FAIL if any one check fails.
- For each FAIL, the output MUST name : the failing rule, the offending location (field name or line region), the canonical fix.

ALWAYS name the canonical fix in the verdict line. NEVER emit "FAIL" without "fix: ...".
