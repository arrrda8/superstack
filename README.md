# Superstack

A Claude Code skill for building production-ready, full-stack web products with award-winning design, expert copywriting, IT security, testing, CI/CD, accessibility, and auto-deployment.

## What it does

Superstack turns Claude into a senior full-stack developer, creative director, security engineer, and DevOps specialist — all in one. It enforces a structured 12-phase workflow:

| Phase | What happens |
|---|---|
| **0 — Requirements** | Mandatory. Gathers ALL requirements upfront in one session: product, audience, tech stack, SaaS features, hosting credentials, environments, compliance, design — so the agent can work through everything without interruption. |
| **1 — Copywriting** | Researches competitors and audience, maps pain points, crafts conversion-focused copy (PAS framework, headline formulas, contextual CTAs). |
| **2 — Design System** | Visual direction inspired by Awwwards/Dribbble/Codrops. Framer Motion + GSAP. Custom icons (no generic AI icon sets). Responsive. Dark mode. |
| **3 — Full Scaffold** | Complete production-ready project. Responsive, SEO'd, accessible. Includes auth (+ social login), DB with seed data, file uploads, search, email templates, i18n, PWA, multi-environment config. |
| **4 — IT Security** | Continuous auditing + full sweep: XSS, CSRF, IDOR, injection, auth hardening, security headers, dependency audit. |
| **5 — Accessibility** | WCAG 2.1 AA compliance: keyboard navigation, screen reader support, focus management, color contrast, reduced motion, touch targets. |
| **6 — Performance** | Core Web Vitals: LCP < 2.5s, CLS < 0.1, INP < 200ms. Image, font, JS, caching optimization. |
| **7 — Testing** | Vitest unit tests, Playwright E2E tests, API tests — generated with the scaffold. |
| **8 — CI/CD** | GitHub Actions pipeline (lint, type-check, test, build, E2E). Multi-environment workflows (DEV/TEST/PROD). |
| **9 — Analytics** | Sentry error tracking, privacy-friendly analytics (Plausible/Umami/PostHog), uptime monitoring, structured logging. |
| **10 — SEO** | Technical SEO, Schema markup, Open Graph, sitemaps, canonical URLs, hreflang, content SEO. |
| **11 — Deployment** | If credentials provided: auto-deploys to Vercel, Dokploy, VPS, Docker. Otherwise: step-by-step deployment docs. |

## Key features

- **All questions upfront** — Gathers every requirement in Phase 0 so the agent can work autonomously through all phases without interruption.
- **SaaS-aware** — Proactively asks about admin panels, affiliate programs, onboarding, notifications, roles, multi-tenancy, billing, API access, audit logs, and more.
- **Anti-AI design** — No generic gradients, grids, or default icon sets. Custom typography, curated icons, award-winning layout patterns.
- **Fully responsive** — Every component tested at 375px, 768px, 1024px, 1440px.
- **Real copywriting** — Pain-point research, competitor analysis, conversion-focused messaging.
- **Security-first** — Full vulnerability sweep before delivery.
- **Accessible** — WCAG 2.1 AA: keyboard nav, screen readers, focus management, reduced motion.
- **Multi-environment** — DEV/TEST/PROD configs, separate secrets, separate deployments.
- **Auto-deploy** — Give it your hosting credentials and it handles deployment end-to-end.
- **Email templates** — Transactional emails (welcome, reset, invoice) built with React Email/MJML.
- **Dark mode, i18n, PWA** — All asked about during brainstorming, all built into the scaffold when needed.

## Installation

### Option 1: Install from `.skill` file

```bash
claude install-skill superstack.skill
```

### Option 2: Clone this repo

```bash
git clone https://github.com/YOUR_USERNAME/superstack-skill.git
cp -r superstack-skill/superstack ~/.claude/skills/superstack
```

### Option 3: Manual

Copy `superstack/` (with `SKILL.md` + `references/`) to `~/.claude/skills/superstack/` (global) or `your-repo/.claude/skills/superstack/` (project-local).

## File structure

```
superstack/
├── SKILL.md                            # Main skill (12 phases)
└── references/
    ├── design-patterns.md              # CSS, Framer Motion, GSAP, layout recipes
    ├── copywriting-framework.md        # Research, headlines, section guide
    ├── security-checklist.md           # Vulnerabilities, fixes, hardening
    ├── testing-patterns.md             # Vitest, Playwright, API test recipes
    └── saas-features.md               # Admin, affiliate, onboarding, roles, billing
```

## Usage

The skill triggers automatically when you ask Claude to build a web project:

- *"Build me a landing page for my SaaS product"*
- *"Create a portfolio website for my architecture firm"*
- *"I need a customer dashboard with auth, billing, and GDPR compliance"*
- *"Make a full-stack MVP for my startup idea"*
- *"Build and deploy a fintech SaaS to my Vercel account"*

## License

MIT
