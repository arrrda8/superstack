# Tracking Pixels & Privacy-Compliant Analytics

## Table of Contents
- [1. Architecture Overview](#1-architecture-overview)
- [2. Google Consent Mode v2 (MANDATORY)](#2-google-consent-mode-v2-mandatory)
- [3. Google Tag Manager Integration](#3-google-tag-manager-integration)
- [4. Facebook/Meta Pixel + Conversions API](#4-facebookmeta-pixel--conversions-api)
  - [Shared utilities](#shared-utilities)
  - [Client-side tracking helper](#client-side-tracking-helper)
  - [Server-side Meta CAPI route](#server-side-meta-capi-route)
- [5. TikTok Pixel + Events API](#5-tiktok-pixel--events-api)
- [6. LinkedIn Insight Tag + Conversions API](#6-linkedin-insight-tag--conversions-api)
- [7. Privacy-Friendly Analytics Alternatives](#7-privacy-friendly-analytics-alternatives)
  - [Plausible Analytics](#plausible-analytics)
  - [Umami (self-hosted)](#umami-self-hosted)
  - [PostHog (product analytics + session replay)](#posthog-product-analytics--session-replay)
- [8. Critical Rules](#8-critical-rules)

## 1. Architecture Overview

```
Browser (Client)                          Server (Next.js)
+---------------------------+             +---------------------------+
|                           |             |                           |
|  Consent Manager (CMP)    |             |  API Routes               |
|        |                  |             |    /api/track/meta        |
|        v                  |             |    /api/track/tiktok      |
|  GTM Container            |    fetch    |    /api/track/linkedin    |
|    - Meta Pixel       ----+------------>|        |                  |
|    - TikTok Pixel         |             |        v                  |
|    - LinkedIn Pixel       |             |  Server-Side APIs         |
|    - Google Ads           |             |    - Meta CAPI            |
|                           |             |    - TikTok Events API    |
|  dataLayer.push()         |             |    - LinkedIn CAPI        |
|    { event_id: "abc" }    |             |    { event_id: "abc" }    |
|                           |             |    (deduplication)        |
+---------------------------+             +---------------------------+
```

## 2. Google Consent Mode v2 (MANDATORY)

**CRITICAL: German court ruling March 2025 -- loading GTM or any tracking script
without prior consent violates TTDSG Section 25 and GDPR. Default state MUST deny all.**

The consent defaults script uses static string content only (no user input),
which is safe for inline script injection via Next.js Script component.

```typescript
// src/components/tracking/consent-defaults.tsx
import Script from "next/script";

// This MUST load BEFORE any Google tag / GTM script.
// Uses strategy="beforeInteractive" to ensure it runs first.
// The inline script content is a static string -- safe from XSS.
export function ConsentDefaults() {
  const consentScript = [
    "window.dataLayer = window.dataLayer || [];",
    "function gtag(){dataLayer.push(arguments);}",
    "gtag('consent', 'default', {",
    "  'ad_storage': 'denied',",
    "  'ad_user_data': 'denied',",
    "  'ad_personalization': 'denied',",
    "  'analytics_storage': 'denied',",
    "  'functionality_storage': 'denied',",
    "  'personalization_storage': 'denied',",
    "  'security_storage': 'granted',",
    "  'wait_for_update': 500",
    "});",
    "gtag('set', 'ads_data_redaction', true);",
    "gtag('set', 'url_passthrough', true);",
  ].join("\n");

  return (
    <Script
      id="consent-defaults"
      strategy="beforeInteractive"
    >
      {consentScript}
    </Script>
  );
}
```

```typescript
// src/lib/tracking/consent.ts

type ConsentChoices = {
  analytics: boolean;
  marketing: boolean;
  functionality: boolean;
};

export function updateConsent(choices: ConsentChoices) {
  // Called by your CMP (cookie banner) when user makes a choice
  if (typeof window === "undefined" || !window.gtag) return;

  window.gtag("consent", "update", {
    analytics_storage: choices.analytics ? "granted" : "denied",
    ad_storage: choices.marketing ? "granted" : "denied",
    ad_user_data: choices.marketing ? "granted" : "denied",
    ad_personalization: choices.marketing ? "granted" : "denied",
    functionality_storage: choices.functionality ? "granted" : "denied",
    personalization_storage: choices.functionality ? "granted" : "denied",
  });
}

// Extend Window for TypeScript
declare global {
  interface Window {
    gtag: (...args: unknown[]) => void;
    dataLayer: Record<string, unknown>[];
    fbq: (...args: unknown[]) => void;
    ttq: { track: (...args: unknown[]) => void; identify: (...args: unknown[]) => void };
  }
}
```

## 3. Google Tag Manager Integration

```typescript
// src/components/tracking/gtm.tsx
"use client";

import Script from "next/script";

const GTM_ID = process.env.NEXT_PUBLIC_GTM_ID;

export function GoogleTagManager() {
  if (!GTM_ID) return null;

  // GTM loader script -- static content, safe for inline use
  const gtmScript = [
    "(function(w,d,s,l,i){w[l]=w[l]||[];w[l].push({'gtm.start':",
    "new Date().getTime(),event:'gtm.js'});var f=d.getElementsByTagName(s)[0],",
    "j=d.createElement(s),dl=l!='dataLayer'?'&l='+l:'';j.async=true;j.src=",
    `'https://www.googletagmanager.com/gtm.js?id='+i+dl;f.parentNode.insertBefore(j,f);`,
    `})(window,document,'script','dataLayer','${GTM_ID}');`,
  ].join("\n");

  return (
    <>
      <Script id="gtm-script" strategy="afterInteractive">
        {gtmScript}
      </Script>
      {/* GTM noscript fallback */}
      <noscript>
        <iframe
          src={`https://www.googletagmanager.com/ns.html?id=${GTM_ID}`}
          height="0"
          width="0"
          style={{ display: "none", visibility: "hidden" }}
        />
      </noscript>
    </>
  );
}
```

```typescript
// src/app/layout.tsx (integration)
import { ConsentDefaults } from "@/components/tracking/consent-defaults";
import { GoogleTagManager } from "@/components/tracking/gtm";

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="de">
      <head>
        {/* Consent defaults MUST come first */}
        <ConsentDefaults />
      </head>
      <body>
        <GoogleTagManager />
        {children}
      </body>
    </html>
  );
}
```

## 4. Facebook/Meta Pixel + Conversions API

### Shared utilities

```typescript
// src/lib/tracking/utils.ts
import { createHash, randomUUID } from "crypto";

export function generateEventId(): string {
  return randomUUID();
}

export function hashPII(value: string): string {
  return createHash("sha256")
    .update(value.trim().toLowerCase())
    .digest("hex");
}

export function getClientIp(request: Request): string {
  return (
    request.headers.get("x-forwarded-for")?.split(",")[0]?.trim() ||
    request.headers.get("x-real-ip") ||
    "0.0.0.0"
  );
}
```

### Client-side tracking helper

```typescript
// src/lib/tracking/client.ts
"use client";

type TrackEventOptions = {
  eventName: string;
  data?: Record<string, unknown>;
  userData?: { email?: string; phone?: string };
};

export async function trackEvent({ eventName, data = {}, userData }: TrackEventOptions) {
  const eventId = crypto.randomUUID(); // browser API

  // 1. Push to dataLayer (GTM fires client pixels if consent granted)
  window.dataLayer?.push({
    event: eventName,
    event_id: eventId,
    ...data,
  });

  // 2. Send to server for CAPI (always fires, even if client pixel blocked)
  try {
    await fetch("/api/track", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        eventName,
        eventId,
        data,
        userData,
        sourceUrl: window.location.href,
        userAgent: navigator.userAgent,
        fbp: getCookie("_fbp"),
        fbc: getCookie("_fbc"),
      }),
    });
  } catch {
    // Tracking should never break the app
  }
}

function getCookie(name: string): string | undefined {
  return document.cookie
    .split("; ")
    .find((row) => row.startsWith(`${name}=`))
    ?.split("=")[1];
}
```

### Server-side Meta CAPI route

```typescript
// src/app/api/track/meta/route.ts
import { NextRequest, NextResponse } from "next/server";
import { hashPII, getClientIp } from "@/lib/tracking/utils";

const META_PIXEL_ID = process.env.META_PIXEL_ID!;
const META_ACCESS_TOKEN = process.env.META_PIXEL_TOKEN!;
const META_API_VERSION = "v21.0";

export async function POST(request: NextRequest) {
  const body = await request.json();
  const { eventName, eventId, data, userData, sourceUrl, userAgent, fbp, fbc } = body;

  const eventData = {
    data: [
      {
        event_name: eventName,
        event_id: eventId, // MUST match client-side event_id for deduplication
        event_time: Math.floor(Date.now() / 1000),
        event_source_url: sourceUrl,
        action_source: "website",
        user_data: {
          client_ip_address: getClientIp(request),
          client_user_agent: userAgent,
          ...(fbp && { fbp }),
          ...(fbc && { fbc }),
          ...(userData?.email && { em: hashPII(userData.email) }),
          ...(userData?.phone && { ph: hashPII(userData.phone) }),
        },
        custom_data: data,
      },
    ],
  };

  const response = await fetch(
    `https://graph.facebook.com/${META_API_VERSION}/${META_PIXEL_ID}/events?access_token=${META_ACCESS_TOKEN}`,
    {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(eventData),
    }
  );

  const result = await response.json();
  return NextResponse.json(result);
}
```

## 5. TikTok Pixel + Events API

```typescript
// src/app/api/track/tiktok/route.ts
import { NextRequest, NextResponse } from "next/server";
import { hashPII, getClientIp } from "@/lib/tracking/utils";

const TIKTOK_PIXEL_ID = process.env.TIKTOK_PIXEL_ID!;
const TIKTOK_ACCESS_TOKEN = process.env.TIKTOK_ACCESS_TOKEN!;

export async function POST(request: NextRequest) {
  const body = await request.json();
  const { eventName, eventId, data, userData, sourceUrl, userAgent } = body;

  const payload = {
    pixel_code: TIKTOK_PIXEL_ID,
    event: eventName,
    event_id: eventId, // deduplication with client pixel
    timestamp: new Date().toISOString(),
    context: {
      page: { url: sourceUrl },
      user_agent: userAgent,
      ip: getClientIp(request),
      user: {
        ...(userData?.email && { email: hashPII(userData.email) }),
        ...(userData?.phone && { phone_number: hashPII(userData.phone) }),
      },
    },
    properties: data,
  };

  const response = await fetch(
    "https://business-api.tiktok.com/open_api/v1.3/event/track/",
    {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        "Access-Token": TIKTOK_ACCESS_TOKEN,
      },
      body: JSON.stringify({ batch: [payload] }),
    }
  );

  const result = await response.json();
  return NextResponse.json(result);
}
```

## 6. LinkedIn Insight Tag + Conversions API

```typescript
// src/app/api/track/linkedin/route.ts
import { NextRequest, NextResponse } from "next/server";
import { hashPII } from "@/lib/tracking/utils";

const LINKEDIN_AD_ACCOUNT_ID = process.env.LINKEDIN_AD_ACCOUNT_ID!;
const LINKEDIN_ACCESS_TOKEN = process.env.LINKEDIN_ACCESS_TOKEN!;
const LINKEDIN_CONVERSION_ID = process.env.LINKEDIN_CONVERSION_ID!;

export async function POST(request: NextRequest) {
  const body = await request.json();
  const { eventName, eventId, userData } = body;

  const payload = {
    conversion: `urn:lla:llaPartnerConversion:${LINKEDIN_CONVERSION_ID}`,
    conversionHappenedAt: Date.now(),
    conversionValue: { currencyCode: "EUR", amount: "0" },
    eventId,
    user: {
      userIds: [
        ...(userData?.email
          ? [{ idType: "SHA256_EMAIL", idValue: hashPII(userData.email) }]
          : []),
      ],
      userInfo: {
        ...(userData?.firstName && { firstName: userData.firstName }),
        ...(userData?.lastName && { lastName: userData.lastName }),
        ...(userData?.company && { companyName: userData.company }),
      },
    },
  };

  const response = await fetch(
    "https://api.linkedin.com/rest/conversionEvents",
    {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        Authorization: `Bearer ${LINKEDIN_ACCESS_TOKEN}`,
        "LinkedIn-Version": "202411",
        "X-Restli-Protocol-Version": "2.0.0",
      },
      body: JSON.stringify({ elements: [payload] }),
    }
  );

  const result = await response.json();
  return NextResponse.json(result);
}
```

## 7. Privacy-Friendly Analytics Alternatives

These tools do NOT require consent if configured without cookies/PII:

### Plausible Analytics

```typescript
// src/components/tracking/plausible.tsx
import Script from "next/script";

export function PlausibleAnalytics() {
  const domain = process.env.NEXT_PUBLIC_PLAUSIBLE_DOMAIN;
  if (!domain) return null;

  return (
    <Script
      src="https://plausible.io/js/script.js"
      data-domain={domain}
      strategy="afterInteractive"
      defer
    />
  );
}

// Custom event tracking:
// window.plausible('Signup', { props: { plan: 'pro' } })
```

### Umami (self-hosted)

```typescript
// src/components/tracking/umami.tsx
import Script from "next/script";

export function UmamiAnalytics() {
  const websiteId = process.env.NEXT_PUBLIC_UMAMI_WEBSITE_ID;
  const src = process.env.NEXT_PUBLIC_UMAMI_SRC; // your self-hosted URL
  if (!websiteId || !src) return null;

  return (
    <Script
      src={src}
      data-website-id={websiteId}
      strategy="afterInteractive"
      defer
    />
  );
}
```

### PostHog (product analytics + session replay)

```typescript
// src/lib/tracking/posthog.ts
"use client";

import posthog from "posthog-js";

export function initPostHog() {
  if (typeof window === "undefined") return;

  posthog.init(process.env.NEXT_PUBLIC_POSTHOG_KEY!, {
    api_host: process.env.NEXT_PUBLIC_POSTHOG_HOST || "https://eu.i.posthog.com",
    person_profiles: "identified_only",
    capture_pageview: false, // handle manually in App Router
    persistence: "memory", // no cookies = no consent needed
  });
}

// Use in layout:
// initPostHog();
// posthog.capture('$pageview');
```

## 8. Critical Rules

1. **Consent BEFORE tracking (always)**
   - Load consent defaults with all denied BEFORE any tag/pixel
   - GTM tags must be configured to fire only when consent is granted
   - No tracking cookies may be set before explicit opt-in

2. **Event deduplication**
   - Generate ONE `event_id` per event
   - Send the SAME `event_id` to both client pixel (via dataLayer) and server CAPI
   - Meta, TikTok, and LinkedIn all support deduplication via event_id

3. **Hash PII server-side**
   - Use SHA256 for emails, phone numbers, names
   - Trim and lowercase before hashing
   - Never send raw PII to third-party APIs

4. **Progressive enhancement**
   - Server-side CAPI fires regardless of ad blockers or client-side consent
   - Client pixels enhance match quality when consent is granted
   - Track on server even when browser blocks client-side scripts

5. **GDPR compliance checklist**
   - Informed consent: clear explanation of what is tracked and why
   - Granular opt-in: separate toggles for analytics vs. marketing
   - Easy withdrawal: user can revoke consent at any time
   - Data minimization: only collect what you actually need
   - Privacy policy must list all tracking tools and their purpose
   - Cookie banner must not use dark patterns (no pre-checked boxes)
