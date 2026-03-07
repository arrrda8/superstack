# Scaffolding Templates

Use these as starting points based on project type. Adapt to user requirements.

## SaaS Template

Auth, dashboard, billing (Stripe), team management, settings, onboarding wizard, admin panel, landing page with pricing.

**Key packages**: Better Auth, Stripe, Drizzle, Zustand, TanStack Query, React Hook Form + Zod

**Route structure**:
```
app/
├── (marketing)/          # Public: landing, pricing, blog
├── (auth)/               # Login, signup, reset
├── (dashboard)/          # Authenticated: dashboard, settings, billing
│   └── @modal/           # Parallel route for modals
└── api/
    └── webhooks/         # Stripe, etc.
```

## Landing Page Template

Hero, pain/solution sections, social proof, features, pricing, FAQ, CTA, footer. Lenis + GSAP scroll storytelling.

**Key packages**: Lenis, GSAP, Framer Motion, React Hook Form + Zod (contact form)

## E-Commerce Template

Product catalog, cart, checkout (Stripe), order management, inventory, customer accounts, admin panel.

**Key packages**: Stripe, Supabase Storage (product images), Zustand (cart state)

## Internal Tool / Dashboard Template

Data tables, charts, filters, CRUD operations, role-based access, export functionality.

**Key packages**: TanStack Table, Recharts/Tremor, React Hook Form + Zod, nuqs (URL state for filters)

## Portfolio Template

Project showcase, about, contact form, blog (MDX), case studies. Heavy on animations and visual storytelling.

**Key packages**: MDX, Lenis, GSAP, Framer Motion

## Common Files Across All Templates

```
next.config.ts
tailwind.config.ts / app/globals.css (@theme)
tsconfig.json
biome.json
.env.example
app/layout.tsx
app/page.tsx
app/not-found.tsx
components/ui/*
lib/utils.ts
public/favicon.ico, robots.txt
```
