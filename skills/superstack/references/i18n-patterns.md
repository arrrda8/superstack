# Internationalization (i18n) Patterns — Next.js + next-intl

## Table of Contents
- [1. next-intl Setup](#1-next-intl-setup)
  - [Install](#install)
  - [Routing Config (`src/i18n/routing.ts`)](#routing-config-srci18nroutingts)
  - [Middleware (`src/middleware.ts`)](#middleware-srcmiddlewarets)
  - [Request Config (`src/i18n/request.ts`)](#request-config-srci18nrequestts)
  - [Next.js Config (`next.config.ts`)](#nextjs-config-nextconfigts)
  - [Layout (`src/app/[locale]/layout.tsx`)](#layout-srcapplocalelayouttsx)
- [2. Message Files](#2-message-files)
  - [Structure (`messages/de.json`)](#structure-messagesdejson)
  - [Corresponding English (`messages/en.json`)](#corresponding-english-messagesenjson)
  - [Type Safety (`src/i18n/types.ts`)](#type-safety-srci18ntypests)
- [3. Server Components](#3-server-components)
- [4. Client Components](#4-client-components)
  - [Rich Text / HTML in Translations](#rich-text--html-in-translations)
  - [Selective Provider (reduce client bundle)](#selective-provider-reduce-client-bundle)
- [5. Pluralization & Formatting](#5-pluralization--formatting)
  - [Plural Rules (ICU)](#plural-rules-icu)
  - [Number, Date, Currency Formatting](#number-date-currency-formatting)
  - [Server-side Formatting](#server-side-formatting)
- [6. Language Switcher](#6-language-switcher)
  - [Dropdown Variant (shadcn/ui)](#dropdown-variant-shadcnui)
- [7. SEO](#7-seo)
  - [Localized Metadata (`src/app/[locale]/layout.tsx`)](#localized-metadata-srcapplocalelayouttsx)
  - [Sitemap per Locale (`src/app/sitemap.ts`)](#sitemap-per-locale-srcappsitemapts)
- [8. Translation Workflow](#8-translation-workflow)
  - [Extract Keys Script (`scripts/check-i18n.ts`)](#extract-keys-script-scriptscheck-i18nts)
  - [CI Check (GitHub Actions)](#ci-check-github-actions)
  - [Platform Integration (Crowdin / Lokalise)](#platform-integration-crowdin--lokalise)
- [9. RTL Support](#9-rtl-support)
  - [Tailwind Config (`tailwind.config.ts`)](#tailwind-config-tailwindconfigts)
  - [Logical CSS Properties](#logical-css-properties)
  - [RTL-Aware Layout](#rtl-aware-layout)
  - [Common RTL Gotchas](#common-rtl-gotchas)

Production-ready patterns for multilingual Next.js App Router applications using next-intl.

---

## 1. next-intl Setup

### Install

```bash
bun add next-intl
```

### Routing Config (`src/i18n/routing.ts`)

```typescript
import { defineRouting } from "next-intl/routing";
import { createNavigation } from "next-intl/navigation";

export const routing = defineRouting({
  locales: ["de", "en", "fr"] as const,
  defaultLocale: "de",
  localePrefix: "as-needed", // "always" | "as-needed" | "never"
  pathnames: {
    "/about": {
      de: "/ueber-uns",
      en: "/about",
      fr: "/a-propos",
    },
    "/products/[slug]": {
      de: "/produkte/[slug]",
      en: "/products/[slug]",
      fr: "/produits/[slug]",
    },
  },
});

export type Locale = (typeof routing.locales)[number];
export type Pathnames = keyof typeof routing.pathnames;

export const { Link, redirect, usePathname, useRouter, getPathname } =
  createNavigation(routing);
```

### Middleware (`src/middleware.ts`)

```typescript
import createMiddleware from "next-intl/middleware";
import { routing } from "@/i18n/routing";

export default createMiddleware(routing);

export const config = {
  matcher: [
    // Match all pathnames except API routes, _next, and static files
    "/((?!api|_next|_vercel|.*\\..*).*)",
  ],
};
```

### Request Config (`src/i18n/request.ts`)

```typescript
import { getRequestConfig } from "next-intl/server";
import { routing } from "./routing";

export default getRequestConfig(async ({ requestLocale }) => {
  let locale = await requestLocale;

  if (!locale || !routing.locales.includes(locale as any)) {
    locale = routing.defaultLocale;
  }

  return {
    locale,
    messages: (await import(`../../messages/${locale}.json`)).default,
  };
});
```

### Next.js Config (`next.config.ts`)

```typescript
import createNextIntlPlugin from "next-intl/plugin";

const withNextIntl = createNextIntlPlugin("./src/i18n/request.ts");

const nextConfig = {
  // your config
};

export default withNextIntl(nextConfig);
```

### Layout (`src/app/[locale]/layout.tsx`)

```typescript
import { NextIntlClientProvider } from "next-intl";
import { getMessages, setRequestLocale } from "next-intl/server";
import { routing } from "@/i18n/routing";
import { notFound } from "next/navigation";

export function generateStaticParams() {
  return routing.locales.map((locale) => ({ locale }));
}

export default async function LocaleLayout({
  children,
  params,
}: {
  children: React.ReactNode;
  params: Promise<{ locale: string }>;
}) {
  const { locale } = await params;

  if (!routing.locales.includes(locale as any)) {
    notFound();
  }

  setRequestLocale(locale);
  const messages = await getMessages();

  return (
    <html lang={locale} dir={locale === "ar" ? "rtl" : "ltr"}>
      <body>
        <NextIntlClientProvider messages={messages}>
          {children}
        </NextIntlClientProvider>
      </body>
    </html>
  );
}
```

---

## 2. Message Files

### Structure (`messages/de.json`)

```json
{
  "common": {
    "loading": "Laden...",
    "save": "Speichern",
    "cancel": "Abbrechen",
    "delete": "Loeschen",
    "confirm": "Bestaetigen",
    "back": "Zurueck",
    "next": "Weiter",
    "error": "Ein Fehler ist aufgetreten"
  },
  "auth": {
    "login": "Anmelden",
    "logout": "Abmelden",
    "email": "E-Mail-Adresse",
    "password": "Passwort",
    "forgotPassword": "Passwort vergessen?",
    "noAccount": "Noch kein Konto? {signupLink}"
  },
  "dashboard": {
    "title": "Dashboard",
    "welcome": "Willkommen, {name}!",
    "stats": {
      "revenue": "Umsatz",
      "users": "Benutzer",
      "orders": "Bestellungen"
    }
  },
  "notifications": {
    "count": "Du hast {count, plural, =0 {keine Benachrichtigungen} one {eine Benachrichtigung} other {# Benachrichtigungen}}.",
    "newMessage": "{sender} hat dir eine Nachricht gesendet"
  },
  "pricing": {
    "price": "{amount, number, ::currency/EUR}",
    "perMonth": "{amount, number, ::currency/EUR} / Monat"
  }
}
```

### Corresponding English (`messages/en.json`)

```json
{
  "common": {
    "loading": "Loading...",
    "save": "Save",
    "cancel": "Cancel",
    "delete": "Delete",
    "confirm": "Confirm",
    "back": "Back",
    "next": "Next",
    "error": "An error occurred"
  },
  "auth": {
    "login": "Sign in",
    "logout": "Sign out",
    "email": "Email address",
    "password": "Password",
    "forgotPassword": "Forgot password?",
    "noAccount": "No account yet? {signupLink}"
  },
  "dashboard": {
    "title": "Dashboard",
    "welcome": "Welcome, {name}!",
    "stats": {
      "revenue": "Revenue",
      "users": "Users",
      "orders": "Orders"
    }
  },
  "notifications": {
    "count": "You have {count, plural, =0 {no notifications} one {one notification} other {# notifications}}.",
    "newMessage": "{sender} sent you a message"
  },
  "pricing": {
    "price": "{amount, number, ::currency/EUR}",
    "perMonth": "{amount, number, ::currency/EUR} / month"
  }
}
```

### Type Safety (`src/i18n/types.ts`)

```typescript
// Generate types from default locale messages
import de from "../../messages/de.json";

type Messages = typeof de;

declare global {
  interface IntlMessages extends Messages {}
}
```

---

## 3. Server Components

```typescript
// src/app/[locale]/dashboard/page.tsx
import { getTranslations, setRequestLocale } from "next-intl/server";

interface Props {
  params: Promise<{ locale: string }>;
}

export async function generateMetadata({ params }: Props) {
  const { locale } = await params;
  const t = await getTranslations({ locale, namespace: "dashboard" });

  return {
    title: t("title"),
  };
}

export default async function DashboardPage({ params }: Props) {
  const { locale } = await params;
  setRequestLocale(locale);

  const t = await getTranslations("dashboard");
  const tCommon = await getTranslations("common");

  return (
    <div>
      <h1>{t("title")}</h1>
      <p>{t("welcome", { name: "Erdal" })}</p>

      <div className="grid grid-cols-3 gap-4">
        <StatCard label={t("stats.revenue")} value="€124,500" />
        <StatCard label={t("stats.users")} value="1,234" />
        <StatCard label={t("stats.orders")} value="567" />
      </div>

      <button>{tCommon("save")}</button>
    </div>
  );
}
```

---

## 4. Client Components

```typescript
"use client";

import { useTranslations, useLocale } from "next-intl";

export function NotificationBadge({ count }: { count: number }) {
  const t = useTranslations("notifications");
  const locale = useLocale();

  return (
    <div>
      <p>{t("count", { count })}</p>
      <span className="text-sm text-muted-foreground">
        {locale === "de" ? "Aktuell" : "Current"}
      </span>
    </div>
  );
}
```

### Rich Text / HTML in Translations

```typescript
"use client";

import { useTranslations } from "next-intl";

export function AuthPrompt() {
  const t = useTranslations("auth");

  return (
    <p>
      {t.rich("noAccount", {
        signupLink: (chunks) => (
          <a href="/signup" className="text-primary underline">
            {chunks}
          </a>
        ),
      })}
    </p>
  );
}
```

### Selective Provider (reduce client bundle)

```typescript
// Only pass specific namespace messages to client
import { NextIntlClientProvider } from "next-intl";
import { getMessages } from "next-intl/server";
import { pick } from "lodash-es";

export default async function DashboardLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  const messages = await getMessages();

  return (
    <NextIntlClientProvider messages={pick(messages, ["common", "dashboard"])}>
      {children}
    </NextIntlClientProvider>
  );
}
```

---

## 5. Pluralization & Formatting

### Plural Rules (ICU)

```json
{
  "items": "{count, plural, =0 {Keine Eintraege} one {Ein Eintrag} other {# Eintraege}}",
  "daysLeft": "{days, plural, =0 {Heute} one {Noch ein Tag} other {Noch # Tage}}"
}
```

### Number, Date, Currency Formatting

```typescript
"use client";

import { useFormatter, useNow, useTimeZone } from "next-intl";

export function FormattedValues() {
  const format = useFormatter();
  const now = useNow();
  const timeZone = useTimeZone();

  return (
    <div>
      {/* Number */}
      <p>{format.number(1234567.89)}</p>
      {/* de: 1.234.567,89 | en: 1,234,567.89 */}

      {/* Currency */}
      <p>{format.number(49.99, { style: "currency", currency: "EUR" })}</p>
      {/* de: 49,99 € | en: €49.99 */}

      {/* Percent */}
      <p>{format.number(0.156, { style: "percent" })}</p>
      {/* de: 16 % | en: 16% */}

      {/* Date */}
      <p>
        {format.dateTime(now, {
          year: "numeric",
          month: "long",
          day: "numeric",
        })}
      </p>
      {/* de: 7. Maerz 2026 | en: March 7, 2026 */}

      {/* Relative time */}
      <p>
        {format.relativeTime(new Date("2026-03-01"), now)}
      </p>
      {/* de: vor 6 Tagen | en: 6 days ago */}

      {/* List */}
      <p>{format.list(["React", "Next.js", "TypeScript"], { type: "conjunction" })}</p>
      {/* de: React, Next.js und TypeScript | en: React, Next.js, and TypeScript */}
    </div>
  );
}
```

### Server-side Formatting

```typescript
import { getFormatter } from "next-intl/server";

export default async function InvoicePage() {
  const format = await getFormatter();

  const total = format.number(1299.99, {
    style: "currency",
    currency: "EUR",
  });

  const date = format.dateTime(new Date(), {
    dateStyle: "long",
  });

  return (
    <div>
      <p>Total: {total}</p>
      <p>Date: {date}</p>
    </div>
  );
}
```

---

## 6. Language Switcher

```typescript
"use client";

import { useLocale } from "next-intl";
import { useRouter, usePathname } from "@/i18n/routing";
import { routing, type Locale } from "@/i18n/routing";

const localeLabels: Record<Locale, string> = {
  de: "Deutsch",
  en: "English",
  fr: "Francais",
};

const localeFlags: Record<Locale, string> = {
  de: "DE",
  en: "GB",
  fr: "FR",
};

export function LanguageSwitcher() {
  const locale = useLocale() as Locale;
  const router = useRouter();
  const pathname = usePathname();

  function onChange(newLocale: Locale) {
    router.replace(pathname, { locale: newLocale });
  }

  return (
    <div className="relative">
      <select
        value={locale}
        onChange={(e) => onChange(e.target.value as Locale)}
        className="appearance-none rounded-md border bg-background px-3 py-2 pr-8 text-sm"
      >
        {routing.locales.map((loc) => (
          <option key={loc} value={loc}>
            {localeFlags[loc]} {localeLabels[loc]}
          </option>
        ))}
      </select>
    </div>
  );
}
```

### Dropdown Variant (shadcn/ui)

```typescript
"use client";

import { useLocale } from "next-intl";
import { useRouter, usePathname } from "@/i18n/routing";
import { routing, type Locale } from "@/i18n/routing";
import {
  DropdownMenu,
  DropdownMenuContent,
  DropdownMenuItem,
  DropdownMenuTrigger,
} from "@/components/ui/dropdown-menu";
import { Button } from "@/components/ui/button";
import { Globe } from "lucide-react";

export function LanguageSwitcherDropdown() {
  const locale = useLocale() as Locale;
  const router = useRouter();
  const pathname = usePathname();

  return (
    <DropdownMenu>
      <DropdownMenuTrigger asChild>
        <Button variant="ghost" size="icon">
          <Globe className="h-4 w-4" />
        </Button>
      </DropdownMenuTrigger>
      <DropdownMenuContent align="end">
        {routing.locales.map((loc) => (
          <DropdownMenuItem
            key={loc}
            onClick={() => router.replace(pathname, { locale: loc })}
            className={locale === loc ? "bg-accent" : ""}
          >
            {localeLabels[loc]}
          </DropdownMenuItem>
        ))}
      </DropdownMenuContent>
    </DropdownMenu>
  );
}
```

---

## 7. SEO

### Localized Metadata (`src/app/[locale]/layout.tsx`)

```typescript
import { getTranslations } from "next-intl/server";
import { routing, getPathname } from "@/i18n/routing";
import type { Metadata } from "next";

export async function generateMetadata({
  params,
}: {
  params: Promise<{ locale: string }>;
}): Promise<Metadata> {
  const { locale } = await params;
  const t = await getTranslations({ locale, namespace: "metadata" });

  const baseUrl = "https://example.com";

  // Build alternates for hreflang
  const languages: Record<string, string> = {};
  for (const loc of routing.locales) {
    const path = getPathname({ locale: loc, href: "/" });
    languages[loc] = `${baseUrl}${path}`;
  }

  return {
    title: {
      default: t("title"),
      template: `%s | ${t("siteName")}`,
    },
    description: t("description"),
    alternates: {
      canonical: `${baseUrl}/${locale}`,
      languages,
    },
    openGraph: {
      title: t("title"),
      description: t("description"),
      locale: locale,
      alternateLocale: routing.locales.filter((l) => l !== locale),
    },
  };
}
```

### Sitemap per Locale (`src/app/sitemap.ts`)

```typescript
import { routing, getPathname } from "@/i18n/routing";
import type { MetadataRoute } from "next";

const baseUrl = "https://example.com";
const pages = ["/", "/about", "/pricing", "/contact"] as const;

export default function sitemap(): MetadataRoute.Sitemap {
  const entries: MetadataRoute.Sitemap = [];

  for (const page of pages) {
    for (const locale of routing.locales) {
      const path = getPathname({ locale, href: page });

      const alternates: Record<string, string> = {};
      for (const altLocale of routing.locales) {
        const altPath = getPathname({ locale: altLocale, href: page });
        alternates[altLocale] = `${baseUrl}${altPath}`;
      }

      entries.push({
        url: `${baseUrl}${path}`,
        lastModified: new Date(),
        changeFrequency: "weekly",
        priority: page === "/" ? 1 : 0.8,
        alternates: { languages: alternates },
      });
    }
  }

  return entries;
}
```

---

## 8. Translation Workflow

### Extract Keys Script (`scripts/check-i18n.ts`)

```typescript
// scripts/check-i18n.ts — Run with: bun scripts/check-i18n.ts
import fs from "fs";
import path from "path";

const messagesDir = path.resolve("messages");
const defaultLocale = "de";

function flattenKeys(obj: Record<string, any>, prefix = ""): string[] {
  return Object.entries(obj).flatMap(([key, value]) => {
    const fullKey = prefix ? `${prefix}.${key}` : key;
    if (typeof value === "object" && value !== null) {
      return flattenKeys(value, fullKey);
    }
    return [fullKey];
  });
}

const files = fs.readdirSync(messagesDir).filter((f) => f.endsWith(".json"));
const allKeys: Record<string, Set<string>> = {};

for (const file of files) {
  const locale = file.replace(".json", "");
  const content = JSON.parse(
    fs.readFileSync(path.join(messagesDir, file), "utf-8")
  );
  allKeys[locale] = new Set(flattenKeys(content));
}

const defaultKeys = allKeys[defaultLocale];
let hasErrors = false;

for (const [locale, keys] of Object.entries(allKeys)) {
  if (locale === defaultLocale) continue;

  const missing = [...defaultKeys].filter((k) => !keys.has(k));
  const extra = [...keys].filter((k) => !defaultKeys.has(k));

  if (missing.length > 0) {
    console.error(`\n[${locale}] Missing ${missing.length} keys:`);
    missing.forEach((k) => console.error(`  - ${k}`));
    hasErrors = true;
  }

  if (extra.length > 0) {
    console.warn(`\n[${locale}] Extra ${extra.length} keys:`);
    extra.forEach((k) => console.warn(`  + ${k}`));
  }
}

if (hasErrors) {
  process.exit(1);
} else {
  console.log("All translations are in sync.");
}
```

### CI Check (GitHub Actions)

```yaml
# .github/workflows/i18n-check.yml
name: i18n Check
on: [pull_request]
jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: oven-sh/setup-bun@v2
      - run: bun scripts/check-i18n.ts
```

### Platform Integration (Crowdin / Lokalise)

```typescript
// crowdin.yml — place in project root
project_id: "YOUR_PROJECT_ID"
api_token_env: "CROWDIN_TOKEN"
base_path: "."
base_url: "https://api.crowdin.com"

files:
  - source: "/messages/de.json"
    translation: "/messages/%two_letters_code%.json"
    type: "json"
```

---

## 9. RTL Support

### Tailwind Config (`tailwind.config.ts`)

```typescript
import type { Config } from "tailwindcss";
import rtlPlugin from "tailwindcss-rtl";

export default {
  plugins: [rtlPlugin],
} satisfies Config;
```

### Logical CSS Properties

```typescript
// Use logical properties instead of physical direction
// Instead of: pl-4 pr-2 ml-auto mr-0 text-left border-l-2
// Use:        ps-4 pe-2 ms-auto me-0 text-start border-s-2

export function SidebarItem({ label }: { label: string }) {
  return (
    <div className="flex items-center gap-3 ps-4 pe-2 border-s-2 border-primary">
      <span className="text-start">{label}</span>
      {/* ChevronRight auto-flips in RTL if using logical transform */}
      <ChevronRight className="ms-auto rtl:rotate-180" />
    </div>
  );
}
```

### RTL-Aware Layout

```typescript
import { useLocale } from "next-intl";

const rtlLocales = ["ar", "he", "fa", "ur"];

export function useDirection() {
  const locale = useLocale();
  return rtlLocales.includes(locale) ? "rtl" : "ltr";
}

// In layout:
export default async function LocaleLayout({
  children,
  params,
}: {
  children: React.ReactNode;
  params: Promise<{ locale: string }>;
}) {
  const { locale } = await params;
  const dir = rtlLocales.includes(locale) ? "rtl" : "ltr";

  return (
    <html lang={locale} dir={dir}>
      <body className={dir === "rtl" ? "font-arabic" : "font-sans"}>
        {children}
      </body>
    </html>
  );
}
```

### Common RTL Gotchas

```typescript
// 1. Icons that imply direction need manual flip
<ArrowLeft className="rtl:rotate-180" />

// 2. Swiper / carousels need dir="ltr" forced or rtl option
<Swiper dir="ltr" /> // or configure RTL mode

// 3. Number inputs — Arabic/Persian numerals
// Use Intl.NumberFormat for display, keep internal values as standard numbers

// 4. CSS Grid / Flexbox — already direction-aware with logical properties
// flex-row works correctly in both LTR and RTL
```
