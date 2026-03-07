---
description: Redesign an existing web project with Superstack's premium design standards
argument-hint: [optional scope: full | homepage | components | dark-mode]
---

You are now using the **Superstack** skill for redesigning an existing project. Read the skill file at `skills/superstack/SKILL.md` for design standards and anti-patterns.

## Process

1. **Analyze current design** — Screenshot or read the existing pages. Identify AI anti-patterns (symmetric grids, generic gradients, default icons, uniform spacing).

2. **Gather design direction** — Ask the user about: visual mood, brand colors/fonts, animation level, icon style, reference URLs. Use the Design Direction questions from Phase 0.

3. **Design token system** — Read `references/design-tokens.md`. Set up OKLCH color system derived from brand hue. Create `@theme` block in globals.css.

4. **Apply Superstack design standards** — Read `references/design-patterns.md` and `references/animation-playbook.md`:
   - Add Lenis smooth scroll + GSAP ScrollTrigger
   - Replace generic layouts with bento grids, asymmetric splits, editorial compositions
   - Add noise/grain textures to hero and feature sections
   - Switch to custom/curated icons (Phosphor, Tabler, Iconoir)
   - Implement staggered animations with spring physics
   - Ensure responsive at 375px, 768px, 1024px, 1440px

5. **Quality check** — Verify against the anti-patterns table and quality checklist in SKILL.md.

If the user specified a scope: "$ARGUMENTS" — focus on that area. Otherwise do a full redesign.
