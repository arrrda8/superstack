# Design Token System

Every project starts with a `tokens.css` (or inside `globals.css` via `@theme`) defining the design foundation. Deriving all colors from a single brand hue ensures visual consistency without manual color-picking.

## Three-Layer Architecture

1. **Primitive tokens** — raw OKLCH values (e.g., `--blue-500: oklch(59% 0.24 255)`)
2. **Semantic tokens** — purpose-driven (e.g., `--color-primary: var(--blue-500)`)
3. **Component tokens** — variant-specific (e.g., `--button-bg: var(--color-primary)`)

Semantic tokens are what make refactors safe — changing the brand hue updates everything.

## Tailwind v4 @theme Example

```css
/* globals.css */
@theme {
  /* Brand palette — change the hue (260) to shift entire palette */
  --color-brand: oklch(0.65 0.25 260);
  --color-brand-light: oklch(0.85 0.15 260);
  --color-brand-dark: oklch(0.45 0.25 260);

  /* Surfaces */
  --color-surface: oklch(0.98 0.005 260);
  --color-surface-dark: oklch(0.15 0.02 260);

  /* Text */
  --color-text: oklch(0.20 0.02 260);
  --color-text-muted: oklch(0.55 0.02 260);

  /* Borders */
  --color-border: oklch(0.90 0.01 260);

  /* Semantic */
  --color-success: oklch(0.65 0.20 145);
  --color-warning: oklch(0.75 0.15 85);
  --color-error: oklch(0.55 0.25 25);

  /* Spacing */
  --space-xs: 0.25rem;
  --space-sm: 0.5rem;
  --space-md: 1rem;
  --space-lg: 2rem;
  --space-xl: 4rem;
  --space-2xl: 8rem;

  /* Radii */
  --radius-sm: 0.375rem;
  --radius-md: 0.5rem;
  --radius-lg: 0.75rem;
  --radius-full: 9999px;

  /* Typography */
  --font-heading: 'Inter Variable', sans-serif;
  --font-body: 'Inter Variable', sans-serif;
  --font-mono: 'JetBrains Mono Variable', monospace;

  /* Shadows */
  --shadow-sm: 0 1px 2px oklch(0 0 0 / 0.05);
  --shadow-md: 0 4px 6px oklch(0 0 0 / 0.07);
  --shadow-lg: 0 10px 25px oklch(0 0 0 / 0.1);

  /* Transitions */
  --transition-fast: 150ms ease;
  --transition-base: 250ms ease;
  --transition-slow: 400ms ease;
  --transition-spring: 500ms cubic-bezier(0.34, 1.56, 0.64, 1);
}
```

## Dark Mode

Semantic tokens flip, primitives stay the same:

```css
@media (prefers-color-scheme: dark) {
  :root {
    --color-surface: var(--color-surface-dark);
    --color-text: oklch(0.93 0.01 260);
    --color-text-muted: oklch(0.65 0.01 260);
    --color-border: oklch(0.30 0.02 260);
  }
}
```

Rule of thumb: bump OKLCH lightness up 15-20% for dark theme text, down 15-20% for backgrounds.

## Generating a Full Palette

1. Extract your brand color's OKLCH hue (e.g., 260)
2. Generate an 11-step scale (50-950) by varying lightness: 50=97%, 100=93%, 200=87%, 300=78%, 400=67%, **500=55% (brand)**, 600=45%, 700=37%, 800=29%, 900=21%, 950=14%
3. Keep chroma consistent across the scale
4. Steps are perceptually even because OKLCH lightness is perceptually uniform

## Token Categories Quick Reference

| Category | Tokens |
|----------|--------|
| Color | primary, secondary, accent, destructive, surface, on-surface, muted, border |
| Spacing | xs, sm, md, lg, xl, 2xl (rem scale) |
| Typography | font-family, font-size (xs-4xl), font-weight, line-height |
| Shadows | sm, md, lg, xl (elevation system) |
| Radii | sm, md, lg, full |
| Animation | fast (150ms), base (250ms), slow (400ms), spring |
| Z-index | base(0), dropdown(10), sticky(20), modal(30), toast(40), tooltip(50) |
