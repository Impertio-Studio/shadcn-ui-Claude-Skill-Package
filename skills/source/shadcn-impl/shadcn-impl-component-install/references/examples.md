# Examples : End-to-End Component Install Workflows

Every example below is run-as-shown. No omissions, no "fill-in-blanks".

## Example 1 : Greenfield Next.js Project, init + add Button

```bash
# 1. Create the Next.js app.
pnpm create next-app@latest my-app --typescript --tailwind --app --eslint
cd my-app

# 2. Initialize shadcn (interactive ; pick new-york / neutral / cssVariables=yes).
pnpm dlx shadcn@latest init

# 3. Inspect the generated components.json.
cat components.json
```

The init produces a `components.json` like :

```json
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

Note `tailwind.config: ""` : Next.js scaffolds Tailwind v4, so the
config-file field is empty (v4 is CSS-first, no `tailwind.config.js`).

```bash
# 4. Add Button.
pnpm dlx shadcn@latest add button

# 5. Inspect the result.
cat components/ui/button.tsx | head -20
```

Output : `components/ui/button.tsx` containing the standard cva-based
Button, importing from `@/lib/utils`, with `cursor-pointer` if you passed
`--pointer` to init. Runtime deps installed automatically :
`@radix-ui/react-slot` (for the `asChild` Slot composition).

```bash
# 6. Use the button.
```
```tsx
// app/page.tsx
import { Button } from "@/components/ui/button"
export default function Home() {
  return <Button variant="default" size="default">Click me</Button>
}
```

## Example 2 : Vite + React Greenfield with the Full Three-File Alias Setup

The Vite path-alias gotcha : Vite splits TS config across `tsconfig.json`
and `tsconfig.app.json`. Both need the alias. Vite itself needs the alias
in `vite.config.ts`. Three places, identical alias.

```bash
# 1. Scaffold Vite + React + TS.
pnpm create vite@latest my-vite-app
# Select : React, TypeScript
cd my-vite-app
pnpm install

# 2. Add Tailwind v4.
pnpm add tailwindcss @tailwindcss/vite
```

Edit `src/index.css` :

```css
@import "tailwindcss";
```

Edit `tsconfig.json` :

```json
{
  "files": [],
  "references": [
    { "path": "./tsconfig.app.json" },
    { "path": "./tsconfig.node.json" }
  ],
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    }
  }
}
```

Edit `tsconfig.app.json` (identical alias block) :

```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    }
    /* ...other compilerOptions left as scaffold default... */
  },
  "include": ["src"]
}
```

```bash
pnpm add -D @types/node
```

Edit `vite.config.ts` :

```ts
import path from "path"
import react from "@vitejs/plugin-react"
import tailwindcss from "@tailwindcss/vite"
import { defineConfig } from "vite"

export default defineConfig({
  plugins: [react(), tailwindcss()],
  resolve: {
    alias: {
      "@": path.resolve(__dirname, "./src"),
    },
  },
})
```

```bash
# 3. Initialize shadcn for the Vite project.
pnpm dlx shadcn@latest init -t vite

# 4. Add Button.
pnpm dlx shadcn@latest add button
```

Source : https://ui.shadcn.com/docs/installation/vite (verified 2026-05-19).

## Example 3 : Add Button, Then Customise a Variant in Place

```bash
pnpm dlx shadcn@latest add button
git add . && git commit -m "Add raw shadcn Button"
```

Edit `components/ui/button.tsx`. Add a `brand` variant and an `xl` size :

```tsx
const buttonVariants = cva(
  "inline-flex items-center justify-center gap-2 whitespace-nowrap rounded-md text-sm font-medium transition-all disabled:pointer-events-none disabled:opacity-50 [&_svg]:pointer-events-none [&_svg:not([class*='size-'])]:size-4 shrink-0 outline-none focus-visible:ring-[3px] focus-visible:ring-ring/50 cursor-pointer",
  {
    variants: {
      variant: {
        default: "bg-primary text-primary-foreground hover:bg-primary/90",
        destructive: "bg-destructive text-white hover:bg-destructive/90",
        outline: "border bg-background hover:bg-accent hover:text-accent-foreground",
        secondary: "bg-secondary text-secondary-foreground hover:bg-secondary/80",
        ghost: "hover:bg-accent hover:text-accent-foreground",
        link: "text-primary underline-offset-4 hover:underline",
        // ADDED for this project :
        brand: "bg-[oklch(0.62_0.18_265)] text-white hover:bg-[oklch(0.56_0.18_265)]",
      },
      size: {
        default: "h-9 px-4 py-2 has-[>svg]:px-3",
        sm: "h-8 rounded-md gap-1.5 px-3 has-[>svg]:px-2.5",
        lg: "h-10 rounded-md px-6 has-[>svg]:px-4",
        icon: "size-9",
        // ADDED for this project :
        xl: "h-12 rounded-md px-8 text-lg has-[>svg]:px-6",
      },
    },
    defaultVariants: { variant: "default", size: "default" },
  }
)
```

```bash
git add . && git commit -m "Customise Button : brand variant + xl size"
```

Two distinct commits : the raw `add` and your customisation. Later when
upstream changes ship, `git diff` between these commits shows EXACTLY what
to port forward.

## Example 4 : Add Form, Integrate with Existing Auth Context

```bash
pnpm dlx shadcn@latest add form input label
```

This installs : `react-hook-form`, `@hookform/resolvers`, `zod`,
`@radix-ui/react-label`, `@radix-ui/react-slot`. Files written :
`components/ui/form.tsx`, `components/ui/input.tsx`,
`components/ui/label.tsx`.

Integrate the Form with an existing auth context :

```tsx
// app/login/page.tsx
"use client"
import { useContext } from "react"
import { useForm } from "react-hook-form"
import { zodResolver } from "@hookform/resolvers/zod"
import { z } from "zod"
import { Form, FormField, FormItem, FormLabel, FormControl, FormMessage } from "@/components/ui/form"
import { Input } from "@/components/ui/input"
import { Button } from "@/components/ui/button"
import { AuthContext } from "@/context/auth"

const schema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
})

export default function LoginPage() {
  const auth = useContext(AuthContext)
  const form = useForm<z.infer<typeof schema>>({
    resolver: zodResolver(schema),
    defaultValues: { email: "", password: "" },
  })

  async function onSubmit(values: z.infer<typeof schema>) {
    await auth.signIn(values.email, values.password)
  }

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
        <Button type="submit" disabled={form.formState.isSubmitting}>Sign in</Button>
      </form>
    </Form>
  )
}
```

Form is the shadcn primitive layer ; for the deeper Form usage doctrine,
see `shadcn-impl-form-validation`. The `add form` invocation is identical
regardless of which context you integrate against.

## Example 5 : Update Workflow with Diff

Project is six months old, original Button has a `brand` variant added
locally. Upstream shadcn has shipped `cursor-pointer` defaults and a
new `data-icon` slot.

```bash
# 1. Commit your current state.
git add . && git commit -m "Snapshot before shadcn sync"

# 2. Preview what would change for Button only.
pnpm dlx shadcn@latest add button --diff
```

Output (truncated diff style) :

```diff
--- components/ui/button.tsx
+++ <registry>/button.tsx
@@ -3,7 +3,7 @@
 import { cva, type VariantProps } from "class-variance-authority"
 import { cn } from "@/lib/utils"

-const buttonVariants = cva("inline-flex items-center ...", {
+const buttonVariants = cva("inline-flex items-center ... cursor-pointer [&_svg]:pointer-events-none ...", {
   variants: {
     variant: {
       default: "bg-primary ...",
@@ -15,7 +15,6 @@
       link: "text-primary underline-offset-4 hover:underline",
-      brand: "bg-[oklch(0.62_0.18_265)] text-white hover:bg-[oklch(0.56_0.18_265)]",
     },
```

Reading the diff : upstream added `cursor-pointer` and the `[&_svg]`
spacing pattern. The `brand` variant would be REMOVED if you accept the
rewrite (because it does not exist in the registry).

```bash
# 3. Accept the rewrite.
pnpm dlx shadcn@latest add button --overwrite

# 4. Re-add the brand variant. Find what was lost :
git diff HEAD components/ui/button.tsx
# (shows the brand variant deletion)

# 5. Hand-port the brand variant onto the new shape :
# edit components/ui/button.tsx, re-add the `brand` entry in `variant`.

# 6. Commit the merged result.
git add . && git commit -m "Sync Button to upstream + re-apply brand variant"
```

ALWAYS run step 1 (the snapshot commit) before `--overwrite`. The CLI does
not back up the previous file content ; recovery without git is manual.

## Example 6 : package.json#imports Setup (shadcn 4.7.0+)

For an ESM-pure project that wants Node-native subpath imports :

Edit `package.json` :

```json
{
  "name": "my-app",
  "type": "module",
  "imports": {
    "#components/*": "./src/components/*",
    "#lib/*": "./src/lib/*",
    "#hooks/*": "./src/hooks/*",
    "#registry/*": "./src/registry/*"
  }
}
```

Run init (interactive ; when asked about aliases, point them at the
`#`-prefixed forms) :

```bash
pnpm dlx shadcn@latest init
```

Verify the resulting `components.json` :

```json
{
  "aliases": {
    "utils": "#lib/utils",
    "components": "#components",
    "ui": "#components/ui",
    "lib": "#lib",
    "hooks": "#hooks"
  }
}
```

```bash
pnpm dlx shadcn@latest add button
```

The generated `src/components/ui/button.tsx` imports `cn` as :

```tsx
import { cn } from "#lib/utils"
```

NEVER configure `@/*` in tsconfig AND `#components/*` in package.json#imports
for the same project. The resolution order is implementation-defined and
silent. Stick to one resolver.

Source : https://ui.shadcn.com/docs/changelog (May 2026 entry).

## Example 7 : Custom Registry Add (Private Design System)

Your org has a private registry at `https://registry.myorg.com/{name}.json`
behind an Authorization header.

```bash
# Set the token in your shell or .env :
export REGISTRY_TOKEN="ghp_xxxxxxxxxxxxxxxxxxxx"
```

Edit `components.json` :

```json
{
  "registries": {
    "@myorg": {
      "url": "https://registry.myorg.com/{name}.json",
      "headers": { "Authorization": "Bearer ${REGISTRY_TOKEN}" }
    }
  }
}
```

Add a private component :

```bash
pnpm dlx shadcn@latest add @myorg/billing-form
```

The CLI substitutes `{name}` with `billing-form`, sends the GET request
with the Authorization header (expanded from `${REGISTRY_TOKEN}` at
invocation time), and writes the resulting file(s) to your `aliases.ui`
or `aliases.components` path.

ALWAYS keep secrets in env vars, NEVER in components.json or the URL
directly. components.json is committed to git ; the token must not be.

For the full registry-authoring + resolution semantics : see
`shadcn-core-registry`.

## Verified Sources

- https://ui.shadcn.com/docs/cli (verified 2026-05-19)
- https://ui.shadcn.com/docs/installation/vite (verified 2026-05-19)
- https://ui.shadcn.com/docs/components/radix/form (verified 2026-05-19)
- https://ui.shadcn.com/docs/changelog (May 2026 package.json#imports entry ; verified 2026-05-19)
- https://ui.shadcn.com/docs/registry (verified 2026-05-19)
