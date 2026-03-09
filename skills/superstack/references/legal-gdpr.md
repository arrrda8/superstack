# Legal & GDPR Reference — Superstack

## Table of Contents
- [1. Impressum (Germany — DDG since May 2024)](#1-impressum-germany--ddg-since-may-2024)
  - [Required Fields](#required-fields)
  - [Placement Rules](#placement-rules)
  - [Implementation](#implementation)
  - [Penalty](#penalty)
- [2. European Accessibility Act (EAA) — Since June 2025](#2-european-accessibility-act-eaa--since-june-2025)
  - [Who Is Affected](#who-is-affected)
  - [Technical Requirements](#technical-requirements)
  - [Implementation Checklist](#implementation-checklist)
  - [Key WCAG 2.1 AA Requirements](#key-wcag-21-aa-requirements)
  - [Penalty](#penalty-1)
  - [Recommendation](#recommendation)
- [3. GDPR Data Deletion (Art. 17 — Right to Erasure)](#3-gdpr-data-deletion-art-17--right-to-erasure)
  - [Legal Requirements](#legal-requirements)
  - [Supabase Implementation](#supabase-implementation)
  - [API Route](#api-route)
- [4. Cookie Consent Implementation](#4-cookie-consent-implementation)
  - [Recommended Tools (DACH Market)](#recommended-tools-dach-market)
  - [Google Consent Mode v2](#google-consent-mode-v2)
  - [Updating Consent After User Choice](#updating-consent-after-user-choice)
  - [TCF v2.3](#tcf-v23)
- [5. Privacy Policy (Datenschutzerklärung)](#5-privacy-policy-datenschutzerklaerung)
  - [Recommended Generator](#recommended-generator)
  - [Must Cover](#must-cover)
  - [Placement](#placement)
- [6. EU Data Act (Since September 2025)](#6-eu-data-act-since-september-2025)
  - [Key Requirements for SaaS Providers](#key-requirements-for-saas-providers)
  - [Implementation](#implementation-1)
- [7. Terms of Service (AGB)](#7-terms-of-service-agb)
  - [Required Documents for EU SaaS](#required-documents-for-eu-saas)
  - [AI Disclaimer](#ai-disclaimer)
- [8. Data Retention Policy](#8-data-retention-policy)
  - [Retention Periods (Germany)](#retention-periods-germany)
  - [Automatic Cleanup Implementation](#automatic-cleanup-implementation)
  - [Cron Trigger](#cron-trigger)

## 1. Impressum (Germany — DDG since May 2024)

Since May 2024, the legal basis is **DDG (Digital-Dienste-Gesetz)**, NOT TMG.
TMG references are outdated.

### Required Fields

- Full legal name (person or company)
- Physical address (NO PO box allowed)
- Email address
- Phone number (strongly recommended, arguably required)
- Handelsregister entry (court + registration number)
- USt-IdNr. (VAT ID) if applicable
- Vertretungsberechtigte (authorized representatives for legal entities)
- Responsible person for editorial content (if applicable): "Verantwortlich i.S.d. DDG"

### Placement Rules

- Must be reachable within **2 clicks** from every page
- Label: "Impressum" (preferred) or "Kontakt"
- Must be on a **separate, dedicated page** — not buried inside Datenschutzerklärung
- Must be machine-readable (no image-only Impressum)

### Implementation

```typescript
// app/impressum/page.tsx
export const metadata = { title: "Impressum", robots: "noindex" };

export default function ImpressumPage() {
  return (
    <main className="prose mx-auto max-w-2xl px-4 py-16">
      <h1>Impressum</h1>
      <p>Angaben gemaess DDG (Digital-Dienste-Gesetz)</p>

      <h2>Anbieter</h2>
      <p>
        Firmenname GmbH<br />
        Musterstrasse 1<br />
        60311 Frankfurt am Main
      </p>

      <h2>Kontakt</h2>
      <p>
        E-Mail: info@example.com<br />
        Telefon: +49 69 12345678
      </p>

      <h2>Vertretungsberechtigt</h2>
      <p>Geschäftsführer: Max Mustermann</p>

      <h2>Registereintrag</h2>
      <p>
        Handelsregister: Amtsgericht Frankfurt am Main<br />
        Registernummer: HRB 123456
      </p>

      <h2>Umsatzsteuer-ID</h2>
      <p>USt-IdNr. gemaess Paragraph 27a UStG: DE123456789</p>
    </main>
  );
}
```

### Penalty

Up to **50,000 EUR** fine for missing or incomplete Impressum.

---

## 2. European Accessibility Act (EAA) — Since June 2025

The EAA (Barrierefreiheitsstaerkungsgesetz / BFSG in Germany) has been in effect
since **June 28, 2025**.

### Who Is Affected

- E-commerce (online shops)
- Banking and financial services
- Communication services
- Transport services
- E-books and e-readers

### Technical Requirements

- **WCAG 2.1 Level AA** minimum compliance
- Public **Accessibility Statement** required on your website
- Perceivable, Operable, Understandable, Robust (POUR principles)

### Implementation Checklist

```bash
# Install automated testing
bun add -D @axe-core/playwright axe-core

# Or use the CLI for quick audits
bunx @axe-core/cli https://your-site.com
```

```typescript
// e2e/accessibility.spec.ts (Playwright + axe-core)
import { test, expect } from "@playwright/test";
import AxeBuilder from "@axe-core/playwright";

test("homepage has no accessibility violations", async ({ page }) => {
  await page.goto("/");
  const results = await new AxeBuilder({ page })
    .withTags(["wcag2a", "wcag2aa", "wcag21aa"])
    .analyze();
  expect(results.violations).toEqual([]);
});
```

### Key WCAG 2.1 AA Requirements

- Color contrast ratio: minimum 4.5:1 (normal text), 3:1 (large text)
- All images have alt text
- All interactive elements keyboard-accessible
- Focus indicators visible
- Form inputs have associated labels
- Error messages are descriptive
- Page language declared in `<html lang="de">`
- Skip navigation link
- Responsive down to 320px without horizontal scroll

### Penalty

Up to **100,000 EUR** or **4% of annual revenue** (whichever is higher).

### Recommendation

Combine automated tests (axe-core in CI/CD) with manual screen reader testing
(VoiceOver on macOS, NVDA on Windows). Automated tools catch ~30-40% of issues.

---

## 3. GDPR Data Deletion (Art. 17 — Right to Erasure)

### Legal Requirements

- Response deadline: **1 month** (extendable by 2 months for complex requests)
- Delete from **ALL systems** including backups (or mark for deletion on next backup rotation)
- Notify all third parties who received the data
- Document the deletion (anonymized audit log — no PII in the log itself)

### Supabase Implementation

```typescript
// lib/gdpr/delete-user.ts
import { createClient } from "@supabase/supabase-js";

const supabaseAdmin = createClient(
  process.env.SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!
);

export async function deleteUserData(userId: string): Promise<{
  success: boolean;
  deletedFrom: string[];
  errors: string[];
}> {
  const deletedFrom: string[] = [];
  const errors: string[] = [];

  // Define all user-related tables (order matters for foreign keys)
  const userTables = [
    "notifications",
    "activity_logs",
    "subscriptions",
    "payments",
    "orders",
    "profiles",
  ];

  // 1. Delete from all data tables
  for (const table of userTables) {
    try {
      const { error } = await supabaseAdmin
        .from(table)
        .delete()
        .eq("user_id", userId);
      if (error) throw error;
      deletedFrom.push(table);
    } catch (err) {
      errors.push(`${table}: ${(err as Error).message}`);
    }
  }

  // 2. Delete storage files (avatars, uploads)
  try {
    const { data: files } = await supabaseAdmin.storage
      .from("user-uploads")
      .list(userId);

    if (files && files.length > 0) {
      const paths = files.map((f) => `${userId}/${f.name}`);
      await supabaseAdmin.storage.from("user-uploads").remove(paths);
      deletedFrom.push("storage:user-uploads");
    }
  } catch (err) {
    errors.push(`storage: ${(err as Error).message}`);
  }

  // 3. Delete auth user (this is the final step)
  try {
    const { error } = await supabaseAdmin.auth.admin.deleteUser(userId);
    if (error) throw error;
    deletedFrom.push("auth.users");
  } catch (err) {
    errors.push(`auth: ${(err as Error).message}`);
  }

  // 4. Anonymized audit log (NO PII)
  await supabaseAdmin.from("gdpr_audit_log").insert({
    action: "user_deletion",
    anonymized_id: `deleted_${Date.now()}`,
    tables_affected: deletedFrom,
    errors: errors.length > 0 ? errors : null,
    completed_at: new Date().toISOString(),
  });

  return {
    success: errors.length === 0,
    deletedFrom,
    errors,
  };
}
```

### API Route

```typescript
// app/api/gdpr/delete-account/route.ts
import { deleteUserData } from "@/lib/gdpr/delete-user";
import { getServerSession } from "@/lib/auth";

export async function DELETE() {
  const session = await getServerSession();
  if (!session?.user?.id) {
    return Response.json({ error: "Unauthorized" }, { status: 401 });
  }

  const result = await deleteUserData(session.user.id);

  if (!result.success) {
    // Partial deletion — alert admin for manual cleanup
    return Response.json(
      { error: "Partial deletion", details: result.errors },
      { status: 500 }
    );
  }

  return Response.json({ message: "Account deleted" });
}
```

---

## 4. Cookie Consent Implementation

### Recommended Tools (DACH Market)

- **Cookiebot** or **Usercentrics**: Established, auto-scan cookies, TCF 2.3 support
- **Klaro** (open source): Lightweight, self-hosted, full control

### Google Consent Mode v2

Required since March 2024 for Google Ads / GA4.

```typescript
// app/layout.tsx — MUST load BEFORE any tracking scripts
<Script id="consent-defaults" strategy="beforeInteractive">
  {`
    window.dataLayer = window.dataLayer || [];
    function gtag(){dataLayer.push(arguments);}

    gtag('consent', 'default', {
      'ad_storage': 'denied',
      'ad_user_data': 'denied',
      'ad_personalization': 'denied',
      'analytics_storage': 'denied',
      'functionality_storage': 'denied',
      'personalization_storage': 'denied',
      'security_storage': 'granted',
      'wait_for_update': 500,
    });

    gtag('set', 'ads_data_redaction', true);
    gtag('set', 'url_passthrough', true);
  `}
</Script>
```

### Updating Consent After User Choice

```typescript
// Called by your consent banner when user accepts
function updateConsent(preferences: {
  analytics: boolean;
  marketing: boolean;
  functional: boolean;
}) {
  gtag("consent", "update", {
    analytics_storage: preferences.analytics ? "granted" : "denied",
    ad_storage: preferences.marketing ? "granted" : "denied",
    ad_user_data: preferences.marketing ? "granted" : "denied",
    ad_personalization: preferences.marketing ? "granted" : "denied",
    functionality_storage: preferences.functional ? "granted" : "denied",
    personalization_storage: preferences.functional ? "granted" : "denied",
  });
}
```

### TCF v2.3

IAB Transparency and Consent Framework v2.3 migration deadline: **February 2026**.
If using a CMP (Consent Management Platform), verify it supports TCF 2.3.

---

## 5. Privacy Policy (Datenschutzerklärung)

### Recommended Generator

**Datenschutz-Generator.de** by Dr. Thomas Schwenke — the gold standard for German-language privacy policies. Free for personal use, paid for commercial.

### Must Cover

- Identity of the data controller
- All data processing activities (forms, analytics, cookies)
- Third-party services (Vercel, Supabase, Sentry, Stripe, etc.)
- Legal basis for each processing activity (Art. 6 GDPR)
- Data retention periods
- Rights of data subjects (access, rectification, erasure, portability)
- Right to lodge a complaint with a supervisory authority
- Cookie usage and consent mechanism
- Social media integrations
- Newsletter / email marketing (double opt-in required in Germany)

### Placement

- Separate page: `/datenschutz` or `/privacy`
- Linked in footer on every page
- Update whenever new tools or services are integrated

---

## 6. EU Data Act (Since September 2025)

The EU Data Act has been in effect since **September 12, 2025**.

### Key Requirements for SaaS Providers

- **Data portability**: Users must be able to export their data in a machine-readable format
- **Provider switching**: Enable switching to another provider with max **2-month cancellation period**
- **Secure data transfer**: Data must be transferred securely during a switch
- **No vendor lock-in**: Contractual, technical, or commercial obstacles to switching are prohibited
- **Extraterritorial effect**: Applies to any service offered to EU users

### Implementation

```typescript
// app/api/gdpr/export/route.ts
import { getServerSession } from "@/lib/auth";

export async function GET() {
  const session = await getServerSession();
  if (!session?.user?.id) {
    return Response.json({ error: "Unauthorized" }, { status: 401 });
  }

  // Collect all user data from all tables
  const userData = {
    profile: await getProfile(session.user.id),
    orders: await getOrders(session.user.id),
    subscriptions: await getSubscriptions(session.user.id),
    activityLog: await getActivityLog(session.user.id),
    exportedAt: new Date().toISOString(),
    format: "JSON",
    version: "1.0",
  };

  return new Response(JSON.stringify(userData, null, 2), {
    headers: {
      "Content-Type": "application/json",
      "Content-Disposition": `attachment; filename="data-export-${Date.now()}.json"`,
    },
  });
}
```

---

## 7. Terms of Service (AGB)

### Required Documents for EU SaaS

| Document                          | Required | Notes                                  |
| --------------------------------- | -------- | -------------------------------------- |
| AGB (Terms of Service)            | Yes      | General terms and conditions            |
| Datenschutzerklärung (Privacy)   | Yes      | GDPR Art. 13/14                        |
| Impressum                         | Yes      | DDG requirement                        |
| AVV / DPA (Data Processing Agr.)  | B2B Yes  | Required when processing client data   |
| Widerrufsbelehrung (Cancellation) | B2C Yes  | 14-day right of withdrawal             |

### AI Disclaimer

If your product uses AI (LLMs, generated content), include:

- Disclosure that AI is used
- Limitations of AI output (not guaranteed accurate)
- Human oversight statement
- Data processing disclosure (what data is sent to AI providers)

**ALWAYS recommend lawyer review for AGB.** Template generators provide a starting point, not legal advice.

---

## 8. Data Retention Policy

### Retention Periods (Germany)

| Data Type                | Retention     | Legal Basis                    |
| ------------------------ | ------------- | ------------------------------ |
| Tax/accounting records   | 10 years      | AO Paragraph 147, HGB Par. 257 |
| Transaction logs         | 6 years       | HGB Paragraph 257              |
| Invoices                 | 10 years      | AO Paragraph 147               |
| Employment records       | 6 years       | After employment ends          |
| Consent records          | Duration + 3y | Proof of consent               |
| User accounts (inactive) | Define policy | e.g., 24 months inactivity     |
| Analytics data           | 14 months max | GDPR recommendation            |
| Server logs              | 7 days        | GDPR minimization              |

### Automatic Cleanup Implementation

```typescript
// lib/gdpr/data-retention.ts
import { createClient } from "@supabase/supabase-js";

const supabaseAdmin = createClient(
  process.env.SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!
);

interface RetentionRule {
  table: string;
  dateColumn: string;
  retentionDays: number;
  description: string;
}

const retentionRules: RetentionRule[] = [
  {
    table: "server_logs",
    dateColumn: "created_at",
    retentionDays: 7,
    description: "Server access logs",
  },
  {
    table: "analytics_events",
    dateColumn: "created_at",
    retentionDays: 425, // ~14 months
    description: "Analytics events",
  },
  {
    table: "notifications",
    dateColumn: "created_at",
    retentionDays: 90,
    description: "User notifications",
  },
];

export async function runRetentionCleanup() {
  const results: { table: string; deleted: number }[] = [];

  for (const rule of retentionRules) {
    const cutoff = new Date();
    cutoff.setDate(cutoff.getDate() - rule.retentionDays);

    const { count, error } = await supabaseAdmin
      .from(rule.table)
      .delete({ count: "exact" })
      .lt(rule.dateColumn, cutoff.toISOString());

    if (!error) {
      results.push({ table: rule.table, deleted: count || 0 });
    }
  }

  return results;
}
```

### Cron Trigger

```typescript
// app/api/cron/retention/route.ts
import { runRetentionCleanup } from "@/lib/gdpr/data-retention";

export async function GET(req: Request) {
  // Verify cron secret (Vercel Cron or custom)
  const authHeader = req.headers.get("authorization");
  if (authHeader !== `Bearer ${process.env.CRON_SECRET}`) {
    return Response.json({ error: "Unauthorized" }, { status: 401 });
  }

  const results = await runRetentionCleanup();
  return Response.json({ cleaned: results });
}
```

```json
// vercel.json
{
  "crons": [
    {
      "path": "/api/cron/retention",
      "schedule": "0 3 * * *"
    }
  ]
}
```
