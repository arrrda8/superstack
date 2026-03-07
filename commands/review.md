---
description: Full project review against all Superstack quality standards (design, code, copy, security, a11y, performance, SEO, compliance)
argument-hint: [optional path to project root]
---

You are now using the **Superstack** skill for a comprehensive project review. Read the skill file at `skills/superstack/SKILL.md` for all quality standards.

Run a full review of the current project against every Superstack quality standard. This covers ALL aspects — not just security or performance, but everything from design quality to legal compliance.

## Review Areas (use parallel agents)

### Design Quality
- Check against the AI anti-patterns table in SKILL.md
- Lenis smooth scroll present?
- OKLCH color system or scattered hex values?
- Custom icons or default Lucide/Heroicons?
- Noise/grain textures on key sections?
- Creative layouts (bento, asymmetric) or generic grids?
- Responsive at 375px, 768px, 1024px, 1440px?
- Dark mode working (if applicable)?
- Read `references/design-patterns.md` for full checklist

### Code Quality
- TypeScript strict mode, no `any`?
- Error boundaries, loading/empty states handled?
- UI states complete (loading, error, empty, 404/500)? See `references/ui-state-patterns.md`
- Component API design? See `references/component-catalog.md`
- State management patterns? See `references/state-management.md`
- Linting clean (Biome/ESLint)?

### Copy Quality
- Headlines address pain points, not features?
- CTAs specific and action-oriented?
- No lorem ipsum, TODO placeholders, or generic "Welcome to" text?
- Read `references/copywriting-framework.md` for standards

### Security (Phase 4)
- Read `references/security-checklist.md`
- XSS, CSRF, IDOR, injection, auth, env leaks, dependencies

### Accessibility (Phase 5)
- Read `references/accessibility.md`
- WCAG 2.1 AA: semantic HTML, keyboard nav, ARIA, contrast, reduced motion

### Performance (Phase 6)
- Read `references/performance-patterns.md`
- CWV targets, image optimization, font loading, bundle size

### SEO (Phase 12)
- Read `references/seo-geo.md`
- Meta tags, JSON-LD, sitemap, robots.txt, OG images

### Compliance (Phase 10 + Legal)
- Read `references/legal-gdpr.md` and `references/tracking-pixels.md`
- Cookie consent + Consent Mode v2?
- Impressum + Datenschutz (if German market)?
- GDPR data handling?

### DevOps
- Read `references/ci-cd.md` and `references/git-workflow.md`
- CI/CD pipeline present?
- Environment config (.env.example)?
- README.md complete?

## Output

Create a structured report with:
1. **Score per area** (1-10)
2. **Overall score** (weighted average)
3. **Critical issues** (must fix)
4. **Improvements** (should fix)
5. **Nice-to-haves** (could fix)
6. **Action plan** (prioritized list of fixes)
