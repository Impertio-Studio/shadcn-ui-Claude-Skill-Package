# Anti-patterns : framework integration failures

Five recurring init failures. Each entry : symptom, root cause, fix.

## AP-1 : `rsc: true` in a Vite project

**Symptom** : `<Dialog>` opens but mutating state inside its content
throws `Cannot read properties of undefined (reading useState)` at
runtime. Or : the build succeeds, the page loads, but interactive
components silently do nothing on click.

**Root cause** : `components.json` was hand-edited (or copy-pasted from
a Next.js example) to `"rsc": true`. The shadcn CLI then OMITS the
`"use client"` directive from components it deems "presentational",
trusting Next's RSC compiler to add boundary logic. Vite has no RSC
compiler. The components end up shipped without hooks support.

**Fix** :

```jsonc
// components.json
{ "rsc": false }
```

Re-run `pnpm dlx shadcn@latest add <component> --overwrite` for every
component installed under the wrong flag. NEVER set `rsc:true` outside
Next.js App Router or TanStack Start (with RSC enabled). ALWAYS let the
`-t vite` template write the file; do not hand-author it.

## AP-2 : `rsc: false` in Next.js App Router

**Symptom** : every shadcn component arrives with `"use client"` at
the top, even pure-presentation ones like `<Card>` and `<Badge>`. The
client bundle grows by 30 to 80 kB. Server-rendered pages lose static
HTML payload. Lighthouse drops.

**Root cause** : `components.json` shows `"rsc": false`, usually because
the project was initialized without `-t next` (the user ran `init` with
no template, the CLI defaulted to a generic shape, or the user copied
a Vite config). The CLI assumes the host cannot honor server boundaries
and emits `"use client"` defensively on every file.

**Fix** :

```jsonc
// components.json
{ "rsc": true }
```

Then `pnpm dlx shadcn@latest add <component> --overwrite` per component.
The CLI re-emits without `"use client"` on RSC-safe primitives. Verify
in DevTools network tab that the client bundle shrinks. NEVER set
`rsc:false` in Next.js App Router unless deliberately opting out of RSC.

## AP-3 : Astro without `@astrojs/react` integration installed

**Symptom** : `pnpm dlx shadcn@latest init -t astro` succeeds. The
first `<Button client:load />` in an `.astro` file logs
`Unable to render <Button>. There are no JSX renderers configured.`
at build time. The page renders the wrapper but no React.

**Root cause** : `astro create` was run WITHOUT `--add react`. The
`@astrojs/react` integration is missing from `astro.config.mjs`.
shadcn assumes the host can render React; Astro cannot without the
integration.

**Fix** : install and register the integration :

```bash
pnpm astro add react
```

This adds `@astrojs/react` to dependencies and updates
`astro.config.mjs` :

```js
import react from "@astrojs/react"
export default defineConfig({ integrations: [react()], /* ... */ })
```

Restart `astro dev`. ALWAYS use `--add react` when creating the Astro
project, OR run `astro add react` immediately after. NEVER ship shadcn
inside Astro without verifying `@astrojs/react` is in `integrations`.

## AP-4 : tsconfig paths not synced with components.json aliases

**Symptom** : the editor (VS Code, WebStorm) resolves
`@/components/ui/button` and shows no red squiggle, but
`pnpm dev` or `pnpm build` fails with
`Cannot find module '@/components/ui/button' or its corresponding
type declarations.` or `Failed to resolve import "@/components/ui/button"`.

**Root cause** : three places must agree :
1. `components.json.aliases.ui` (e.g. `"@/components/ui"`)
2. `tsconfig.json#compilerOptions.paths` (e.g. `"@/*": ["./src/*"]`)
3. The bundler alias :
   - Vite : `vite.config.ts#resolve.alias`
   - React Router : auto via create-react-router (do not touch)
   - Astro : nothing extra, but tsconfig must be correct
   - Next.js : nothing extra (Next resolves via tsconfig directly)

If any one drifts (most often : `tsconfig.json` updated but
`tsconfig.app.json` in Vite was forgotten, or `vite.config.ts` lacks
the `resolve.alias` block) the editor and the runtime disagree.

**Fix** : write the same alias to every file the framework demands.
For Vite specifically :

```jsonc
// tsconfig.json    AND    tsconfig.app.json (BOTH)
{ "compilerOptions": { "baseUrl": ".", "paths": { "@/*": ["./src/*"] } } }
```

```ts
// vite.config.ts
resolve: { alias: { "@": path.resolve(__dirname, "./src") } }
```

ALWAYS update both tsconfig files in Vite. NEVER trust the editor alone
to validate alias correctness; run `pnpm build` before committing.

## AP-5 : mixing Tailwind v3 and v4 in a transitional project

**Symptom** : a previously-working v3 project upgrades shadcn and gets
unstyled buttons. Or : a fresh `-t vite` project mixed with an old
`tailwind.config.js` from a template throws
`Cannot apply unknown utility class bg-background`.

**Root cause** : Tailwind v4 uses `@import "tailwindcss";` in a single
CSS file and defines tokens via `@theme { ... }` blocks. v3 uses
`tailwind.config.{js,ts}` with `content` globs, `theme.extend.colors`,
and HSL space-separated CSS variables. shadcn v4 components emit
`bg-background` referencing `--background` tokens in the new
`@theme inline` syntax. v3 PostCSS plugins cannot resolve them.

**Fix** : pick ONE version and remove the other's wiring :

For v4 :
1. Delete `tailwind.config.{js,ts}` (or set
   `components.json.tailwind.config = ""`).
2. Replace `globals.css` with `@import "tailwindcss";` and
   `@theme inline { ... }` for tokens.
3. Use the v4 plugin : `@tailwindcss/vite` (Vite/Astro/React-Router/
   TanStack Start) or `@tailwindcss/postcss` (Next.js).
4. Run `pnpm dlx shadcn@latest add <component> --overwrite` to refetch
   v4-shaped components.

For v3 (legacy projects only) :
1. Keep `tailwind.config.{js,ts}` with HSL tokens
   `--background: 0 0% 100%;` and `theme.extend.colors.background:
   "hsl(var(--background) / <alpha-value>)"`.
2. Use shadcn v3-era component shapes (note : new components from the
   registry assume v4 ; expect partial breakage).

NEVER ship a project with both `tailwind.config.js` AND
`@theme inline` blocks active. Pick a side. See
`shadcn-errors-tailwind-v3-v4-migration` for the full migration
recipe.
