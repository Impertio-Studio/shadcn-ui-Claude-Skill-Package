# Examples : shadcn Core Architecture

Concrete code demonstrating the copy-not-install paradigm, `components.json` configuration, CLI invocations, and registry resolution.

## 1. The copy-not-install paradigm in action

### 1.1 Project initialisation

```bash
# Initialise shadcn in an existing Next.js project
pnpm dlx shadcn@latest init

# Interactive prompts cover :
#   - style : new-york (recommended) | default (deprecated) | sera | luma
#   - baseColor : neutral | stone | zinc | mauve | olive | mist | taupe
#   - cssVariables : true (semantic tokens) | false (inline color utilities)
#   - aliases : @/components, @/lib/utils, @/components/ui, @/lib, @/hooks
```

After init, `components.json` exists at the project root and Tailwind config (v3) or main CSS (v4) carries the shadcn theme variables.

### 1.2 Adding a single component

```bash
pnpm dlx shadcn@latest add button
```

The CLI :
1. Resolves `button` against the default `@shadcn` registry.
2. Fetches the Button source.
3. Writes `components/ui/button.tsx` (or `.jsx` if `tsx: false`).
4. Installs runtime dependencies (`@radix-ui/react-slot`, `class-variance-authority`, `clsx`, `tailwind-merge`).
5. Reports the file paths it touched.

The Button source NOW lives in `components/ui/button.tsx` as application code. Commit it.

### 1.3 Importing the local component

```tsx
// app/page.tsx
import { Button } from '@/components/ui/button'   // LOCAL alias

export default function Home() {
  return <Button variant="default" size="default">Click me</Button>
}
```

ALWAYS use the local alias. NEVER `import { Button } from 'shadcn-ui'` (no such module).

### 1.4 Adding many components at once

```bash
pnpm dlx shadcn@latest add dialog dropdown-menu sheet form input label

# Or, install everything :
pnpm dlx shadcn@latest add --all
```

### 1.5 Preview a re-install (diff workflow)

```bash
pnpm dlx shadcn@latest add button --diff
```

Shows the differences between the current local `components/ui/button.tsx` and the registry's canonical source. ALWAYS run this BEFORE `--overwrite` if the local file has been customised.

### 1.6 Apply an upgrade

```bash
# After reviewing the diff and deciding to overwrite :
pnpm dlx shadcn@latest add button --overwrite

# Then manually re-apply local customisations from git diff.
```

## 2. `components.json` : minimal example

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

Notes :
- `style` and `baseColor` are **immutable after init**. Changing them later requires manual file-by-file reconciliation.
- `tailwind.config` is `""` (blank) for Tailwind v4 because v4 has no JS config; it is a path string (e.g., `"tailwind.config.js"`) for v3.
- `cssVariables: true` is the default; the alternative inlines color utilities like `bg-zinc-950` instead of `bg-background`.

## 3. `components.json` : with custom registries

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
    "cssVariables": true
  },
  "aliases": {
    "components": "@/components",
    "utils": "@/lib/utils",
    "ui": "@/components/ui"
  },
  "registries": {
    "@acme": {
      "url": "https://registry.acme.example.com/{name}.json",
      "headers": {
        "Authorization": "Bearer ${ACME_REGISTRY_TOKEN}"
      }
    },
    "@partner": "https://partner.example.com/registry"
  }
}
```

Then :

```bash
# Resolves to https://registry.acme.example.com/datepicker.json
pnpm dlx shadcn@latest add @acme/datepicker

# Resolves to https://partner.example.com/registry/checkout-form.json
pnpm dlx shadcn@latest add @partner/checkout-form
```

The `${ACME_REGISTRY_TOKEN}` placeholder reads from the local environment at CLI invocation time. ALWAYS keep secrets in `.env.local` or your shell environment, never in `components.json`.

## 4. CLI add : full flag matrix usage

```bash
# Add silently (no prompts, accept defaults)
pnpm dlx shadcn@latest add button -y

# Add with overwrite
pnpm dlx shadcn@latest add button --overwrite

# Add to a non-default path
pnpm dlx shadcn@latest add button --path components/widgets

# Preview without writing
pnpm dlx shadcn@latest add button --dry-run

# Preview diff against existing local file
pnpm dlx shadcn@latest add button --diff

# View registry item before installing
pnpm dlx shadcn@latest view button

# Add a block (multi-file scaffold)
pnpm dlx shadcn@latest add login-01
pnpm dlx shadcn@latest add dashboard-01
pnpm dlx shadcn@latest add sidebar-07
```

## 5. The local component is yours to edit

After `shadcn add button`, the local file looks roughly like :

```tsx
// components/ui/button.tsx
import * as React from "react"
import { Slot } from "@radix-ui/react-slot"
import { cva, type VariantProps } from "class-variance-authority"
import { cn } from "@/lib/utils"

const buttonVariants = cva(
  "inline-flex items-center justify-center whitespace-nowrap rounded-md text-sm font-medium ring-offset-background transition-colors focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2 disabled:pointer-events-none disabled:opacity-50",
  {
    variants: {
      variant: {
        default: "bg-primary text-primary-foreground hover:bg-primary/90",
        destructive: "bg-destructive text-destructive-foreground hover:bg-destructive/90",
        outline: "border border-input bg-background hover:bg-accent hover:text-accent-foreground",
        secondary: "bg-secondary text-secondary-foreground hover:bg-secondary/80",
        ghost: "hover:bg-accent hover:text-accent-foreground",
        link: "text-primary underline-offset-4 hover:underline",
      },
      size: {
        default: "h-10 px-4 py-2",
        sm: "h-9 rounded-md px-3",
        lg: "h-11 rounded-md px-8",
        icon: "h-10 w-10",
      },
    },
    defaultVariants: { variant: "default", size: "default" },
  }
)

export function Button({ className, variant, size, asChild = false, ...props }) {
  const Comp = asChild ? Slot : "button"
  return <Comp className={cn(buttonVariants({ variant, size, className }))} {...props} />
}
export { buttonVariants }
```

ALWAYS feel free to :
- Add a new variant : `loading: "bg-primary/60 cursor-wait"`.
- Add a new size : `xs: "h-7 rounded px-2 text-xs"`.
- Change the base class.
- Rename the component.
- Add framework-specific wiring : a `Link`-prop, a `loading`-prop, an icon slot.

After editing, the changes survive across project boundaries because the file lives in YOUR repo.

## 6. The traditional-library comparison (same Button, MUI side)

```bash
npm install @mui/material @emotion/react @emotion/styled
```

```tsx
import Button from '@mui/material/Button'

<Button variant="contained" color="primary">Click me</Button>
```

Differences :
- The MUI Button source is in `node_modules/@mui/material/Button/`. The consumer cannot meaningfully edit it.
- Customisation flows through `theme.components.MuiButton.styleOverrides`, the `sx` prop, or wrapping in a styled-component.
- An `npm update @mui/material` may introduce breaking changes via semver-major; the consumer's customisations need re-checking.

With shadcn, the equivalent customisation is "open the local file and edit". With MUI, it is "configure the theme override layer".

## 7. Registry hosting (short note)

ALWAYS host a custom registry as static JSON files behind a URL. For internal use, this can be :
- A folder in the same monorepo served via your dev server.
- A GitHub Pages site.
- A private CDN with the `headers` field providing auth.

Generate the registry JSON files with :

```bash
pnpm dlx shadcn@latest build --output ./public/r
```

See [shadcn-core-registry](../../shadcn-core-registry/SKILL.md) for the full schema and authoring workflow.
