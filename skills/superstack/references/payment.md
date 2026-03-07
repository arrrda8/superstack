# Payment & Billing Patterns (2026)

## Table of Contents
- [Stripe + Next.js App Router](#stripe--nextjs-app-router)
  - [Key Principle: Server Actions, NOT API Routes](#key-principle-server-actions-not-api-routes)
  - [Installation](#installation)
  - [Stripe Server Client (`lib/stripe.ts`)](#stripe-server-client-libstripets)
  - [Embedded Checkout (Recommended Pattern)](#embedded-checkout-recommended-pattern)
- [Webhook Handler](#webhook-handler)
- [Stripe Billing Portal](#stripe-billing-portal)
- [EU VAT Handling](#eu-vat-handling)
  - [Option A: Stripe Tax (Recommended)](#option-a-stripe-tax-recommended)
  - [VAT ID Validation (B2B Reverse Charge)](#vat-id-validation-b2b-reverse-charge)
  - [VAT Rules Summary](#vat-rules-summary)
- [Merchant of Record Alternatives](#merchant-of-record-alternatives)
  - [When to Use MoR](#when-to-use-mor)
  - [When to Stick with Stripe](#when-to-stick-with-stripe)
- [Pricing Page Patterns](#pricing-page-patterns)
  - [Structure](#structure)
  - [Conversion Best Practices](#conversion-best-practices)

## Stripe + Next.js App Router

### Key Principle: Server Actions, NOT API Routes

In 2026 Next.js, use Server Actions for payment flows. API Routes are only needed for webhooks.

### Installation

```bash
bun add stripe @stripe/stripe-js @stripe/react-stripe-js
```

### Stripe Server Client (`lib/stripe.ts`)

```typescript
import Stripe from "stripe";

export const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, {
  apiVersion: "2025-12-18.acacia",
  typescript: true,
});
```

### Embedded Checkout (Recommended Pattern)

Keeps the user on your domain — no redirect to Stripe-hosted page.

**Server Action (`app/actions/checkout.ts`)**

```typescript
"use server";

import { stripe } from "@/lib/stripe";
import { auth } from "@/lib/auth";

export async function createCheckoutSession(priceId: string) {
  const session = await auth.api.getSession({ headers: await headers() });
  if (!session) throw new Error("Not authenticated");

  const checkoutSession = await stripe.checkout.sessions.create({
    ui_mode: "embedded",
    mode: "subscription",
    customer_email: session.user.email,
    line_items: [{ price: priceId, quantity: 1 }],
    automatic_tax: { enabled: true },
    tax_id_collection: { enabled: true },
    return_url: `${process.env.NEXT_PUBLIC_APP_URL}/checkout/success?session_id={CHECKOUT_SESSION_ID}`,
    metadata: {
      userId: session.user.id,
    },
  });

  return { clientSecret: checkoutSession.client_secret };
}
```

**Client Component (`app/checkout/page.tsx`)**

```typescript
"use client";

import { useCallback } from "react";
import { loadStripe } from "@stripe/stripe-js";
import {
  EmbeddedCheckoutProvider,
  EmbeddedCheckout,
} from "@stripe/react-stripe-js";
import { createCheckoutSession } from "@/app/actions/checkout";

const stripePromise = loadStripe(process.env.NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY!);

export default function CheckoutPage({ params }: { params: { priceId: string } }) {
  const fetchClientSecret = useCallback(async () => {
    const { clientSecret } = await createCheckoutSession(params.priceId);
    return clientSecret!;
  }, [params.priceId]);

  return (
    <div className="max-w-lg mx-auto py-12">
      <EmbeddedCheckoutProvider stripe={stripePromise} options={{ fetchClientSecret }}>
        <EmbeddedCheckout />
      </EmbeddedCheckoutProvider>
    </div>
  );
}
```

---

## Webhook Handler

Webhooks still require an API route (Stripe sends POST requests directly).

**`app/api/webhooks/stripe/route.ts`**

```typescript
import { headers } from "next/headers";
import { stripe } from "@/lib/stripe";
import { db } from "@/lib/db";

export async function POST(req: Request) {
  const body = await req.text();
  const headersList = await headers();
  const signature = headersList.get("stripe-signature")!;

  let event;

  try {
    event = stripe.webhooks.constructEvent(
      body,
      signature,
      process.env.STRIPE_WEBHOOK_SECRET!
    );
  } catch (err) {
    console.error("Webhook signature verification failed:", err);
    return new Response("Invalid signature", { status: 400 });
  }

  switch (event.type) {
    case "checkout.session.completed": {
      const session = event.data.object;
      await db.update(users).set({
        stripeCustomerId: session.customer as string,
        subscriptionId: session.subscription as string,
        subscriptionStatus: "active",
        plan: session.metadata?.plan ?? "pro",
      }).where(eq(users.id, session.metadata!.userId));
      break;
    }

    case "customer.subscription.updated": {
      const subscription = event.data.object;
      await db.update(users).set({
        subscriptionStatus: subscription.status,
        currentPeriodEnd: new Date(subscription.current_period_end * 1000),
      }).where(eq(users.stripeCustomerId, subscription.customer as string));
      break;
    }

    case "customer.subscription.deleted": {
      const subscription = event.data.object;
      await db.update(users).set({
        subscriptionStatus: "canceled",
        plan: "free",
      }).where(eq(users.stripeCustomerId, subscription.customer as string));
      break;
    }

    case "invoice.payment_failed": {
      const invoice = event.data.object;
      // Send dunning email, update status
      await db.update(users).set({
        subscriptionStatus: "past_due",
      }).where(eq(users.stripeCustomerId, invoice.customer as string));
      // TODO: trigger email via Resend
      break;
    }
  }

  return new Response("OK", { status: 200 });
}
```

---

## Stripe Billing Portal

Let Stripe handle subscription management UI. No need to build custom pages for upgrades, downgrades, payment method changes, or cancellation.

```typescript
"use server";

import { stripe } from "@/lib/stripe";
import { redirect } from "next/navigation";

export async function createPortalSession() {
  const session = await auth.api.getSession({ headers: await headers() });
  if (!session?.user.stripeCustomerId) throw new Error("No billing account");

  const portalSession = await stripe.billingPortal.sessions.create({
    customer: session.user.stripeCustomerId,
    return_url: `${process.env.NEXT_PUBLIC_APP_URL}/settings/billing`,
  });

  redirect(portalSession.url);
}
```

**What the portal handles for you:**
- Upgrade / downgrade between plans
- Update payment method (card, SEPA, etc.)
- View and download invoices
- Cancel or pause subscription
- Update billing address

Configure allowed actions in Stripe Dashboard > Settings > Billing > Customer Portal.

---

## EU VAT Handling

### Option A: Stripe Tax (Recommended)

Enabled via `automatic_tax: { enabled: true }` in checkout session (see above). Stripe calculates the correct VAT rate based on customer location.

- Requires Stripe Tax to be enabled in Dashboard
- Handles EU, UK, and other jurisdictions automatically
- Files tax reports via Stripe (additional fee: 0.5% per transaction)

### VAT ID Validation (B2B Reverse Charge)

```typescript
import { VIES } from "vies-ts";

async function validateVATID(vatId: string): Promise<boolean> {
  const vies = new VIES();
  try {
    const result = await vies.checkVAT({
      countryCode: vatId.slice(0, 2),
      vatNumber: vatId.slice(2),
    });
    return result.valid;
  } catch {
    return false; // VIES service unavailable — retry later
  }
}
```

### VAT Rules Summary

| Scenario | VAT Treatment |
|---|---|
| B2C within same EU country | Charge local VAT rate |
| B2C cross-border EU | Charge destination country VAT (OSS) |
| B2B cross-border EU (valid VAT ID) | Reverse charge (0% VAT) |
| Non-EU customer | No VAT |

---

## Merchant of Record Alternatives

When you want someone else to handle ALL tax compliance, invoicing, and refunds.

| Feature | LemonSqueezy | Paddle |
|---|---|---|
| Fee | ~5% + $0.50 | ~5% + $0.50 |
| EU VAT handling | Full MoR | Full MoR |
| Payout | Monthly | Monthly |
| Next.js SDK | Good | Good |
| Best for | Indie/small SaaS | Larger SaaS |
| Checkout UX | Overlay / embed | Overlay / embed |

### When to Use MoR

- Solo dev / small team with EU customers
- No finance team to manage VAT filings
- Want zero tax compliance burden
- Willing to accept higher fees (~5% vs Stripe ~2.9%)

### When to Stick with Stripe

- Need maximum control over billing UX
- Have a finance team or accountant
- Revenue > $50k/mo (savings from lower fees add up)
- Need complex billing (metered, usage-based)

---

## Pricing Page Patterns

### Structure

```typescript
const PLANS = [
  {
    name: "Starter",
    monthlyPrice: 29,
    annualPrice: 290, // ~17% discount
    priceId: { monthly: "price_xxx", annual: "price_yyy" },
    features: ["5 projects", "10k events/mo", "Email support"],
    cta: "Start free trial",
    highlighted: false,
  },
  {
    name: "Pro",
    monthlyPrice: 79,
    annualPrice: 790,
    priceId: { monthly: "price_xxx", annual: "price_yyy" },
    features: ["Unlimited projects", "100k events/mo", "Priority support", "Custom domain"],
    cta: "Start free trial",
    highlighted: true, // <-- visual emphasis
    badge: "Most Popular",
  },
  {
    name: "Enterprise",
    monthlyPrice: null,
    annualPrice: null,
    features: ["Unlimited everything", "SLA", "Dedicated support", "SSO/SAML", "Custom integrations"],
    cta: "Talk to sales",
    highlighted: false,
  },
] as const;
```

### Conversion Best Practices

- **Annual toggle** with visible savings ("Save 17%")
- **Highlight the middle plan** (border, background, scale)
- **Feature comparison table** below cards for detail-oriented buyers
- **ROI framing**: "Less than the cost of one client lunch per month"
- **Money-back guarantee**: "30-day money-back, no questions asked"
- **CTA per plan**: Free tier = "Get started", Paid = "Start free trial", Enterprise = "Talk to sales"
- **Social proof near pricing**: "Trusted by 500+ teams" or customer logos
