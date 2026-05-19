# References : Anti-Patterns (theming-custom)

Five canonical failures that recur in shadcn theming work. Each entry : the symptom, the wrong code, the WHY it fails, and the fix.

## AP-1 : Flash of unstyled content (missing suppressHydrationWarning)

**Symptom** : on first paint of a Next.js page, the screen briefly renders in the wrong theme (e.g. light when the user's saved preference is dark), then snaps to the correct theme after about 100ms. The browser console logs a React hydration mismatch warning.

**Wrong** :

```tsx
// app/layout.tsx
export default function RootLayout({ children }) {
  return (
    <html lang="en">                                {/* missing suppressHydrationWarning */}
      <body>
        <ThemeProvider attribute="class" defaultTheme="system" enableSystem disableTransitionOnChange>
          {children}
        </ThemeProvider>
      </body>
    </html>
  )
}
```

**WHY it fails** : Next.js renders the HTML on the server with NO theme class on `<html>` (server has no access to `localStorage` or `window.matchMedia`). The client mounts, `next-themes` reads the saved preference, and adds `class="dark"` to `<html>`. React 18+ detects the server-vs-client HTML difference and logs a hydration mismatch warning. The first frame paints with the default theme ; the second frame snaps to the correct one. Users see a flash.

**Fix** :

```tsx
<html lang="en" suppressHydrationWarning>
```

`suppressHydrationWarning` tells React the `<html>` element's attributes are EXPECTED to differ between server and client. The warning is suppressed ; the flash is unavoidable in pure CSR-after-SSR but can be minimized via the next-themes inline script (which the library injects automatically when `attribute="class"` is set ; see https://github.com/pacocoursey/next-themes for the script-injection mechanism).

ALWAYS apply `suppressHydrationWarning` ONLY to `<html>`. NEVER apply it to other elements ; it masks real hydration bugs elsewhere.

## AP-2 : HSL comma-separated values in v3 config

**Symptom** : after pasting a custom palette into `globals.css`, every `bg-*` utility renders the literal CSS variable string in the inspector or produces no color at all. The page appears unstyled where the new utility was used.

**Wrong** :

```css
/* globals.css (v3 project) */
@layer base {
  :root {
    --background: 0, 0%, 100%;       /* commas */
    --primary: hsl(263, 70%, 58%);    /* hsl() wrapper inside the variable */
  }
}
```

```js
// tailwind.config.js
module.exports = {
  theme: {
    extend: {
      colors: {
        background: "hsl(var(--background))",  // wraps with hsl() again
        primary: "hsl(var(--primary))",
      },
    },
  },
}
```

**WHY it fails** : the v3 convention stores HSL components as a space-separated triple in the variable, then wraps with `hsl()` at the consumption site in `tailwind.config.js`. Writing `--background: 0, 0%, 100%` produces `hsl(0, 0%, 100%)` after wrapping. That LOOKS valid, but the comma form is incompatible with the space-separated wrapping convention shadcn uses. Worse : writing `--primary: hsl(263, 70%, 58%)` produces `hsl(hsl(263, 70%, 58%))` after wrapping, which IS invalid CSS, and the browser drops the declaration silently. Tailwind then renders `bg-primary` as an empty utility.

**Fix** : strip the commas and the wrapper from the variable values :

```css
@layer base {
  :root {
    --background: 0 0% 100%;     /* space-separated, no commas */
    --primary: 263 70% 58%;       /* no hsl() wrapper inside the variable */
  }
}
```

The `tailwind.config.js` wrapping (`hsl(var(--background))`) stays unchanged. ALWAYS write v3 HSL values as exactly three space-separated components with no commas and no `hsl()` wrapper.

This is the SINGLE most common bug AI assistants introduce when generating shadcn v3 themes. Verified anti-pattern documented in https://ui.shadcn.com/docs/theming.

## AP-3 : oklch values used in v3 config (won't parse)

**Symptom** : after migrating a project, OR after copying a theme block from the modern theme builder into a project still on Tailwind v3, every themed utility produces no style. The browser console may show "Invalid property value" warnings for color properties.

**Wrong** :

```css
/* globals.css (v3 project still on Tailwind 3.4) */
@layer base {
  :root {
    --background: oklch(1 0 0);              /* oklch in a v3 project */
    --primary: oklch(0.488 0.243 264.376);
  }
}
```

```js
// tailwind.config.js
module.exports = {
  theme: {
    extend: {
      colors: {
        background: "hsl(var(--background))",  // wraps with hsl()
        primary: "hsl(var(--primary))",
      },
    },
  },
}
```

**WHY it fails** : the v3 `tailwind.config.js` wraps variables with `hsl()`. With oklch values, the wrapped result is `hsl(oklch(1 0 0))`. That is invalid CSS. The browser drops the declaration. The Tailwind utility produces no color.

This commonly happens when a user clicks "Copy code" on https://ui.shadcn.com/themes (which defaults to v4 oklch output) and pastes into a project that is still on Tailwind v3. The theme builder has a format toggle in the right panel that emits HSL space-separated values for v3 ; users miss it.

**Fix path A** : in the theme builder, switch the format toggle to HSL BEFORE clicking "Copy code". The output then matches the v3 wrapper convention.

**Fix path B** : migrate the project to Tailwind v4. The migration is a one-time cost ; subsequent theme builder copy-paste cycles work without thinking. See `shadcn-errors-tailwind-v3-v4-migration` for the full migration path.

ALWAYS verify project Tailwind generation BEFORE pasting theme CSS. `package.json` `"tailwindcss": "^4.x"` is v4 ; `"^3.x"` is v3.

## AP-4 : next-themes attribute="data-theme" instead of "class"

**Symptom** : the ModeToggle component appears to work (clicking changes a `data-theme` attribute on `<html>` from `light` to `dark`), but the actual page colors do NOT change. The `.dark` class is never applied. Tailwind's `dark:` variants never fire.

**Wrong** :

```tsx
// app/layout.tsx
<ThemeProvider
  attribute="data-theme"              {/* WRONG : default in next-themes, broken in shadcn */}
  defaultTheme="system"
  enableSystem
  disableTransitionOnChange
>
  {children}
</ThemeProvider>
```

**WHY it fails** : `next-themes`'s default `attribute` is `"data-theme"`, which sets `<html data-theme="dark">`. The shadcn theming model uses a CLASS selector (`.dark { --background: ...; }`) and Tailwind's `dark:` variant is configured by `@custom-variant dark (&:is(.dark *));` in v4 or `darkMode: ["class"]` in v3. Both rely on a CLASS, not a data-attribute. So the toggle changes the attribute, but nothing in the project listens for `data-theme="dark"` ; the page stays light.

**Fix** :

```tsx
<ThemeProvider
  attribute="class"                   {/* REQUIRED for shadcn */}
  defaultTheme="system"
  enableSystem
  disableTransitionOnChange
>
```

ALWAYS pass `attribute="class"` to next-themes in a shadcn project. The shadcn-distributed docs example uses `"class"` ; anyone deviating breaks the toggle silently.

Alternate (rare) : if a project genuinely wants `data-theme`-based theming, the entire shadcn theming setup must be rewritten. `.dark` rules become `[data-theme=dark]` rules, the v4 `@custom-variant dark` becomes `@custom-variant dark (&:is([data-theme=dark] *))`, and the v3 `darkMode: ["class"]` becomes `darkMode: ["selector", "[data-theme='dark']"]`. This is rarely worth the complexity ; stick with `"class"`.

## AP-5 : Two ThemeProviders nested

**Symptom** : the ModeToggle clicks but the page changes erratically. Sometimes the click is ignored. Sometimes the theme flips twice. The browser console may log no error ; the bug is in the React tree, not in CSS.

**Wrong** : two providers nested somewhere in the tree, often introduced when a developer adds app-state context (e.g. user-preferences) on top of next-themes :

```tsx
// app/layout.tsx
<html lang="en" suppressHydrationWarning>
  <body>
    <ThemeProvider attribute="class" defaultTheme="system" enableSystem disableTransitionOnChange>
      <UserPreferencesProvider>
        {/* somewhere deep inside UserPreferencesProvider, a second ThemeProvider is mounted */}
        <ThemeProvider attribute="class" defaultTheme="dark">    {/* WRONG : double-mount */}
          {children}
        </ThemeProvider>
      </UserPreferencesProvider>
    </ThemeProvider>
  </body>
</html>
```

OR (Vite-side) : the custom Context ThemeProvider mounted twice, once at `main.tsx` and again inside a route component.

**WHY it fails** : `next-themes` maintains internal state about the current theme. With two providers, BOTH listen to `localStorage` events. The outer provider applies the class to `<html>` ; the inner provider tries to apply ANOTHER class on its rendered subtree (which `next-themes` does NOT support cleanly). The result : clicks on the toggle update one provider's state but not the other's. The displayed theme depends on cascade order and re-render timing.

For the Vite custom Context provider, the failure is different : `useTheme()` reads the CLOSEST provider, so components inside the inner provider see different theme state than components outside it. The toggle in the header reads the outer provider ; a Settings panel inside the inner provider reads the inner one. The two states drift.

**Fix** : mount EXACTLY ONE ThemeProvider, at the highest layout boundary :

```tsx
// app/layout.tsx (Next.js)
<html lang="en" suppressHydrationWarning>
  <body>
    <ThemeProvider attribute="class" defaultTheme="system" enableSystem disableTransitionOnChange>
      <UserPreferencesProvider>
        {children}                                  {/* no second ThemeProvider */}
      </UserPreferencesProvider>
    </ThemeProvider>
  </body>
</html>
```

ALWAYS audit for nested providers when the toggle behaves erratically. Search the codebase for `ThemeProvider` ; expect exactly one mount site. NEVER co-locate a `ThemeProvider` with a route component or a feature-flag boundary ; theme state belongs at the app root.

NEVER mix the Vite custom Context provider AND `next-themes` in the same app. Pick one based on the framework (next-themes for Next.js, custom Context for Vite) and stay consistent.
