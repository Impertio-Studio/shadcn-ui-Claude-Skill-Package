# Anti-Patterns: Tailwind v3 to v4 Migration

Each anti-pattern below is a real, reproducible bug observed in shadcn projects during or after migration. The "Symptom" line describes what the user sees; the "Root cause" line explains the parser or compiler behavior; the "Fix" gives the exact remediation; the "Why this matters" line explains why AI-generated code reproduces this bug at high frequency.

Six minimum, in order of frequency.

## Anti-pattern 1: Mixing `@theme inline` and `tailwind.config.js` in the same project

```css
/* globals.css (v4 syntax) */
@import "tailwindcss";

@theme inline {
  --color-background: var(--background);
}
```

```js
// tailwind.config.js (v3 syntax STILL PRESENT)
module.exports = {
  theme: {
    extend: {
      colors: {
        background: "hsl(var(--background))",
      },
    },
  },
};
```

- **Symptom** : `bg-background` renders correctly in some component files and as transparent in others, depending on import order. Hot-reload "fixes" the color until next refresh. The bug appears flaky.
- **Root cause** : v4 loads the CSS `@theme inline` block and registers `--color-background` as a Tailwind theme entry. If a `tailwind.config.js` exists, v4 also reads it (when explicitly referenced via `@config`, OR by some toolchain auto-detection paths) and tries to merge a SECOND `colors.background` entry that wraps the same CSS variable through `hsl()`. The two resolvers compete: one yields `var(--background)` (oklch-correct), the other yields `hsl(var(--background))` (which is `hsl(oklch(...))` and parses to invalid color).
- **Fix** : delete `tailwind.config.js`. If the file must remain (custom keyframes, custom fonts, project-specific extensions), remove its color extension entirely so v4's `@theme inline` is the only source.
- **Why this matters** : AI generators copy a v4 `globals.css` from one example and a v3 `tailwind.config.js` from another. The two snippets look correct in isolation but produce a half-migrated project. ALWAYS verify single-source-of-truth for the color extension.

## Anti-pattern 2: Keeping `tailwindcss-animate` in `package.json` after switching to v4 CSS

```json
{
  "dependencies": {
    "tailwindcss-animate": "^1.0.7"
  }
}
```

```css
/* globals.css */
@import "tailwindcss";
/* tw-animate-css is NOT imported here */
```

- **Symptom** : Accordion items expand instantly with no slide animation. Dialog opens with no fade. Sheet slides in instantly. Build succeeds with no warnings.
- **Root cause** : v4 does NOT load the `tailwindcss-animate` plugin even when present in `node_modules`. The plugin's keyframes (`accordion-down`, `accordion-up`, `slide-in-from-right`, `fade-in-0`, `zoom-in-95`) are never registered. shadcn components reference these keyframes by name; with the plugin missing, the animations resolve to no-op.
- **Fix** : run `pnpm remove tailwindcss-animate && pnpm add tw-animate-css`, then add `@import "tw-animate-css";` near the top of `globals.css`.
- **Why this matters** : the `tailwindcss-animate` package STILL EXISTS on npm and STILL INSTALLS in v4 projects without error. There is no automatic warning. AI code generators frequently emit `pnpm add tailwindcss-animate` when adapting shadcn instructions, because most training data predates the v4 plugin swap.

## Anti-pattern 3: HSL space-separated tokens left in a v4 file

```css
/* globals.css */
@import "tailwindcss";

:root {
  --background: 0 0% 100%;  /* v3 format : raw HSL components */
  --foreground: 0 0% 3.9%;
}

@theme inline {
  --color-background: var(--background);
}
```

- **Symptom** : `bg-background` renders as transparent. DevTools shows `background-color: 0 0% 100%` which is not a valid CSS color.
- **Root cause** : v4 expects each CSS variable to be a complete CSS color value (oklch, hsl-with-parens, rgb-with-parens, or a named color). Raw `0 0% 100%` is three space-separated values that ONLY become a color when wrapped in `hsl(...)`. The v3 architecture wrapped these in `tailwind.config.js` via `"hsl(var(--background))"` extending the color theme; v4 does not wrap and treats the raw value as a literal CSS string for the `background-color` property, which is invalid.
- **Fix** : wrap every token with `oklch(...)` (preferred for v4) or `hsl(...)`. Example: `--background: oklch(1 0 0);` or `--background: hsl(0 0% 100%);`.
- **Why this matters** : the most common AI-generated migration uses `@import "tailwindcss";` on line 1 but keeps the v3 token block untouched because the visual diff looks small. The file LOOKS migrated but parses incorrectly.

## Anti-pattern 4: oklch tokens placed in a v3 file (reverse direction)

```css
/* globals.css */
@tailwind base;
@tailwind components;
@tailwind utilities;

@layer base {
  :root {
    --background: oklch(1 0 0);  /* v4 format in a v3 file */
  }
}
```

```js
// tailwind.config.js (v3, still present)
module.exports = {
  theme: {
    extend: {
      colors: {
        background: "hsl(var(--background))",
      },
    },
  },
};
```

- **Symptom** : `bg-background` renders as transparent or an undefined color. The JS config still wraps the variable in `hsl(...)` which produces `hsl(oklch(1 0 0))`, an invalid nested function.
- **Root cause** : the v3 JS color extension hardcodes `hsl(...)` wrapping at the Tailwind utility level. If the token value is already an oklch color, the wrapping produces `hsl(oklch(...))` which is not a valid CSS color expression and falls back to `transparent` or browser-default.
- **Fix** : decide which generation the project is on. If staying on v3, use HSL space-separated tokens. If migrating to v4, do steps 1 through 14 of the runbook in `SKILL.md`. NEVER mix oklch tokens with a v3 JS config.
- **Why this matters** : AI tools sometimes "modernize" the token block to oklch (which they correctly recognize as the new format) without updating the surrounding JS config. The result is a project that parses no color correctly.

## Anti-pattern 5: `ring-2` still in code after v4 upgrade (silent visual regression)

```tsx
// components/ui/button.tsx (copied pre-v4)
const buttonVariants = cva(
  "... focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2 ...",
  { /* ... */ }
);
```

- **Symptom** : focused buttons show a thin 2-pixel ring instead of the expected 3-pixel ring. The visual difference is small but every focusable component is affected (Button, Input, Select, Tabs trigger, Switch, Checkbox, RadioGroup item, Toggle).
- **Root cause** : in v3 the default `ring` was 3px and `ring-2` meant 2px. In v4 the default `ring` was reduced to 1px and the visual equivalent of v3's default is `ring-3`. The number in `ring-2` is still legal and still renders at 2px in v4; what changed is that the v3 design system tuned focus styles relative to the OLD default. shadcn v4 components were re-tuned to `ring-[3px]` to compensate. Components NOT regenerated after the upgrade still carry v3 ring values.
- **Fix** : `pnpm dlx shadcn@latest add button input select tabs switch checkbox radio-group toggle --overwrite` to refresh all focusable components. This is faster and more reliable than hand-editing each file.
- **Why this matters** : the bug is silent (no error, no warning, only a visual regression). AI agents iterating on a project rarely run `shadcn add --overwrite` after a migration because they assume the codemod handled everything, but the codemod does not touch shadcn-generated component files.

## Anti-pattern 6: `darkMode: ['class']` in `tailwind.config.js` AND `@custom-variant dark` in `globals.css` (both wired)

```js
// tailwind.config.js (still present, never deleted)
module.exports = {
  darkMode: ["class"],
  // ...
};
```

```css
/* globals.css */
@import "tailwindcss";
@custom-variant dark (&:is(.dark *));
```

- **Symptom** : dark mode appears to work but a subset of utility classes do not flip. The mode toggle adds `class="dark"` to `<html>`; some classes restyle, others stay on their light variant. The pattern of which classes restyle is unpredictable.
- **Root cause** : v4 reads ONLY the CSS variant. The JS `darkMode` key is dead. However, if the project ALSO uses `@config "../tailwind.config.js";` (referenced explicitly), v4 still loads the JS config but tolerates the duplicate. The unpredictability comes from utility-class generation: some utilities are generated through the v4 CSS pipeline (correctly variant-aware) and some via legacy paths (using the JS config variant rules).
- **Fix** : delete `tailwind.config.js` if not needed. If it must remain (for custom keyframes or fonts), remove the `darkMode` key entirely. The `@custom-variant dark` line in CSS is the SINGLE source of truth in v4.
- **Why this matters** : v4's CSS-variant model and v3's JS-key model coexist when both are present, producing half-correct dark mode. AI-generated migrations frequently leave both in place because each looks correct in its own file. ALWAYS pick one source.

## Anti-pattern 7 (bonus): `@tailwind` directives left at the top of a v4 file

```css
/* globals.css */
@tailwind base;
@tailwind components;
@tailwind utilities;
@import "tailwindcss";

@theme inline { /* ... */ }
```

- **Symptom** : build emits "Unknown at-rule @tailwind" warnings. Some Tailwind utilities work, others do not. PostCSS / Vite output is noisy.
- **Root cause** : v4 removed `@tailwind base`, `@tailwind components`, and `@tailwind utilities` directives. The single replacement is `@import "tailwindcss";`. Leaving the directives in place AND adding the import means the v4 parser encounters unknown directives and emits warnings; depending on toolchain order, some utility layers may register twice or not at all.
- **Fix** : delete the three `@tailwind` lines. Keep only `@import "tailwindcss";`.
- **Why this matters** : AI tools generating a v4 `globals.css` often append `@import "tailwindcss";` ABOVE existing v3 directives rather than replacing them, because the diff looks safer. The result is a file with both syntaxes.

## Anti-pattern 8 (bonus): Chart fills wrapping `hsl(var(--chart-1))` in v4

```tsx
const chartConfig = {
  desktop: { label: "Desktop", color: "hsl(var(--chart-1))" },  // v3 wrapping
};
```

- **Symptom** : recharts renders bars or lines with no fill color in a v4 project. The legend still shows the swatch correctly (because the swatch reads from CSS directly).
- **Root cause** : in v4 the `--chart-1` token is already a valid CSS color (oklch). Wrapping it in `hsl(...)` produces `hsl(oklch(...))`, invalid. The chart consumer reads `chartConfig.desktop.color` as a string and passes it directly to recharts which passes it to SVG `fill`. SVG fill rejects the invalid expression and falls back to none.
- **Fix** : in v4 use `color: "var(--chart-1)"` without wrapping.
- **Why this matters** : the shadcn Charts docs were updated for v4 to drop the `hsl()` wrapper, but example code in tutorials and AI training data still shows the v3 form.
