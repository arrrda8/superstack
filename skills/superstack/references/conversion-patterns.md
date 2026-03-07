# Conversion Optimization Patterns

## Table of Contents
- [1. Social Proof](#1-social-proof)
  - [Testimonial card](#testimonial-card)
  - [Logo bar](#logo-bar)
  - [Aggregate review widget](#aggregate-review-widget)
  - [Live user count / activity indicator](#live-user-count--activity-indicator)
- [2. Trust Signals](#2-trust-signals)
  - [Security badges row](#security-badges-row)
  - [Press mentions / "As seen in"](#press-mentions--as-seen-in)
  - [Money-back guarantee badge](#money-back-guarantee-badge)
- [3. Pricing Psychology](#3-pricing-psychology)
  - [Anchoring — show expensive first](#anchoring--show-expensive-first)
  - [Decoy pricing](#decoy-pricing)
  - [Annual vs monthly toggle](#annual-vs-monthly-toggle)
  - [Pricing card with "Most popular" badge + anchoring](#pricing-card-with-most-popular-badge--anchoring)
- [4. CTA Optimization](#4-cta-optimization)
  - [Button copy principles](#button-copy-principles)
  - [Good vs bad CTA examples](#good-vs-bad-cta-examples)
  - [Above and below fold placement](#above-and-below-fold-placement)
  - [Ethical urgency](#ethical-urgency)
- [5. Landing Page Structure](#5-landing-page-structure)
  - [Proven high-converting layout](#proven-high-converting-layout)
  - [Hero section template](#hero-section-template)
  - [Section component pattern](#section-component-pattern)
  - [FAQ section (objection handling)](#faq-section-objection-handling)
- [6. Exit Intent](#6-exit-intent)
  - [Mouse leave detection hook](#mouse-leave-detection-hook)
  - [Exit intent popup](#exit-intent-popup)
- [7. Scarcity & Urgency](#7-scarcity--urgency)
  - [Ethical guidelines](#ethical-guidelines)
  - [Real countdown timer](#real-countdown-timer)
  - [Limited spots (real capacity)](#limited-spots-real-capacity)
  - [Stock indicator (e-commerce, real data only)](#stock-indicator-e-commerce-real-data-only)
- [8. Form Optimization](#8-form-optimization)
  - [Reduce fields — minimal form](#reduce-fields--minimal-form)
  - [Inline validation](#inline-validation)
  - [Multi-step form with progress bar](#multi-step-form-with-progress-bar)
  - [Social login shortcut](#social-login-shortcut)
- [9. Micro-Copy](#9-micro-copy)
  - [Button labels](#button-labels)
  - [Form placeholders](#form-placeholders)
  - [Error messages](#error-messages)
  - [Empty states for conversion](#empty-states-for-conversion)
- [10. A/B Testing](#10-ab-testing)
  - [Vercel Toolbar / Flags SDK](#vercel-toolbar--flags-sdk)
  - [Cookie-based split via middleware](#cookie-based-split-via-middleware)
  - [PostHog experiments](#posthog-experiments)
  - [What to test first (priority order)](#what-to-test-first-priority-order)
  - [Statistical significance](#statistical-significance)
- [Quick Reference: Conversion Checklist](#quick-reference-conversion-checklist)

Reference for conversion rate optimization patterns, UI components, and psychological triggers for web apps and landing pages.

> **Ethics note:** Scarcity and urgency tactics should only reflect real constraints. Fake countdown timers, fabricated stock levels, or manufactured urgency erode trust and can violate consumer protection laws in the EU (UWG, Omnibus Directive) and other jurisdictions. Only use these patterns when the underlying data is genuine.

---

## 1. Social Proof

### Testimonial card

```tsx
interface Testimonial {
  quote: string;
  name: string;
  role: string;
  company: string;
  avatarUrl: string;
  rating?: number;
}

function TestimonialCard({ testimonial }: { testimonial: Testimonial }) {
  return (
    <figure className="rounded-2xl border border-border bg-card p-6 shadow-sm">
      {testimonial.rating && (
        <div className="mb-3 flex gap-0.5">
          {Array.from({ length: 5 }).map((_, i) => (
            <Star
              key={i}
              className={cn(
                "h-4 w-4",
                i < testimonial.rating!
                  ? "fill-amber-400 text-amber-400"
                  : "text-muted-foreground"
              )}
            />
          ))}
        </div>
      )}
      <blockquote className="text-sm leading-relaxed text-foreground">
        &ldquo;{testimonial.quote}&rdquo;
      </blockquote>
      <figcaption className="mt-4 flex items-center gap-3">
        <Image
          src={testimonial.avatarUrl}
          alt={testimonial.name}
          width={40}
          height={40}
          className="rounded-full object-cover"
        />
        <div>
          <p className="text-sm font-semibold text-foreground">
            {testimonial.name}
          </p>
          <p className="text-xs text-muted-foreground">
            {testimonial.role}, {testimonial.company}
          </p>
        </div>
      </figcaption>
    </figure>
  );
}
```

### Logo bar

```tsx
function LogoBar({ logos }: { logos: { src: string; alt: string }[] }) {
  return (
    <section className="py-12 border-y bg-muted/30">
      <p className="mb-8 text-center text-sm font-medium text-muted-foreground">
        Trusted by teams at
      </p>
      <div className="flex flex-wrap items-center justify-center gap-x-12 gap-y-6 px-4">
        {logos.map((logo) => (
          <Image
            key={logo.alt}
            src={logo.src}
            alt={logo.alt}
            width={120}
            height={40}
            className="h-8 w-auto opacity-60 grayscale transition-all hover:opacity-100 hover:grayscale-0"
          />
        ))}
      </div>
    </section>
  );
}
```

### Aggregate review widget

```tsx
function ReviewSummary({
  average,
  total,
}: {
  average: number;
  total: number;
}) {
  return (
    <div className="flex items-center gap-2">
      <div className="flex gap-0.5">
        {Array.from({ length: 5 }).map((_, i) => (
          <Star
            key={i}
            className={cn(
              "h-5 w-5",
              i < Math.round(average)
                ? "fill-amber-400 text-amber-400"
                : "text-gray-300"
            )}
          />
        ))}
      </div>
      <span className="font-semibold">{average.toFixed(1)}</span>
      <span className="text-muted-foreground text-sm">
        ({total.toLocaleString()} reviews)
      </span>
    </div>
  );
}
```

### Live user count / activity indicator

```tsx
"use client";

import { useEffect, useState } from "react";

function LiveViewers() {
  const [count, setCount] = useState<number | null>(null);

  useEffect(() => {
    // Fetch real count from your analytics or presence API
    async function fetchCount() {
      const res = await fetch("/api/active-viewers");
      const data = await res.json();
      setCount(data.count);
    }
    fetchCount();
    const interval = setInterval(fetchCount, 30_000);
    return () => clearInterval(interval);
  }, []);

  if (!count) return null;

  return (
    <div className="inline-flex items-center gap-2 rounded-full bg-emerald-50 px-3 py-1 text-sm text-emerald-700 dark:bg-emerald-950 dark:text-emerald-300">
      <span className="relative flex h-2 w-2">
        <span className="absolute inline-flex h-full w-full animate-ping rounded-full bg-emerald-400 opacity-75" />
        <span className="relative inline-flex h-2 w-2 rounded-full bg-emerald-500" />
      </span>
      {count} people viewing this right now
    </div>
  );
}
```

> **Note:** Only show real-time counts if backed by actual data (analytics, WebSocket connections). Fabricating numbers is deceptive.

---

## 2. Trust Signals

### Security badges row

```tsx
import { ShieldCheck, RefreshCw, Lock, Award } from "lucide-react";

function TrustBadges() {
  const badges = [
    { icon: ShieldCheck, label: "256-bit SSL" },
    { icon: Lock, label: "GDPR compliant" },
    { icon: RefreshCw, label: "30-day money-back" },
    { icon: Award, label: "ISO 27001 certified" },
  ];

  return (
    <div className="flex flex-wrap items-center justify-center gap-6 py-6 text-sm text-muted-foreground">
      {badges.map(({ icon: Icon, label }) => (
        <div key={label} className="flex items-center gap-2">
          <Icon className="h-5 w-5 text-emerald-600" />
          <span>{label}</span>
        </div>
      ))}
    </div>
  );
}
```

### Press mentions / "As seen in"

```tsx
function PressBar() {
  return (
    <section className="border-y border-border bg-muted/30 py-8">
      <p className="mb-6 text-center text-xs font-semibold uppercase tracking-widest text-muted-foreground">
        As featured in
      </p>
      <div className="flex flex-wrap items-center justify-center gap-10 px-4 opacity-50">
        <Image src="/press/techcrunch.svg" alt="TechCrunch" width={130} height={30} />
        <Image src="/press/forbes.svg" alt="Forbes" width={80} height={30} />
        <Image src="/press/wired.svg" alt="Wired" width={80} height={30} />
      </div>
    </section>
  );
}
```

### Money-back guarantee badge

```tsx
function MoneyBackGuarantee() {
  return (
    <div className="mx-auto max-w-sm rounded-xl border-2 border-emerald-200 bg-emerald-50 p-6 text-center dark:border-emerald-800 dark:bg-emerald-950">
      <div className="mx-auto mb-3 flex h-12 w-12 items-center justify-center rounded-full bg-emerald-100 dark:bg-emerald-900">
        <ShieldCheck className="h-6 w-6 text-emerald-600" />
      </div>
      <h3 className="text-base font-semibold text-foreground">
        30-Day Money-Back Guarantee
      </h3>
      <p className="mt-1 text-sm text-muted-foreground">
        Try it risk-free. If you are not satisfied, get a full refund — no
        questions asked.
      </p>
    </div>
  );
}
```

---

## 3. Pricing Psychology

### Anchoring — show expensive first

Place the highest-priced plan on the left (or first in the list) so that all subsequent plans feel affordable by comparison.

### Decoy pricing

Introduce a mid-tier plan that makes the target plan look like better value:

| Plan | Price | Features |
|---|---|---|
| Basic | $9/mo | 1 project, 1 user |
| Pro (decoy) | $29/mo | 5 projects, 3 users |
| Business (target) | $39/mo | Unlimited projects, 10 users |

The $10 gap between Pro and Business — with a massive feature jump — makes Business the obvious choice.

### Annual vs monthly toggle

```tsx
"use client";

import { useState } from "react";

function PricingToggle({
  onToggle,
}: {
  onToggle: (annual: boolean) => void;
}) {
  const [isAnnual, setIsAnnual] = useState(true);

  function toggle() {
    setIsAnnual(!isAnnual);
    onToggle(!isAnnual);
  }

  return (
    <div className="flex items-center justify-center gap-3">
      <span
        className={cn("text-sm", !isAnnual ? "font-semibold text-foreground" : "text-muted-foreground")}
      >
        Monthly
      </span>
      <button
        role="switch"
        aria-checked={isAnnual}
        onClick={toggle}
        className={cn(
          "relative h-7 w-12 rounded-full transition-colors",
          isAnnual ? "bg-primary" : "bg-muted"
        )}
      >
        <span
          className={cn(
            "absolute top-0.5 h-6 w-6 rounded-full bg-white shadow transition-transform",
            isAnnual ? "translate-x-5" : "translate-x-0.5"
          )}
        />
      </button>
      <span
        className={cn("text-sm", isAnnual ? "font-semibold text-foreground" : "text-muted-foreground")}
      >
        Annual
        <span className="ml-1.5 rounded-full bg-emerald-100 px-2 py-0.5 text-xs font-medium text-emerald-700 dark:bg-emerald-900 dark:text-emerald-300">
          Save 20%
        </span>
      </span>
    </div>
  );
}
```

### Pricing card with "Most popular" badge + anchoring

```tsx
interface PricingPlan {
  name: string;
  price: number;
  originalPrice?: number;
  period: string;
  features: string[];
  cta: string;
  popular?: boolean;
}

function PricingCard({ plan }: { plan: PricingPlan }) {
  return (
    <div
      className={cn(
        "relative rounded-2xl border p-8 flex flex-col",
        plan.popular
          ? "border-primary shadow-lg shadow-primary/10 ring-1 ring-primary scale-105"
          : "border-border"
      )}
    >
      {plan.popular && (
        <div className="absolute -top-3 left-1/2 -translate-x-1/2 rounded-full bg-primary px-4 py-1 text-xs font-semibold text-primary-foreground">
          Most popular
        </div>
      )}
      <h3 className="text-lg font-semibold">{plan.name}</h3>
      <div className="mt-4 flex items-baseline gap-2">
        {plan.originalPrice && (
          <span className="text-lg text-muted-foreground line-through">
            ${plan.originalPrice}
          </span>
        )}
        <span className="text-4xl font-bold">${plan.price}</span>
        <span className="text-muted-foreground">/{plan.period}</span>
      </div>
      <ul className="mt-6 space-y-3 flex-1">
        {plan.features.map((f) => (
          <li key={f} className="flex items-center gap-2 text-sm">
            <Check className="h-4 w-4 text-emerald-500 shrink-0" />
            {f}
          </li>
        ))}
      </ul>
      <Button
        className="mt-8 w-full"
        variant={plan.popular ? "default" : "outline"}
      >
        {plan.cta}
      </Button>
    </div>
  );
}
```

---

## 4. CTA Optimization

### Button copy principles

- **Action-oriented**: "Start free trial" not "Submit"
- **First person**: "Start my free trial" outperforms "Start your free trial" in many tests
- **Reduce risk**: "Try free for 14 days" > "Sign up" — no commitment language
- **Specific outcome**: "Get my report" > "Download"

### Good vs bad CTA examples

| Bad | Good | Why |
|---|---|---|
| Submit | Create my account | Specific action |
| Download | Get the free guide | States the value |
| Buy now | Start my 14-day trial | Reduces friction |
| Learn more | See how it works | Sets expectation |
| Click here | Explore pricing | Meaningful action |

### Above and below fold placement

```tsx
function HeroSection() {
  return (
    <section className="py-20">
      <h1 className="text-5xl font-bold tracking-tight">
        Ship faster with less code
      </h1>
      <p className="mt-4 max-w-lg text-lg text-muted-foreground">
        The full-stack framework that turns weeks of work into hours.
      </p>
      {/* Primary CTA — above the fold */}
      <div className="mt-8 flex flex-col sm:flex-row gap-3">
        <Button size="lg" className="text-base px-8">
          Start building free
        </Button>
        <Button size="lg" variant="outline" className="text-base">
          See a demo
        </Button>
      </div>
      {/* Trust micro-text */}
      <p className="mt-4 text-xs text-muted-foreground">
        No credit card required. Setup in 5 minutes.
      </p>
    </section>
  );
}

// Sticky bottom CTA for mobile — appears when scrolled past hero CTA
function StickyBottomCTA() {
  return (
    <div className="fixed bottom-0 inset-x-0 z-50 border-t border-border bg-background/80 backdrop-blur-md p-3 md:hidden">
      <Button className="w-full" size="lg">
        Start building free
      </Button>
    </div>
  );
}
```

### Ethical urgency

Only use urgency if it reflects reality (real deadline, real limited offer).

```tsx
"use client";

import { useEffect, useState } from "react";

function OfferBanner({ expiresAt }: { expiresAt: Date }) {
  const [timeLeft, setTimeLeft] = useState("");

  useEffect(() => {
    function update() {
      const diff = expiresAt.getTime() - Date.now();
      if (diff <= 0) {
        setTimeLeft("Expired");
        return;
      }
      const h = Math.floor(diff / 3_600_000);
      const m = Math.floor((diff % 3_600_000) / 60_000);
      const s = Math.floor((diff % 60_000) / 1000);
      setTimeLeft(`${h}h ${m}m ${s}s`);
    }
    update();
    const id = setInterval(update, 1000);
    return () => clearInterval(id);
  }, [expiresAt]);

  if (timeLeft === "Expired") return null;

  return (
    <div className="bg-primary text-primary-foreground text-center py-2 text-sm font-medium">
      Early-bird pricing ends in <span className="font-mono">{timeLeft}</span>{" "}
      — Save 40%
    </div>
  );
}
```

```tsx
// Good: real deadline
<p className="text-sm text-muted-foreground">
  Early-bird pricing ends March 15. Regular price: $99/mo.
</p>

// Bad: fake urgency
// "Only 2 spots left!" (resets every page load)
// "This offer expires in 00:14:32" (fake timer)
```

---

## 5. Landing Page Structure

### Proven high-converting layout

```
1. HERO
   - Headline (pain or outcome)
   - Subheadline (how you solve it)
   - Primary CTA
   - Hero image / demo video
   - Social proof snippet (logo bar or "Trusted by 500+ teams")

2. PAIN / PROBLEM
   - 3 pain points the audience recognizes
   - "Sound familiar?" empathy copy

3. SOLUTION
   - How your product solves the pain
   - 3 key features with icons
   - Screenshot or product walkthrough

4. SOCIAL PROOF
   - 3 testimonials (ideally with photos and titles)
   - Case study numbers ("Reduced churn by 34%")

5. HOW IT WORKS
   - 3-step process (simple)
   - Visual timeline or numbered cards

6. PRICING
   - 2-3 plans (no more)
   - Annual/monthly toggle
   - Feature comparison table

7. FAQ
   - 5-8 common objections as questions
   - Address pricing, security, migration, support

8. FINAL CTA
   - Repeat the hero CTA
   - Add a guarantee or risk-reversal statement
   - "Still not sure? Book a 15-min call"
```

### Hero section template

```tsx
function Hero() {
  return (
    <section className="relative overflow-hidden py-20 md:py-32">
      <div className="container mx-auto px-4 text-center">
        {/* Social proof micro-badge */}
        <div className="inline-flex items-center gap-2 rounded-full border px-4 py-1.5 text-sm mb-6">
          <span className="flex -space-x-1">
            {[1, 2, 3].map((i) => (
              <Image
                key={i}
                src={`/avatars/${i}.jpg`}
                alt=""
                width={24}
                height={24}
                className="rounded-full border-2 border-background"
              />
            ))}
          </span>
          <span className="text-muted-foreground">
            Joined by 2,400+ marketers
          </span>
        </div>

        <h1 className="text-4xl md:text-6xl font-bold tracking-tight max-w-3xl mx-auto">
          Turn Ad Spend Into{" "}
          <span className="text-primary">Predictable Revenue</span>
        </h1>
        <p className="mt-6 text-lg text-muted-foreground max-w-xl mx-auto">
          AI-powered marketing intelligence that shows you exactly where to
          optimize — so you scale what works and cut what doesn't.
        </p>

        <div className="mt-8 flex flex-col sm:flex-row items-center justify-center gap-4">
          <Button size="lg" className="text-base px-8">
            Start Free Trial
          </Button>
          <Button size="lg" variant="outline" className="text-base">
            Watch 2-Min Demo
          </Button>
        </div>

        <p className="mt-4 text-xs text-muted-foreground">
          No credit card required. Setup in 5 minutes.
        </p>
      </div>
    </section>
  );
}
```

### Section component pattern

```tsx
function Section({
  id,
  children,
  className,
}: {
  id?: string;
  children: React.ReactNode;
  className?: string;
}) {
  return (
    <section
      id={id}
      className={cn("mx-auto max-w-6xl px-4 py-20 sm:px-6 lg:px-8", className)}
    >
      {children}
    </section>
  );
}

function SectionHeader({
  badge,
  title,
  description,
}: {
  badge?: string;
  title: string;
  description: string;
}) {
  return (
    <div className="mx-auto mb-12 max-w-2xl text-center">
      {badge && (
        <span className="mb-3 inline-block rounded-full bg-primary/10 px-3 py-1 text-xs font-medium text-primary">
          {badge}
        </span>
      )}
      <h2 className="text-3xl font-bold tracking-tight sm:text-4xl">{title}</h2>
      <p className="mt-3 text-lg text-muted-foreground">{description}</p>
    </div>
  );
}
```

### FAQ section (objection handling)

```tsx
import {
  Accordion,
  AccordionContent,
  AccordionItem,
  AccordionTrigger,
} from "@/components/ui/accordion";

const faqs = [
  {
    q: "Can I cancel anytime?",
    a: "Yes. Cancel with one click, no questions asked. You keep access until the end of your billing period.",
  },
  {
    q: "Is my data secure?",
    a: "We use AES-256 encryption, are SOC 2 compliant, and never sell your data to third parties.",
  },
  {
    q: "How long does setup take?",
    a: "Most teams are up and running in under 10 minutes. Our onboarding wizard connects your ad accounts automatically.",
  },
];

function FAQSection() {
  return (
    <Section>
      <SectionHeader
        title="Frequently Asked Questions"
        description="Everything you need to know before getting started."
      />
      <div className="mx-auto max-w-2xl">
        <Accordion type="single" collapsible>
          {faqs.map((faq, i) => (
            <AccordionItem key={i} value={`faq-${i}`}>
              <AccordionTrigger>{faq.q}</AccordionTrigger>
              <AccordionContent>{faq.a}</AccordionContent>
            </AccordionItem>
          ))}
        </Accordion>
      </div>
    </Section>
  );
}
```

---

## 6. Exit Intent

### Mouse leave detection hook

```tsx
"use client";

import { useEffect, useState, useCallback } from "react";

function useExitIntent() {
  const [showPopup, setShowPopup] = useState(false);
  const [dismissed, setDismissed] = useState(false);

  const handleMouseLeave = useCallback(
    (e: MouseEvent) => {
      // Only trigger when mouse moves to top of viewport (closing tab / leaving)
      if (e.clientY <= 0 && !dismissed) {
        setShowPopup(true);
      }
    },
    [dismissed]
  );

  useEffect(() => {
    // Don't show on mobile (no mouse leave)
    if (window.matchMedia("(pointer: coarse)").matches) return;
    // Don't show if already dismissed this session
    if (sessionStorage.getItem("exit-intent-dismissed")) return;

    document.addEventListener("mouseleave", handleMouseLeave);
    return () => document.removeEventListener("mouseleave", handleMouseLeave);
  }, [handleMouseLeave]);

  const dismiss = () => {
    setShowPopup(false);
    setDismissed(true);
    sessionStorage.setItem("exit-intent-dismissed", "1");
  };

  return { showPopup, dismiss };
}
```

### Exit intent popup

```tsx
import {
  Dialog,
  DialogContent,
  DialogHeader,
  DialogTitle,
  DialogDescription,
} from "@/components/ui/dialog";

function ExitIntentPopup() {
  const { showPopup, dismiss } = useExitIntent();

  return (
    <Dialog open={showPopup} onOpenChange={(open) => !open && dismiss()}>
      <DialogContent className="sm:max-w-md">
        <DialogHeader>
          <DialogTitle>Wait — before you go!</DialogTitle>
          <DialogDescription>
            Get our free Marketing ROI Calculator and see exactly where your
            budget is leaking.
          </DialogDescription>
        </DialogHeader>
        <form
          onSubmit={async (e) => {
            e.preventDefault();
            const email = new FormData(e.currentTarget).get("email") as string;
            await fetch("/api/lead", {
              method: "POST",
              body: JSON.stringify({ email, source: "exit-intent" }),
            });
            dismiss();
          }}
          className="flex gap-2 mt-4"
        >
          <Input
            name="email"
            type="email"
            required
            placeholder="you@company.com"
            className="flex-1"
          />
          <Button type="submit">Get Free Calculator</Button>
        </form>
        <p className="text-xs text-muted-foreground mt-2">
          No spam. Unsubscribe anytime.
        </p>
      </DialogContent>
    </Dialog>
  );
}
```

**Best practices for exit intent:**
- Show only once per session (use `sessionStorage` or `shown` state).
- Offer genuine value (lead magnet, discount), not just "don't leave".
- On mobile, use scroll-up detection or time-on-page delay instead of mouse-leave.
- Respect user intent — provide a clear close button.

---

## 7. Scarcity & Urgency

### Ethical guidelines

> **Rule: Only use scarcity and urgency when they reflect reality.**
>
> - Countdown timers must reference a real deadline (launch date, cohort close, sale end).
> - "Limited spots" must be actually limited (e.g., consulting capacity, cohort size).
> - Stock indicators must reflect real inventory from your database.
> - Fake urgency destroys trust and can violate consumer protection laws (EU Omnibus Directive, German UWG).
> - If the "offer" resets every day, it is not a real offer.

### Real countdown timer

```tsx
"use client";

import { useEffect, useState } from "react";

function calcTimeLeft(deadline: Date) {
  const total = deadline.getTime() - Date.now();
  return {
    total,
    days: Math.max(0, Math.floor(total / (1000 * 60 * 60 * 24))),
    hours: Math.max(0, Math.floor((total / (1000 * 60 * 60)) % 24)),
    minutes: Math.max(0, Math.floor((total / (1000 * 60)) % 60)),
    seconds: Math.max(0, Math.floor((total / 1000) % 60)),
  };
}

function CountdownTimer({ deadline }: { deadline: Date }) {
  const [timeLeft, setTimeLeft] = useState(calcTimeLeft(deadline));

  useEffect(() => {
    const timer = setInterval(() => {
      const remaining = calcTimeLeft(deadline);
      setTimeLeft(remaining);
      if (remaining.total <= 0) clearInterval(timer);
    }, 1000);
    return () => clearInterval(timer);
  }, [deadline]);

  if (timeLeft.total <= 0) return null;

  return (
    <div className="flex items-center justify-center gap-3">
      {(["days", "hours", "minutes", "seconds"] as const).map((unit) => (
        <div key={unit} className="text-center">
          <div className="rounded-lg bg-foreground px-3 py-2 text-2xl font-bold tabular-nums text-background">
            {String(timeLeft[unit]).padStart(2, "0")}
          </div>
          <p className="mt-1 text-xs text-muted-foreground">{unit}</p>
        </div>
      ))}
    </div>
  );
}
```

### Limited spots (real capacity)

```tsx
function LimitedSpots({
  total,
  taken,
}: {
  total: number;
  taken: number;
}) {
  const remaining = total - taken;
  const percentage = (taken / total) * 100;

  if (remaining <= 0) {
    return (
      <p className="text-sm font-medium text-destructive">
        All spots are taken. Join the waitlist.
      </p>
    );
  }

  return (
    <div className="space-y-2">
      <div className="h-2 w-full overflow-hidden rounded-full bg-muted">
        <div
          className={cn(
            "h-full rounded-full transition-all",
            remaining <= total * 0.2 ? "bg-amber-500" : "bg-primary"
          )}
          style={{ width: `${percentage}%` }}
        />
      </div>
      <p className="text-sm text-muted-foreground">
        <span className="font-semibold text-foreground">{remaining}</span> of{" "}
        {total} spots remaining
      </p>
    </div>
  );
}
```

### Stock indicator (e-commerce, real data only)

```tsx
function StockIndicator({ stock }: { stock: number }) {
  if (stock <= 0) {
    return <p className="text-sm font-medium text-destructive">Out of stock</p>;
  }
  if (stock <= 5) {
    return (
      <p className="text-sm font-medium text-amber-600">
        Only {stock} left in stock
      </p>
    );
  }
  return <p className="text-sm text-emerald-600">In stock</p>;
}
```

---

## 8. Form Optimization

### Reduce fields — minimal form

```tsx
// Bad: 8 fields upfront
// Good: 1-3 fields (email, name) — ask the rest during onboarding

function MinimalSignupForm() {
  return (
    <form className="space-y-4 max-w-sm">
      <div>
        <Label htmlFor="email">Work email</Label>
        <Input
          id="email"
          type="email"
          required
          placeholder="you@company.com"
          autoFocus
        />
      </div>
      <Button type="submit" className="w-full">
        Start free trial
      </Button>
      <p className="text-center text-xs text-muted-foreground">
        No credit card required. Setup in 2 minutes.
      </p>
    </form>
  );
}
```

### Inline validation

```tsx
"use client";

import { useState } from "react";

function ValidatedInput({
  label,
  type = "text",
  validate,
  ...props
}: {
  label: string;
  type?: string;
  validate: (value: string) => string | null;
} & React.InputHTMLAttributes<HTMLInputElement>) {
  const [value, setValue] = useState("");
  const [error, setError] = useState<string | null>(null);
  const [touched, setTouched] = useState(false);

  const handleBlur = () => {
    setTouched(true);
    setError(validate(value));
  };

  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    setValue(e.target.value);
    if (touched) setError(validate(e.target.value));
  };

  return (
    <div>
      <Label>{label}</Label>
      <Input
        type={type}
        value={value}
        onChange={handleChange}
        onBlur={handleBlur}
        className={cn(
          touched && error && "border-destructive focus-visible:ring-destructive"
        )}
        {...props}
      />
      {error && touched && (
        <p className="mt-1 text-xs text-destructive">{error}</p>
      )}
    </div>
  );
}

// Usage
<ValidatedInput
  label="Email"
  type="email"
  placeholder="you@company.com"
  validate={(v) =>
    /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(v) ? null : "Please enter a valid email (e.g. you@company.com)"
  }
/>
```

### Multi-step form with progress bar

```tsx
"use client";

import { useState } from "react";

function MultiStepForm() {
  const [step, setStep] = useState(0);
  const steps = ["Account", "Company", "Goals"];

  return (
    <div className="max-w-md mx-auto space-y-6">
      {/* Progress bar */}
      <div className="space-y-2">
        <div className="flex justify-between text-xs text-muted-foreground">
          {steps.map((s, i) => (
            <span
              key={s}
              className={cn(i <= step && "text-foreground font-medium")}
            >
              {s}
            </span>
          ))}
        </div>
        <div className="h-1.5 rounded-full bg-muted overflow-hidden">
          <div
            className="h-full rounded-full bg-primary transition-all duration-300"
            style={{ width: `${((step + 1) / steps.length) * 100}%` }}
          />
        </div>
      </div>

      {/* Step content */}
      {step === 0 && (
        <div className="space-y-3">
          <Input placeholder="Full name" />
          <Input placeholder="Work email" type="email" />
        </div>
      )}
      {step === 1 && (
        <div className="space-y-3">
          <Input placeholder="Company name" />
          <Input placeholder="Website URL" type="url" />
        </div>
      )}
      {step === 2 && (
        <div className="space-y-3">
          <p className="text-sm font-medium">What's your primary goal?</p>
          {["Increase leads", "Reduce CAC", "Scale ad spend"].map((goal) => (
            <label
              key={goal}
              className="flex items-center gap-3 rounded-lg border p-3 cursor-pointer hover:bg-muted/50"
            >
              <input type="radio" name="goal" value={goal} />
              <span className="text-sm">{goal}</span>
            </label>
          ))}
        </div>
      )}

      <div className="flex gap-3">
        {step > 0 && (
          <Button variant="outline" onClick={() => setStep(step - 1)}>
            Back
          </Button>
        )}
        <Button
          className="flex-1"
          onClick={() =>
            step < steps.length - 1 ? setStep(step + 1) : handleSubmit()
          }
        >
          {step < steps.length - 1 ? "Continue" : "Create Account"}
        </Button>
      </div>
    </div>
  );
}
```

### Social login shortcut

```tsx
function SocialLogin() {
  return (
    <div className="space-y-3">
      <Button variant="outline" className="w-full gap-2">
        <GoogleIcon className="h-5 w-5" />
        Continue with Google
      </Button>
      <Button variant="outline" className="w-full gap-2">
        <GitHubIcon className="h-5 w-5" />
        Continue with GitHub
      </Button>
      <div className="relative my-4">
        <div className="absolute inset-0 flex items-center">
          <div className="w-full border-t border-border" />
        </div>
        <div className="relative flex justify-center">
          <span className="bg-background px-3 text-xs text-muted-foreground">
            or continue with email
          </span>
        </div>
      </div>
    </div>
  );
}
```

---

## 9. Micro-Copy

Good micro-copy reduces friction, increases confidence, and nudges action.

### Button labels

| Context | Bad | Good | Why |
|---|---|---|---|
| Signup | Submit | Create my account | Specific action |
| Download | Download | Get the free guide | States the value |
| Purchase | Buy now | Start my 14-day trial | Reduces friction |
| Navigation | Learn more | See how it works | Sets expectation |
| Delete | Delete | Remove permanently | Sets expectation |
| Loading | Loading... | Saving your changes... | Reassuring |
| Success | Done | Changes saved | Clear feedback |

### Form placeholders

```tsx
// Bad: placeholder repeats the label
<Label>Email</Label>
<Input placeholder="Email" />

// Good: placeholder provides an example
<Label>Work email</Label>
<Input placeholder="you@company.com" />

// Good: placeholder sets expectations
<Label>Company website</Label>
<Input placeholder="https://example.com" />
```

### Error messages

```tsx
// Bad — vague, unhelpful
"Invalid input"
"Error"
"Field required"

// Good — explain what went wrong and how to fix it
"Please enter a valid email address (e.g., name@company.com)"
"Password must be at least 8 characters with one number"
"We need your company name to set up your workspace"
```

### Empty states for conversion

```tsx
function EmptyState({
  icon: Icon,
  title,
  description,
  action,
}: {
  icon: React.ComponentType<{ className?: string }>;
  title: string;
  description: string;
  action: { label: string; onClick: () => void };
}) {
  return (
    <div className="flex flex-col items-center justify-center py-16 text-center">
      <div className="mb-4 rounded-full bg-muted p-4">
        <Icon className="h-8 w-8 text-muted-foreground" />
      </div>
      <h3 className="text-lg font-semibold">{title}</h3>
      <p className="mt-1 max-w-sm text-sm text-muted-foreground">
        {description}
      </p>
      <Button onClick={action.onClick} className="mt-6">
        {action.label}
      </Button>
    </div>
  );
}

// Usage
<EmptyState
  icon={BarChart3}
  title="No campaigns connected yet"
  description="Connect your first ad account to start seeing real-time performance data."
  action={{ label: "Connect Ad Account", onClick: openSetup }}
/>
```

---

## 10. A/B Testing

### Vercel Toolbar / Flags SDK

Vercel's Flags SDK combined with Edge Config enables feature flags and A/B tests at the edge with zero latency.

```tsx
// flags.ts
import { flag } from "@vercel/flags/next";

export const newPricingPage = flag<boolean>({
  key: "new-pricing-page",
  decide() {
    return Math.random() > 0.5; // 50/50 split
  },
});
```

```tsx
// app/pricing/page.tsx
import { newPricingPage } from "@/flags";

export default async function PricingPage() {
  const showNewPricing = await newPricingPage();
  return showNewPricing ? <NewPricing /> : <OldPricing />;
}
```

### Cookie-based split via middleware

```ts
// middleware.ts
import { NextResponse } from "next/server";
import type { NextRequest } from "next/server";

export function middleware(request: NextRequest) {
  const response = NextResponse.next();

  // Assign experiment bucket (50/50 split)
  if (!request.cookies.get("ab-hero")) {
    const variant = Math.random() < 0.5 ? "control" : "variant";
    response.cookies.set("ab-hero", variant, {
      maxAge: 60 * 60 * 24 * 30, // 30 days
      path: "/",
    });
  }

  return response;
}
```

```tsx
// app/page.tsx
import { cookies } from "next/headers";

export default async function Home() {
  const cookieStore = await cookies();
  const heroVariant = cookieStore.get("ab-hero")?.value ?? "control";
  return heroVariant === "variant" ? <HeroVariantB /> : <HeroControl />;
}
```

### PostHog experiments

```bash
bun add posthog-js posthog-node
```

```tsx
// src/lib/posthog/provider.tsx
"use client";

import posthog from "posthog-js";
import { PostHogProvider as PHProvider } from "posthog-js/react";
import { useEffect } from "react";

export function PostHogProvider({ children }: { children: React.ReactNode }) {
  useEffect(() => {
    posthog.init(process.env.NEXT_PUBLIC_POSTHOG_KEY!, {
      api_host: process.env.NEXT_PUBLIC_POSTHOG_HOST ?? "https://eu.i.posthog.com",
      capture_pageview: false, // We handle this manually
    });
  }, []);

  return <PHProvider client={posthog}>{children}</PHProvider>;
}
```

```tsx
// Using feature flags for experiments
"use client";

import { useFeatureFlagVariantKey } from "posthog-js/react";

function HeroHeadline() {
  const variant = useFeatureFlagVariantKey("hero-headline-test");

  const headlines: Record<string, string> = {
    control: "Marketing Intelligence for Agencies",
    "benefit-led": "Turn Ad Spend Into Predictable Revenue",
    "pain-led": "Stop Wasting 30% of Your Ad Budget",
  };

  return (
    <h1 className="text-4xl md:text-6xl font-bold tracking-tight">
      {headlines[variant as string] ?? headlines.control}
    </h1>
  );
}
```

```tsx
// Track conversion events
import posthog from "posthog-js";

function handleSignup() {
  posthog.capture("signup_completed", {
    plan: "pro",
    source: "pricing_page",
  });
}
```

### What to test first (priority order)

1. **Headline** — biggest impact, easiest to test
2. **CTA copy and color** — "Start free trial" vs "Get started free"
3. **Social proof placement** — above vs below the fold
4. **Pricing structure** — 2 vs 3 tiers, monthly vs annual default
5. **Hero image vs video** — static screenshot vs product demo
6. **Form length** — 1 field (email) vs 3 fields (name + email + company)
7. **Page length** — long-form vs short-form landing page

### Statistical significance

- **Minimum sample size**: ~1,000 visitors per variant for reliable results
- **Duration**: Run tests for at least 2 full business weeks (capture weekday/weekend patterns)
- **One test at a time**: Avoid running overlapping tests on the same page
- **95% confidence**: Standard threshold — do not call a winner below this
- **Do not peek**: Decide sample size upfront; checking daily inflates false positives
- **Use a calculator**: PostHog has built-in significance tracking; for manual, use Evan Miller's calculator or VWO's calculator

---

## Quick Reference: Conversion Checklist

- [ ] Hero has a clear, benefit-driven headline
- [ ] Primary CTA is above the fold with action-oriented copy
- [ ] Social proof appears early (logo bar, user count, testimonials)
- [ ] Trust signals near checkout/signup (badges, guarantee, security)
- [ ] Pricing uses anchoring and highlights recommended plan
- [ ] FAQ handles top 3-5 objections
- [ ] Forms ask minimal fields at signup
- [ ] Inline validation with helpful error messages
- [ ] CTA repeats at bottom of page
- [ ] Mobile has sticky CTA or simplified flow
- [ ] Exit intent offers genuine value (lead magnet)
- [ ] Urgency/scarcity is real and verifiable
- [ ] A/B test is running on the highest-impact element
