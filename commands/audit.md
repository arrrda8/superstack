---
description: Audit an existing web project for security, accessibility, performance, and SEO issues
argument-hint: [optional focus area: security | a11y | performance | seo | all]
---

You are now using the **Superstack** skill for auditing an existing project. Read the skill file at `skills/superstack/SKILL.md` for quality standards and reference files.

Run a comprehensive audit of the current project. If the user specified a focus area: "$ARGUMENTS" — prioritize that area. Otherwise run all four audits.

## Audit Phases

1. **Security Audit** (Phase 4) — Read `references/security-checklist.md`. Check for XSS, CSRF, IDOR, SQL injection, auth flaws, env leaks, dependency vulnerabilities, webhook verification.

2. **Accessibility Audit** (Phase 5) — Read `references/accessibility.md`. Check WCAG 2.1 AA: semantic HTML, keyboard navigation, ARIA, color contrast, reduced motion, touch targets.

3. **Performance Audit** (Phase 6) — Read `references/performance-patterns.md`. Check CWV targets (LCP < 2.5s, CLS < 0.1, INP < 200ms), image optimization, font loading, bundle size, caching.

4. **SEO Audit** (Phase 12) — Read `references/seo-geo.md`. Check meta tags, JSON-LD, sitemap, robots.txt, canonical URLs, OG images, Core Web Vitals.

Use parallel agents for independent audit areas. Output a summary table with severity ratings (Critical/High/Medium/Low) and specific fix recommendations with code examples.
