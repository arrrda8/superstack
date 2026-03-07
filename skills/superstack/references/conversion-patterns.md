# Conversion Patterns

Reference for conversion rate optimization patterns, UI components, and psychological triggers for web apps and landing pages.

> **Ethics note:** Scarcity and urgency tactics should only reflect real constraints. Fake countdown timers, fabricated stock levels, or manufactured urgency erode trust and can violate consumer protection laws in the EU (UWG) and other jurisdictions. Only use these patterns when the underlying data is genuine.

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
            <svg
              key={i}
              className={`h-4 w-4 ${
                i < testimonial.rating! ? "text-amber-400" : "text-muted"
              }`}
              fill="currentColor"
              viewBox="0 0 20 20"
            >
              <path d="M9.049 2.927c.3-.921 1.603-.921 1.902 0l1.07 3.292a1 1 0 00.95.69h3.462c.969 0 1.371 1.24.588 1.81l-2.8 2.034a1 1 0 00-.364 1.118l1.07 3.292c.3.921-.755 1.688-1.54 1.118l-2.8-2.034a1 1 0 00-1.175 0l-2.8 2.034c-.784.57-1.838-.197-1.539-1.118l1.07-3.292a1 1 0 00-.364-1.118L2.98 8.72c-.783-.57-.38-1.81.588-1.81h3.461a1 1 0 00.951-.69l1.07-3.292z" />
            </svg>
          ))}
        </div>
      )}
      <blockquote className="text-sm leading-relaxed text-foreground">
        &ldquo;{testimonial.quote}&rdquo;
      </blockquote>
      <figcaption className="mt-4 flex items-center gap-3">
        <img
          src={testimonial.avatarUrl}
          alt={testimonial.name}
          className="h-10 w-10 rounded-full object-cover"
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
    <section className="py-12">
      <p className="mb-8 text-center text-sm font-medium text-muted-foreground">
        Trusted by teams at
      </p>
      <div className="flex flex-wrap items-center justify-center gap-x-12 gap-y-6">
        {logos.map((logo) => (
          <img
            key={logo.alt}
            src={logo.src}
            alt={logo.alt}
            className="h-8 opacity-60 grayscale transition-all hover:opacity-100 hover:grayscale-0"
          />
        ))}
      </div>
    </section>
  );
}
```

### Live user count / activity indicator

```tsx
"use client";

import { useEffect, useState } from "react";

function LiveActivity({ baseCount = 127 }: { baseCount?: number }) {
  const [count, setCount] = useState(baseCount);

  useEffect(() => {
    // Simulate small fluctuations — replace with real WebSocket/API data
    const interval = setInterval(() => {
      setCount((prev) => prev + Math.floor(Math.random() * 3) - 1);
    }, 5000);
    return () => clearInterval(interval);
  }, []);

  return (
    <div className="inline-flex items-center gap-2 rounded-full bg-emerald-50 px-3 py-1 text-sm text-emerald-700 dark:bg-emerald-950 dark:text-emerald-300">
      <span className="relative flex h-2 w-2">
        <span className="absolute inline-flex h-full w-full animate-ping rounded-full bg-emerald-400 opacity-75" />
        <span className="relative inline-flex h-2 w-2 rounded-full bg-emerald-500" />
      </span>
      {count} people using this right now
    </div>
  );
}
```

> **Note:** Only show real-time counts if backed by actual data (analytics, WebSocket connections). Fabricating numbers is deceptive.

---

## 2. Trust Signals

### Security badges row

```tsx
function TrustBadges() {
  const badges = [
    { icon: "shield-check", label: "256-bit SSL" },
    { icon: "lock", label: "GDPR compliant" },
    { icon: "credit-card", label: "Secure payments" },
    { icon: "undo", label: "30-day money-back" },
  ];

  return (
    <div className="flex flex-wrap items-center justify-center gap-6 py-6">
      {badges.map((badge) => (
        <div
          key={badge.label}
          className="flex items-center gap-2 text-sm text-muted-foreground"
        >
          <Icon name={badge.icon} className="h-5 w-5 text-emerald-600" />
          <span>{badge.label}</span>
        </div>
      ))}
    </div>
  );
}
```

### Press mentions

```tsx
function PressBar() {
  return (
    <section className="border-y border-border bg-muted/30 py-8">
      <p className="mb-6 text-center text-xs font-semibold uppercase tracking-widest text-muted-foreground">
        As seen in
      </p>
      <div className="flex flex-wrap items-center justify-center gap-10">
        {["TechCrunch", "Forbes", "Wired", "The Verge"].map((name) => (
          <span
            key={name}
            className="text-lg font-serif font-bold text-muted-foreground/50"
          >
            {name}
          </span>
        ))}
      </div>
    </section>
  );
}
```

### Money-back guarantee badge

```tsx
function MoneyBackGuarantee() {
  return (
    <div className="mx-auto max-w-sm rounded-xl border-2 border-dashed border-emerald-300 bg-emerald-50 p-6 text-center dark:border-emerald-800 dark:bg-emerald-950">
      <div className="mx-auto mb-3 flex h-12 w-12 items-center justify-center rounded-full bg-emerald-100 dark:bg-emerald-900">
        <Icon name="shield-check" className="h-6 w-6 text-emerald-600" />
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
  monthlyPrice,
  annualPrice,
}: {
  monthlyPrice: number;
  annualPrice: number;
}) {
  const [isAnnual, setIsAnnual] = useState(true);
  const savings = Math.round(
    ((monthlyPrice * 12 - annualPrice) / (monthlyPrice * 12)) * 100
  );

  return (
    <div>
      <div className="flex items-center justify-center gap-3">
        <span
          className={`text-sm ${!isAnnual ? "font-semibold text-foreground" : "text-muted-foreground"}`}
        >
          Monthly
        </span>
        <button
          role="switch"
          aria-checked={isAnnual}
          onClick={() => setIsAnnual(!isAnnual)}
          className={`relative h-6 w-11 rounded-full transition-colors ${
            isAnnual ? "bg-primary" : "bg-muted"
          }`}
        >
          <span
            className={`absolute top-0.5 left-0.5 h-5 w-5 rounded-full bg-white transition-transform ${
              isAnnual ? "translate-x-5" : "translate-x-0"
            }`}
          />
        </button>
        <span
          className={`text-sm ${isAnnual ? "font-semibold text-foreground" : "text-muted-foreground"}`}
        >
          Annual
          <span className="ml-1 rounded-full bg-emerald-100 px-2 py-0.5 text-xs font-medium text-emerald-700 dark:bg-emerald-900 dark:text-emerald-300">
            Save {savings}%
          </span>
        </span>
      </div>

      <div className="mt-6 text-center">
        <span className="text-4xl font-bold text-foreground">
          ${isAnnual ? Math.round(annualPrice / 12) : monthlyPrice}
        </span>
        <span className="text-muted-foreground">/mo</span>
        {isAnnual && (
          <p className="mt-1 text-sm text-muted-foreground">
            Billed annually at ${annualPrice}/year
          </p>
        )}
      </div>
    </div>
  );
}
```

### "Most popular" badge

```tsx
function PricingCard({
  isPopular,
  name,
  price,
  features,
}: {
  isPopular?: boolean;
  name: string;
  price: string;
  features: string[];
}) {
  return (
    <div
      className={`relative rounded-2xl border p-8 ${
        isPopular
          ? "border-primary shadow-lg shadow-primary/10 ring-1 ring-primary"
          : "border-border"
      }`}
    >
      {isPopular && (
        <div className="absolute -top-3 left-1/2 -translate-x-1/2 rounded-full bg-primary px-4 py-1 text-xs font-semibold text-primary-foreground">
          Most popular
        </div>
      )}
      <h3 className="text-lg font-semibold">{name}</h3>
      <p className="mt-2 text-3xl font-bold">{price}</p>
      <ul className="mt-6 space-y-3">
        {features.map((f) => (
          <li key={f} className="flex items-center gap-2 text-sm">
            <Icon name="check" className="h-4 w-4 text-emerald-500" />
            {f}
          </li>
        ))}
      </ul>
      <button
        className={`mt-8 w-full rounded-lg py-2.5 text-sm font-medium ${
          isPopular
            ? "bg-primary text-primary-foreground hover:bg-primary/90"
            : "border border-border bg-background hover:bg-muted"
        }`}
      >
        Get started
      </button>
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
      <div className="mt-8 flex gap-3">
        <button className="rounded-lg bg-primary px-6 py-3 font-medium text-primary-foreground">
          Start building free
        </button>
        <button className="rounded-lg border border-border px-6 py-3 font-medium">
          See a demo
        </button>
      </div>
    </section>
  );
}

function StickyBottomCTA() {
  return (
    // Appears when scrolled past hero CTA
    <div className="fixed bottom-0 inset-x-0 z-50 border-t border-border bg-background/80 backdrop-blur-md p-3 md:hidden">
      <button className="w-full rounded-lg bg-primary py-3 font-medium text-primary-foreground">
        Start building free
      </button>
    </div>
  );
}
```

### Urgency without being sleazy

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

---

## 6. Exit Intent

### Mouse leave detection

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
function ExitIntentPopup() {
  const { showPopup, dismiss } = useExitIntent();

  if (!showPopup) return null;

  return (
    <div className="fixed inset-0 z-50 flex items-center justify-center bg-black/50 backdrop-blur-sm">
      <div className="relative mx-4 max-w-md rounded-2xl bg-background p-8 shadow-2xl">
        <button
          onClick={dismiss}
          className="absolute top-4 right-4 text-muted-foreground hover:text-foreground"
          aria-label="Close"
        >
          <Icon name="x" className="h-5 w-5" />
        </button>

        <h3 className="text-xl font-bold">Wait — before you go</h3>
        <p className="mt-2 text-sm text-muted-foreground">
          Get 20% off your first month. This offer is only available right now.
        </p>

        <form
          onSubmit={(e) => {
            e.preventDefault();
            // Handle email capture
            dismiss();
          }}
          className="mt-6"
        >
          <input
            type="email"
            placeholder="your@email.com"
            required
            className="w-full rounded-lg border border-border bg-background px-4 py-2.5 text-sm"
          />
          <button
            type="submit"
            className="mt-3 w-full rounded-lg bg-primary py-2.5 text-sm font-medium text-primary-foreground"
          >
            Claim my 20% discount
          </button>
        </form>

        <button
          onClick={dismiss}
          className="mt-3 w-full text-center text-xs text-muted-foreground hover:underline"
        >
          No thanks, I will pay full price
        </button>
      </div>
    </div>
  );
}
```

---

## 7. Scarcity & Urgency

> **Ethical requirement:** Only use scarcity/urgency when reflecting real constraints. Fake timers and invented stock counts violate trust and potentially EU consumer law.

### Real countdown timer

```tsx
"use client";

import { useEffect, useState } from "react";

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
```

### Stock / capacity indicator (real data only)

```tsx
function SpotsRemaining({ total, remaining }: { total: number; remaining: number }) {
  const percentage = (remaining / total) * 100;
  const isLow = percentage <= 20;

  return (
    <div className="space-y-2">
      <div className="flex items-center justify-between text-sm">
        <span className={isLow ? "font-semibold text-amber-600" : "text-muted-foreground"}>
          {remaining} of {total} spots remaining
        </span>
      </div>
      <div className="h-2 w-full overflow-hidden rounded-full bg-muted">
        <div
          className={`h-full rounded-full transition-all ${
            isLow ? "bg-amber-500" : "bg-primary"
          }`}
          style={{ width: `${100 - percentage}%` }}
        />
      </div>
    </div>
  );
}
```

---

## 8. Form Optimization

### Reduce fields — minimal form

```tsx
// Bad: 8 fields
// Good: 3 fields (name, email, company) — ask the rest later

function MinimalSignupForm() {
  return (
    <form className="space-y-4">
      <div>
        <label htmlFor="email" className="text-sm font-medium">
          Work email
        </label>
        <input
          id="email"
          type="email"
          required
          placeholder="you@company.com"
          className="mt-1 w-full rounded-lg border border-border px-4 py-2.5"
        />
      </div>
      <button
        type="submit"
        className="w-full rounded-lg bg-primary py-2.5 font-medium text-primary-foreground"
      >
        Start free trial
      </button>
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
      <label className="text-sm font-medium">{label}</label>
      <input
        type={type}
        value={value}
        onChange={handleChange}
        onBlur={handleBlur}
        className={`mt-1 w-full rounded-lg border px-4 py-2.5 ${
          error && touched
            ? "border-red-500 focus:ring-red-500"
            : "border-border focus:ring-primary"
        }`}
        {...props}
      />
      {error && touched && (
        <p className="mt-1 text-xs text-red-500">{error}</p>
      )}
    </div>
  );
}
```

### Multi-step form with progress

```tsx
"use client";

import { useState } from "react";

function MultiStepForm({ steps }: { steps: { title: string; content: React.ReactNode }[] }) {
  const [currentStep, setCurrentStep] = useState(0);

  return (
    <div>
      {/* Progress bar */}
      <div className="mb-8">
        <div className="flex items-center justify-between">
          {steps.map((step, i) => (
            <div key={i} className="flex items-center">
              <div
                className={`flex h-8 w-8 items-center justify-center rounded-full text-sm font-medium ${
                  i <= currentStep
                    ? "bg-primary text-primary-foreground"
                    : "bg-muted text-muted-foreground"
                }`}
              >
                {i < currentStep ? (
                  <Icon name="check" className="h-4 w-4" />
                ) : (
                  i + 1
                )}
              </div>
              {i < steps.length - 1 && (
                <div
                  className={`mx-2 h-0.5 w-12 sm:w-24 ${
                    i < currentStep ? "bg-primary" : "bg-muted"
                  }`}
                />
              )}
            </div>
          ))}
        </div>
        <p className="mt-2 text-sm font-medium">
          Step {currentStep + 1}: {steps[currentStep].title}
        </p>
      </div>

      {/* Step content */}
      <div>{steps[currentStep].content}</div>

      {/* Navigation */}
      <div className="mt-6 flex justify-between">
        <button
          onClick={() => setCurrentStep((s) => Math.max(0, s - 1))}
          disabled={currentStep === 0}
          className="rounded-lg border border-border px-4 py-2 text-sm disabled:opacity-50"
        >
          Back
        </button>
        <button
          onClick={() =>
            setCurrentStep((s) => Math.min(steps.length - 1, s + 1))
          }
          className="rounded-lg bg-primary px-6 py-2 text-sm font-medium text-primary-foreground"
        >
          {currentStep === steps.length - 1 ? "Submit" : "Continue"}
        </button>
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
      <button className="flex w-full items-center justify-center gap-2 rounded-lg border border-border py-2.5 text-sm font-medium hover:bg-muted">
        <GoogleIcon className="h-5 w-5" />
        Continue with Google
      </button>
      <button className="flex w-full items-center justify-center gap-2 rounded-lg border border-border py-2.5 text-sm font-medium hover:bg-muted">
        <GitHubIcon className="h-5 w-5" />
        Continue with GitHub
      </button>
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

### Button labels

| Context | Bad | Good |
|---|---|---|
| Signup | Submit | Create my account |
| Delete | Delete | Remove permanently |
| Cancel subscription | Cancel | I want to cancel |
| Save changes | Save | Save changes |
| Loading state | Loading... | Saving your changes... |
| Success | Done | Changes saved |

### Form placeholders

```tsx
// Bad: placeholder repeats the label
<label>Email</label>
<input placeholder="Email" />

// Good: placeholder provides an example
<label>Work email</label>
<input placeholder="you@company.com" />

// Good: placeholder sets expectations
<label>Company website</label>
<input placeholder="https://example.com" />
```

### Error messages

```tsx
// Bad
"Invalid input"
"Error"
"Field required"

// Good — explain what went wrong and how to fix it
"Please enter a valid email address (e.g., name@company.com)"
"Password must be at least 8 characters with one number"
"We need your company name to set up your workspace"
```

### Empty states

```tsx
function EmptyState({
  icon,
  title,
  description,
  action,
}: {
  icon: string;
  title: string;
  description: string;
  action: { label: string; onClick: () => void };
}) {
  return (
    <div className="flex flex-col items-center justify-center py-16 text-center">
      <div className="mb-4 rounded-full bg-muted p-4">
        <Icon name={icon} className="h-8 w-8 text-muted-foreground" />
      </div>
      <h3 className="text-lg font-semibold">{title}</h3>
      <p className="mt-1 max-w-sm text-sm text-muted-foreground">
        {description}
      </p>
      <button
        onClick={action.onClick}
        className="mt-4 rounded-lg bg-primary px-4 py-2 text-sm font-medium text-primary-foreground"
      >
        {action.label}
      </button>
    </div>
  );
}

// Usage
<EmptyState
  icon="inbox"
  title="No messages yet"
  description="When customers reach out, their messages will appear here."
  action={{ label: "Set up live chat", onClick: openSetup }}
/>
```

---

## 10. A/B Testing

### Vercel Toolbar / Edge Config flags

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

function PricingSection() {
  const variant = useFeatureFlagVariantKey("pricing-test");
  // variant: "control" | "three-tiers" | "single-tier" | undefined

  if (variant === "three-tiers") return <ThreeTierPricing />;
  if (variant === "single-tier") return <SingleTierPricing />;
  return <DefaultPricing />;
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
- **Use a calculator**: PostHog has built-in significance tracking; for manual, use Evan Miller's calculator
