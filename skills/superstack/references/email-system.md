# Email System Patterns (2026)

## Table of Contents
- [React Email + Resend Setup](#react-email--resend-setup)
  - [Installation](#installation)
  - [Email Template (`emails/welcome.tsx`)](#email-template-emailswelcometsx)
  - [Sending via Server Action (`app/actions/email.ts`)](#sending-via-server-action-appactionsemailts)
  - [Local Preview](#local-preview)
- [Service Comparison](#service-comparison)
- [Essential Templates (8 Every App Needs)](#essential-templates-8-every-app-needs)
  - [Best Practices](#best-practices)
- [Email Deliverability: SPF, DKIM, DMARC](#email-deliverability-spf-dkim-dmarc)
  - [SPF Record](#spf-record)
  - [DKIM Record](#dkim-record)
  - [DMARC Record](#dmarc-record)
  - [Domain Warming Schedule (New Domains)](#domain-warming-schedule-new-domains)
- [Onboarding Drip Sequence](#onboarding-drip-sequence)
  - [Sequence Overview (5-7 Emails over 14-21 Days)](#sequence-overview-5-7-emails-over-14-21-days)
  - [Key Principle: Behavior-Based, Not Time-Based](#key-principle-behavior-based-not-time-based)
  - [Implementation: Resend + Supabase State Tracking](#implementation-resend--supabase-state-tracking)
  - [Scheduling Options](#scheduling-options)

## React Email + Resend Setup

### Installation

```bash
bun add resend @react-email/components
```

### Email Template (`emails/welcome.tsx`)

```tsx
import {
  Html,
  Head,
  Body,
  Container,
  Section,
  Text,
  Button,
  Heading,
  Preview,
  Img,
} from "@react-email/components";

interface WelcomeEmailProps {
  name: string;
  loginUrl: string;
}

export default function WelcomeEmail({ name, loginUrl }: WelcomeEmailProps) {
  return (
    <Html>
      <Head />
      <Preview>Welcome to YourApp — let's get you started</Preview>
      <Body style={{ backgroundColor: "#f6f9fc", fontFamily: "sans-serif" }}>
        <Container style={{ maxWidth: "560px", margin: "0 auto", padding: "40px 20px" }}>
          <Img
            src="https://yourapp.com/logo.png"
            width={120}
            height={36}
            alt="YourApp"
          />
          <Section style={{ backgroundColor: "#ffffff", borderRadius: "8px", padding: "32px", marginTop: "24px" }}>
            <Heading as="h1" style={{ fontSize: "24px", marginBottom: "16px" }}>
              Welcome, {name}!
            </Heading>
            <Text style={{ fontSize: "16px", lineHeight: "24px", color: "#374151" }}>
              Your account is ready. Here's what to do next:
            </Text>
            <Text style={{ fontSize: "16px", lineHeight: "24px", color: "#374151" }}>
              1. Complete your profile{"\n"}
              2. Create your first project{"\n"}
              3. Invite your team
            </Text>
            <Button
              href={loginUrl}
              style={{
                backgroundColor: "#2563eb",
                color: "#ffffff",
                padding: "12px 24px",
                borderRadius: "6px",
                fontSize: "16px",
                textDecoration: "none",
                display: "inline-block",
                marginTop: "16px",
              }}
            >
              Go to Dashboard
            </Button>
          </Section>
          <Text style={{ fontSize: "12px", color: "#9ca3af", textAlign: "center", marginTop: "24px" }}>
            YourApp Inc. · Frankfurt, Germany
          </Text>
        </Container>
      </Body>
    </Html>
  );
}
```

### Sending via Server Action (`app/actions/email.ts`)

```typescript
"use server";

import { Resend } from "resend";
import WelcomeEmail from "@/emails/welcome";

const resend = new Resend(process.env.RESEND_API_KEY!);

export async function sendWelcomeEmail(email: string, name: string) {
  const { data, error } = await resend.emails.send({
    from: "YourApp <hello@yourapp.com>",
    to: email,
    subject: "Welcome to YourApp!",
    react: WelcomeEmail({ name, loginUrl: `${process.env.NEXT_PUBLIC_APP_URL}/login` }),
  });

  if (error) {
    console.error("Email send failed:", error);
    throw new Error("Failed to send welcome email");
  }

  return data;
}
```

### Local Preview

```bash
npx react-email dev
# Opens http://localhost:3000 with live preview of all templates in /emails
```

---

## Service Comparison

| Feature | Resend | Postmark | SendGrid |
|---|---|---|---|
| Free tier | 3,000 emails/mo | 100 emails/mo | 100 emails/day |
| Pricing | $20/mo for 50k | $15/mo for 10k | $20/mo for 50k |
| React Email | Native | Manual | Manual |
| DX | Excellent | Good | Average |
| Deliverability | Very high | Highest | Good |
| Best for | Next.js startups | Transactional-critical | High volume |
| Webhooks | Yes | Yes | Yes |
| Analytics | Basic | Detailed | Detailed |

**Recommendation:** Resend for Next.js projects. Native React Email support, simple API, great DX. Switch to Postmark if deliverability becomes mission-critical (e.g., financial notifications).

---

## Essential Templates (8 Every App Needs)

| # | Template | Trigger | Key Content |
|---|---|---|---|
| 1 | Welcome / Verification | User signs up | Greeting, verify link, quick start steps |
| 2 | Password Reset | User requests reset | Reset link (expires 1h), security notice |
| 3 | Email Verification | User changes email | Verify new address, confirm link |
| 4 | Invoice / Receipt | Payment succeeds | Amount, date, plan, download link |
| 5 | Payment Failed | Invoice payment fails | Update payment method CTA, grace period info |
| 6 | Trial Ending | 3 days before trial ends | Remaining days, upgrade CTA, feature recap |
| 7 | Feature Notification | New feature ships | What's new, how to use, CTA |
| 8 | Account Deactivation | User deletes account | Confirmation, data retention policy, reactivation window |

### Best Practices

- **Subject lines:** Clear and action-oriented, under 50 characters
- **From address:** Use a recognizable sender (hello@, team@, not noreply@)
- **Preheader text:** Always set — it shows in inbox preview
- **Unsubscribe:** Required for marketing emails (CAN-SPAM, GDPR)
- **Plain text fallback:** Always include for accessibility and deliverability
- **Mobile-first:** 560px max-width container, 16px+ font size

---

## Email Deliverability: SPF, DKIM, DMARC

### SPF Record

```
Type: TXT
Host: @
Value: v=spf1 include:send.resend.com ~all
```

If using multiple senders (e.g., Resend + Google Workspace):

```
v=spf1 include:send.resend.com include:_spf.google.com ~all
```

### DKIM Record

Provided by your email service. Example for Resend:

```
Type: TXT
Host: resend._domainkey
Value: (provided by Resend during domain verification)
```

- Use 2048-bit DKIM keys (not 1024-bit)
- Rotate keys annually

### DMARC Record

```
Type: TXT
Host: _dmarc
```

**Rollout in phases:**

```
# Phase 1: Monitor only (2 weeks)
v=DMARC1; p=none; rua=mailto:dmarc-reports@yourapp.com

# Phase 2: Quarantine (2 weeks)
v=DMARC1; p=quarantine; pct=50; rua=mailto:dmarc-reports@yourapp.com

# Phase 3: Reject (production)
v=DMARC1; p=reject; rua=mailto:dmarc-reports@yourapp.com
```

### Domain Warming Schedule (New Domains)

| Week | Daily Volume | Notes |
|---|---|---|
| 1 | 50-100 | Send to engaged users only |
| 2 | 200-500 | Expand to active users |
| 3 | 500-1,000 | Monitor bounce rates |
| 4 | 1,000-5,000 | Check spam complaint rate (<0.1%) |
| 5+ | Scale up | Increase by 2x per week if metrics are clean |

---

## Onboarding Drip Sequence

### Sequence Overview (5-7 Emails over 14-21 Days)

| # | Email | Timing | Trigger |
|---|---|---|---|
| 1 | Welcome | Immediately | Signup |
| 2 | Quick Win | Day 1 | Signup + hasn't completed onboarding |
| 3 | Setup Nudge | Day 3 | Hasn't created first project |
| 4 | Social Proof | Day 5 | Active but not converted |
| 5 | Aha Moment Push | Day 7 | Hasn't hit key feature |
| 6 | Trial Reminder | Day 11 | Trial ending in 3 days |
| 7 | Last Chance | Day 13 | Trial ending tomorrow |

### Key Principle: Behavior-Based, Not Time-Based

Skip emails if the user has already completed the action. Never send a "create your first project" nudge to someone who already has 5 projects.

### Implementation: Resend + Supabase State Tracking

**State tracking table:**

```sql
CREATE TABLE onboarding_state (
  user_id UUID PRIMARY KEY REFERENCES auth.users(id),
  welcome_sent_at TIMESTAMPTZ,
  quick_win_sent_at TIMESTAMPTZ,
  setup_nudge_sent_at TIMESTAMPTZ,
  social_proof_sent_at TIMESTAMPTZ,
  aha_push_sent_at TIMESTAMPTZ,
  trial_reminder_sent_at TIMESTAMPTZ,
  last_chance_sent_at TIMESTAMPTZ,
  first_project_at TIMESTAMPTZ,
  aha_moment_at TIMESTAMPTZ,
  converted_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

ALTER TABLE onboarding_state ENABLE ROW LEVEL SECURITY;
```

**Drip logic (Server Action or cron job):**

```typescript
import { db } from "@/lib/db";
import { resend } from "@/lib/resend";
import QuickWinEmail from "@/emails/quick-win";

export async function processOnboardingDrip() {
  const now = new Date();

  // Email #2: Quick Win — 24h after signup, hasn't completed onboarding
  const quickWinTargets = await db.query(`
    SELECT u.id, u.email, u.name, o.*
    FROM users u
    JOIN onboarding_state o ON u.id = o.user_id
    WHERE o.welcome_sent_at IS NOT NULL
      AND o.quick_win_sent_at IS NULL
      AND o.first_project_at IS NULL
      AND o.converted_at IS NULL
      AND o.welcome_sent_at < NOW() - INTERVAL '24 hours'
  `);

  for (const user of quickWinTargets) {
    await resend.emails.send({
      from: "YourApp <hello@yourapp.com>",
      to: user.email,
      subject: "Create your first project in 2 minutes",
      react: QuickWinEmail({ name: user.name }),
    });

    await db.query(`
      UPDATE onboarding_state
      SET quick_win_sent_at = NOW()
      WHERE user_id = $1
    `, [user.id]);
  }

  // Email #6: Trial Reminder — 3 days before trial ends
  const trialReminderTargets = await db.query(`
    SELECT u.id, u.email, u.name, u.trial_ends_at, o.*
    FROM users u
    JOIN onboarding_state o ON u.id = o.user_id
    WHERE o.trial_reminder_sent_at IS NULL
      AND o.converted_at IS NULL
      AND u.trial_ends_at BETWEEN NOW() AND NOW() + INTERVAL '3 days'
  `);

  for (const user of trialReminderTargets) {
    await resend.emails.send({
      from: "YourApp <hello@yourapp.com>",
      to: user.email,
      subject: `Your trial ends in 3 days`,
      react: TrialReminderEmail({
        name: user.name,
        daysLeft: 3,
        upgradeUrl: `${process.env.NEXT_PUBLIC_APP_URL}/pricing`,
      }),
    });

    await db.query(`
      UPDATE onboarding_state
      SET trial_reminder_sent_at = NOW()
      WHERE user_id = $1
    `, [user.id]);
  }
}
```

### Scheduling Options

- **n8n (self-hosted):** Cron trigger every hour, HTTP request to your API endpoint
- **Vercel Cron:** Add to `vercel.json`, calls a route handler
- **Supabase pg_cron:** Run drip logic directly in SQL/Edge Functions

```json
// vercel.json
{
  "crons": [
    {
      "path": "/api/cron/onboarding-drip",
      "schedule": "0 * * * *"
    }
  ]
}
```
