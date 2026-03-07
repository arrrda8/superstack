# Performance Patterns — Next.js (App Router)

## Table of Contents

- [1. next/image Optimization](#1-nextimage-optimization)
  - [Basic responsive image](#basic-responsive-image)
  - [AVIF/WebP format control](#avifwebp-format-control)
  - [Custom loader for Supabase Storage](#custom-loader-for-supabase-storage)
  - [Decision matrix](#decision-matrix)
- [2. Font Loading](#2-font-loading)
  - [Google Fonts (variable)](#google-fonts-variable)
  - [Local fonts](#local-fonts)
  - [Tailwind config for variable fonts](#tailwind-config-for-variable-fonts)
- [3. Code Splitting](#3-code-splitting)
  - [Dynamic imports](#dynamic-imports)
  - [React.lazy + Suspense (client components)](#reactlazy-suspense-client-components)
  - [Conditional dynamic import](#conditional-dynamic-import)
  - [Barrel file anti-pattern](#barrel-file-anti-pattern)
- [4. ISR / SSG / PPR](#4-isr-ssg-ppr)
  - [Decision matrix](#decision-matrix)
  - [ISR with time-based revalidation](#isr-with-time-based-revalidation)
  - [On-demand revalidation (webhook)](#on-demand-revalidation-webhook)
  - [Partial Prerendering (Next 15)](#partial-prerendering-next-15)
- [5. Cache Headers](#5-cache-headers)
  - [Static assets (immutable)](#static-assets-immutable)
  - [API Route caching](#api-route-caching)
  - [Common patterns](#common-patterns)
- [6. Bundle Analysis](#6-bundle-analysis)
  - [Setup](#setup)
  - [Tree-shaking tips](#tree-shaking-tips)
  - [Check for duplicate dependencies](#check-for-duplicate-dependencies)
  - [Common bundle bloaters and replacements](#common-bundle-bloaters-and-replacements)
- [7. Core Web Vitals Debugging](#7-core-web-vitals-debugging)
  - [Setup web-vitals reporting](#setup-web-vitals-reporting)
  - [LCP (< 2.5s) — Largest Contentful Paint](#lcp-25s-largest-contentful-paint)
  - [CLS (< 0.1) — Cumulative Layout Shift](#cls-01-cumulative-layout-shift)
  - [INP (< 200ms) — Interaction to Next Paint](#inp-200ms-interaction-to-next-paint)
  - [Lighthouse CI (CI/CD integration)](#lighthouse-ci-cicd-integration)
- [8. Runtime Performance](#8-runtime-performance)
  - [memo / useMemo / useCallback — when actually needed](#memo-usememo-usecallback-when-actually-needed)
  - [Virtualization (TanStack Virtual)](#virtualization-tanstack-virtual)
  - [Debounce / Throttle](#debounce-throttle)
- [9. Edge Functions](#9-edge-functions)
  - [Decision matrix](#decision-matrix)
  - [Edge middleware (runs before every matched request)](#edge-middleware-runs-before-every-matched-request)
  - [Edge API route](#edge-api-route)
- [10. Prefetching](#10-prefetching)
  - [Link prefetch (automatic in viewport)](#link-prefetch-automatic-in-viewport)
  - [Programmatic prefetch](#programmatic-prefetch)
  - [Data prefetching in RSC](#data-prefetching-in-rsc)
  - [Preloading patterns with `cache()`](#preloading-patterns-with-cache)

Practical patterns for production web performance optimization.

---

## 1. next/image Optimization

### Basic responsive image

```tsx
import Image from "next/image";

// DO: Use responsive sizes prop matching your layout breakpoints
<Image
  src="/hero.jpg"
  alt="Hero"
  width={1200}
  height={630}
  sizes="(max-width: 768px) 100vw, (max-width: 1200px) 50vw, 1200px"
  priority // above-the-fold only
  placeholder="blur"
  blurDataURL="data:image/jpeg;base64,/9j/4AAQ..." // or import static image
  quality={80}
/>

// DON'T: Use fill without a sized container
<div> {/* Missing position: relative + dimensions */}
  <Image src="/hero.jpg" alt="" fill />
</div>

// DO: Sized container for fill
<div className="relative h-[400px] w-full">
  <Image
    src="/hero.jpg"
    alt=""
    fill
    className="object-cover"
    sizes="100vw"
  />
</div>
```

### AVIF/WebP format control

```ts
// next.config.ts
import type { NextConfig } from "next";

const config: NextConfig = {
  images: {
    formats: ["image/avif", "image/webp"], // AVIF first = smaller, slower encode
    deviceSizes: [640, 750, 828, 1080, 1200, 1920],
    imageSizes: [16, 32, 48, 64, 96, 128, 256],
  },
};

export default config;
```

### Custom loader for Supabase Storage

```ts
// lib/supabase-image-loader.ts
import type { ImageLoader } from "next/image";

const SUPABASE_URL = process.env.NEXT_PUBLIC_SUPABASE_URL!;

export const supabaseLoader: ImageLoader = ({ src, width, quality }) => {
  return `${SUPABASE_URL}/storage/v1/render/image/public/${src}?width=${width}&quality=${quality || 75}`;
};

// Usage
<Image
  loader={supabaseLoader}
  src="bucket-name/path/to/image.jpg"
  alt=""
  width={800}
  height={600}
/>
```

### Decision matrix

| Scenario | `priority` | `loading` | `placeholder` |
|---|---|---|---|
| Hero / LCP image | `true` | - | `"blur"` |
| Above fold, not LCP | - | `"eager"` | `"blur"` |
| Below fold | - | `"lazy"` (default) | `"blur"` or `"empty"` |
| Thumbnails / gallery | - | `"lazy"` | `"blur"` |

---

## 2. Font Loading

### Google Fonts (variable)

```tsx
// app/layout.tsx
import { Inter, JetBrains_Mono } from "next/font/google";

const inter = Inter({
  subsets: ["latin"],       // Only load what you need
  display: "swap",          // Prevent FOIT
  variable: "--font-inter", // CSS variable for Tailwind
});

const mono = JetBrains_Mono({
  subsets: ["latin"],
  display: "swap",
  variable: "--font-mono",
});

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="de" className={`${inter.variable} ${mono.variable}`}>
      <body className="font-sans">{children}</body>
    </html>
  );
}
```

### Local fonts

```tsx
import localFont from "next/font/local";

const geist = localFont({
  src: [
    { path: "./fonts/Geist-Regular.woff2", weight: "400", style: "normal" },
    { path: "./fonts/Geist-Bold.woff2", weight: "700", style: "normal" },
  ],
  display: "swap",
  variable: "--font-geist",
  preload: true, // default true — set false for non-critical fonts
});
```

### Tailwind config for variable fonts

```ts
// tailwind.config.ts
import type { Config } from "tailwindcss";

const config: Config = {
  theme: {
    extend: {
      fontFamily: {
        sans: ["var(--font-inter)", "system-ui", "sans-serif"],
        mono: ["var(--font-mono)", "monospace"],
      },
    },
  },
};

export default config;
```

**DO:**
- Always use `display: "swap"` to prevent invisible text during load
- Use `subsets: ["latin"]` — don't load all character sets
- Use variable fonts (one file instead of multiple weights)
- Use `woff2` for local fonts (best compression)

**DON'T:**
- Import fonts in CSS files — `next/font` self-hosts and optimizes automatically
- Use `preload: true` for fonts only used on specific pages
- Load more than 2-3 font families

---

## 3. Code Splitting

### Dynamic imports

```tsx
import dynamic from "next/dynamic";

// DO: Lazy load heavy components
const HeavyChart = dynamic(() => import("@/components/chart"), {
  loading: () => <div className="h-[400px] animate-pulse bg-muted rounded" />,
  ssr: false, // Skip SSR for client-only libs (e.g., Chart.js, Plotly)
});

// DO: Lazy load below-fold sections
const Testimonials = dynamic(() => import("@/components/testimonials"));
const Footer = dynamic(() => import("@/components/footer"));
```

### React.lazy + Suspense (client components)

```tsx
"use client";

import { lazy, Suspense } from "react";

const Editor = lazy(() => import("@/components/editor"));

export function EditorWrapper() {
  return (
    <Suspense fallback={<EditorSkeleton />}>
      <Editor />
    </Suspense>
  );
}
```

### Conditional dynamic import

```tsx
"use client";

import dynamic from "next/dynamic";
import { useState } from "react";

const Modal = dynamic(() => import("@/components/heavy-modal"));

export function ModalTrigger() {
  const [open, setOpen] = useState(false);

  return (
    <>
      <button onClick={() => setOpen(true)}>Open</button>
      {open && <Modal onClose={() => setOpen(false)} />}
    </>
  );
}
```

### Barrel file anti-pattern

```tsx
// DON'T: barrel exports pull in everything
// components/index.ts
export { Button } from "./button";
export { Modal } from "./modal";       // Heavy — gets bundled even if unused
export { Chart } from "./chart";       // Very heavy

// DO: Import directly from the component file
import { Button } from "@/components/button";
```

**Tip:** Use `@next/bundle-analyzer` to verify barrel files aren't bloating your bundle. In `next.config.ts`, set `optimizePackageImports` for known-safe libraries:

```ts
const config: NextConfig = {
  experimental: {
    optimizePackageImports: ["lucide-react", "@radix-ui/react-icons", "date-fns"],
  },
};
```

---

## 4. ISR / SSG / PPR

### Decision matrix

| Strategy | Use when | Data freshness | Build time |
|---|---|---|---|
| **SSG** (`force-static`) | Content rarely changes (legal, about) | Stale until redeploy | Increases with pages |
| **ISR** (`revalidate: N`) | Content changes periodically (blog, products) | Up to N seconds stale | Fixed |
| **On-demand ISR** | Content changes via CMS/webhook | Near-instant after trigger | Fixed |
| **SSR** (`force-dynamic`) | Personalized / real-time data | Always fresh | N/A |
| **PPR** (Next 15) | Mix of static shell + dynamic parts | Shell cached, dynamic fresh | Fixed |

### ISR with time-based revalidation

```tsx
// app/blog/[slug]/page.tsx

// Page-level revalidation
export const revalidate = 3600; // Revalidate at most every hour

export default async function BlogPost({ params }: { params: Promise<{ slug: string }> }) {
  const { slug } = await params;
  const post = await getPost(slug);
  return <Article post={post} />;
}
```

### On-demand revalidation (webhook)

```ts
// app/api/revalidate/route.ts
import { revalidatePath, revalidateTag } from "next/cache";
import { NextRequest, NextResponse } from "next/server";

export async function POST(request: NextRequest) {
  const secret = request.headers.get("x-revalidate-secret");
  if (secret !== process.env.REVALIDATION_SECRET) {
    return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
  }

  const { tag, path } = await request.json();

  if (tag) {
    revalidateTag(tag); // Revalidate all fetches tagged with this
  } else if (path) {
    revalidatePath(path); // Revalidate specific path
  }

  return NextResponse.json({ revalidated: true, now: Date.now() });
}

// Tag your fetches
async function getPost(slug: string) {
  const res = await fetch(`${API}/posts/${slug}`, {
    next: { tags: [`post-${slug}`, "posts"] },
  });
  return res.json();
}
```

### Partial Prerendering (Next 15)

```tsx
// next.config.ts
const config: NextConfig = {
  experimental: {
    ppr: "incremental", // Enable per-route
  },
};

// app/dashboard/page.tsx
export const experimental_ppr = true;

import { Suspense } from "react";

// The static shell is prerendered, dynamic parts stream in
export default function Dashboard() {
  return (
    <div>
      <h1>Dashboard</h1>           {/* Static — prerendered */}
      <StaticSidebar />             {/* Static — prerendered */}
      <Suspense fallback={<KPISkeleton />}>
        <DynamicKPIs />             {/* Dynamic — streams in */}
      </Suspense>
      <Suspense fallback={<ChartSkeleton />}>
        <RealtimeChart />           {/* Dynamic — streams in */}
      </Suspense>
    </div>
  );
}
```

---

## 5. Cache Headers

### Static assets (immutable)

```ts
// next.config.ts — Custom headers
const config: NextConfig = {
  async headers() {
    return [
      {
        // Static assets in _next/static are content-hashed
        source: "/_next/static/:path*",
        headers: [
          {
            key: "Cache-Control",
            value: "public, max-age=31536000, immutable",
          },
        ],
      },
      {
        // Fonts
        source: "/fonts/:path*",
        headers: [
          {
            key: "Cache-Control",
            value: "public, max-age=31536000, immutable",
          },
        ],
      },
      {
        // API responses with stale-while-revalidate
        source: "/api/public/:path*",
        headers: [
          {
            key: "Cache-Control",
            value: "public, s-maxage=60, stale-while-revalidate=300",
          },
        ],
      },
    ];
  },
};
```

### API Route caching

```ts
// app/api/products/route.ts
import { NextResponse } from "next/server";

export async function GET() {
  const products = await getProducts();

  return NextResponse.json(products, {
    headers: {
      "Cache-Control": "public, s-maxage=60, stale-while-revalidate=600",
    },
  });
}
```

### Common patterns

| Resource | Cache-Control | Reason |
|---|---|---|
| `_next/static/*` | `public, max-age=31536000, immutable` | Content-hashed filenames |
| HTML pages (SSG) | `public, s-maxage=3600, stale-while-revalidate=86400` | CDN caches, background revalidation |
| API (public) | `public, s-maxage=60, stale-while-revalidate=300` | Short TTL, SWR for availability |
| API (private) | `private, no-cache` | User-specific data |
| Images (uploaded) | `public, max-age=86400, stale-while-revalidate=604800` | Semi-static |

**DON'T:** Set long `max-age` on HTML pages — you lose the ability to invalidate.

---

## 6. Bundle Analysis

### Setup

```bash
bun add -d @next/bundle-analyzer
```

```ts
// next.config.ts
import type { NextConfig } from "next";
import bundleAnalyzer from "@next/bundle-analyzer";

const withBundleAnalyzer = bundleAnalyzer({
  enabled: process.env.ANALYZE === "true",
});

const config: NextConfig = {
  // ...
};

export default withBundleAnalyzer(config);
```

```bash
ANALYZE=true bun run build
```

### Tree-shaking tips

```tsx
// DON'T: Import entire library
import _ from "lodash";
const sorted = _.sortBy(items, "name");

// DO: Import specific function
import sortBy from "lodash/sortBy";
const sorted = sortBy(items, "name");

// BETTER: Use native JS or smaller alternatives
const sorted = [...items].sort((a, b) => a.name.localeCompare(b.name));
```

```tsx
// DON'T: Import all icons
import * as Icons from "lucide-react";

// DO: Import specific icons
import { ArrowRight, Check } from "lucide-react";
```

### Check for duplicate dependencies

```bash
# Find duplicates in node_modules
npx depcheck
bun pm ls | grep -E "moment|lodash|date-fns" # Check for overlapping utility libs

# In bundle analyzer, look for:
# - Multiple versions of the same lib (e.g., two React versions)
# - Libraries duplicated across chunks
# - Unexpectedly large modules
```

### Common bundle bloaters and replacements

| Heavy | Lightweight alternative |
|---|---|
| `moment` (300kB) | `date-fns` (tree-shakeable) or `dayjs` (2kB) |
| `lodash` (72kB) | Native JS or `lodash-es` (tree-shakeable) |
| `axios` (29kB) | `fetch` (built-in) |
| `uuid` (12kB) | `crypto.randomUUID()` (built-in) |
| `classnames` | `clsx` (< 1kB) or template literals |

---

## 7. Core Web Vitals Debugging

### Setup web-vitals reporting

```tsx
// app/components/web-vitals.tsx
"use client";

import { useReportWebVitals } from "next/web-vitals";

export function WebVitals() {
  useReportWebVitals((metric) => {
    // Send to your analytics
    const body = {
      name: metric.name,     // LCP, CLS, INP, FCP, TTFB
      value: metric.value,
      rating: metric.rating, // "good" | "needs-improvement" | "poor"
      id: metric.id,
    };

    // Beacon API — doesn't block navigation
    if (navigator.sendBeacon) {
      navigator.sendBeacon("/api/vitals", JSON.stringify(body));
    }
  });

  return null;
}

// app/layout.tsx
import { WebVitals } from "@/components/web-vitals";

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html>
      <body>
        <WebVitals />
        {children}
      </body>
    </html>
  );
}
```

### LCP (< 2.5s) — Largest Contentful Paint

Common culprits and fixes:

```tsx
// Problem: LCP image not prioritized
// Fix: Add priority to your hero/LCP image
<Image src="/hero.jpg" alt="" width={1200} height={630} priority />

// Problem: LCP image discovered late (loaded via CSS background or JS)
// Fix: Use <Image> or preload
// app/layout.tsx
<link rel="preload" as="image" href="/hero.jpg" />

// Problem: Render-blocking resources
// Fix: Check for synchronous scripts, large CSS
// Use next/font instead of external font CSS
```

### CLS (< 0.1) — Cumulative Layout Shift

```tsx
// DO: Always set explicit dimensions on images/videos
<Image src="/photo.jpg" alt="" width={800} height={600} />
<video width={640} height={360} />

// DO: Reserve space for dynamic content
<div className="min-h-[400px]"> {/* Skeleton matches final height */}
  <Suspense fallback={<Skeleton className="h-[400px]" />}>
    <DynamicContent />
  </Suspense>
</div>

// DON'T: Inject content above existing content after load
// DON'T: Use CSS that causes reflow (e.g., loading web fonts that change metrics)
// DON'T: Dynamically resize ads/embeds without reserved space
```

### INP (< 200ms) — Interaction to Next Paint

```tsx
"use client";

import { useTransition } from "react";

// DO: Use transitions for non-urgent updates
export function SearchFilter({ onFilter }: { onFilter: (q: string) => void }) {
  const [isPending, startTransition] = useTransition();

  return (
    <input
      onChange={(e) => {
        startTransition(() => {
          onFilter(e.target.value); // Doesn't block input
        });
      }}
      className={isPending ? "opacity-50" : ""}
    />
  );
}

// DO: Move heavy computation to Web Workers
// DO: Break up long tasks with scheduler.yield() or setTimeout
// DON'T: Run expensive JS synchronously in event handlers
```

### Lighthouse CI (CI/CD integration)

```yaml
# .github/workflows/lighthouse.yml
name: Lighthouse
on: [pull_request]
jobs:
  lighthouse:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: oven-sh/setup-bun@v2
      - run: bun install && bun run build
      - uses: treosh/lighthouse-ci-action@v12
        with:
          configPath: ./lighthouserc.json
          uploadArtifacts: true
```

```json
// lighthouserc.json
{
  "ci": {
    "assert": {
      "assertions": {
        "categories:performance": ["error", { "minScore": 0.9 }],
        "largest-contentful-paint": ["error", { "maxNumericValue": 2500 }],
        "cumulative-layout-shift": ["error", { "maxNumericValue": 0.1 }],
        "interactive": ["warn", { "maxNumericValue": 3800 }]
      }
    }
  }
}
```

---

## 8. Runtime Performance

### memo / useMemo / useCallback — when actually needed

```tsx
"use client";

import { memo, useMemo, useCallback } from "react";

// DO: Memo for expensive list item components that re-render often
const ExpensiveRow = memo(function ExpensiveRow({ item }: { item: Item }) {
  return <div>{/* complex rendering */}</div>;
});

// DO: useMemo for genuinely expensive computations
function Dashboard({ transactions }: { transactions: Transaction[] }) {
  const aggregated = useMemo(
    () => transactions.reduce((acc, t) => {
      // Complex aggregation over thousands of items
      return computeAggregates(acc, t);
    }, initialState),
    [transactions]
  );

  return <Chart data={aggregated} />;
}

// DO: useCallback when passing to memoized children or effect dependencies
function Parent() {
  const handleClick = useCallback((id: string) => {
    // handler logic
  }, []);

  return <MemoizedList onItemClick={handleClick} />;
}

// DON'T: Wrap everything in useMemo/useCallback by default
// DON'T: useMemo for simple operations (string concat, basic math)
// DON'T: memo components that always receive new object/array props
```

### Virtualization (TanStack Virtual)

```tsx
"use client";

import { useVirtualizer } from "@tanstack/react-virtual";
import { useRef } from "react";

export function VirtualList({ items }: { items: Item[] }) {
  const parentRef = useRef<HTMLDivElement>(null);

  const virtualizer = useVirtualizer({
    count: items.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 64, // Estimated row height in px
    overscan: 5,            // Render 5 extra items outside viewport
  });

  return (
    <div ref={parentRef} className="h-[600px] overflow-auto">
      <div
        style={{ height: `${virtualizer.getTotalSize()}px`, position: "relative" }}
      >
        {virtualizer.getVirtualItems().map((virtualRow) => (
          <div
            key={virtualRow.key}
            style={{
              position: "absolute",
              top: 0,
              transform: `translateY(${virtualRow.start}px)`,
              height: `${virtualRow.size}px`,
              width: "100%",
            }}
          >
            <Row item={items[virtualRow.index]} />
          </div>
        ))}
      </div>
    </div>
  );
}
```

### Debounce / Throttle

```tsx
"use client";

import { useDeferredValue, useState, useEffect, useRef, useCallback } from "react";

// Option 1: useDeferredValue (React 19 — preferred for rendering)
function Search() {
  const [query, setQuery] = useState("");
  const deferredQuery = useDeferredValue(query);

  return (
    <>
      <input value={query} onChange={(e) => setQuery(e.target.value)} />
      <Results query={deferredQuery} /> {/* Re-renders at lower priority */}
    </>
  );
}

// Option 2: Custom debounce hook (for API calls)
function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => setDebouncedValue(value), delay);
    return () => clearTimeout(timer);
  }, [value, delay]);

  return debouncedValue;
}

// Option 3: Debounced callback (for event handlers)
function useDebouncedCallback<T extends (...args: unknown[]) => void>(
  callback: T,
  delay: number
) {
  const timeoutRef = useRef<ReturnType<typeof setTimeout>>(null);

  return useCallback(
    (...args: Parameters<T>) => {
      if (timeoutRef.current) clearTimeout(timeoutRef.current);
      timeoutRef.current = setTimeout(() => callback(...args), delay);
    },
    [callback, delay]
  ) as T;
}
```

---

## 9. Edge Functions

### Decision matrix

| Factor | Edge Runtime | Node.js Runtime |
|---|---|---|
| Cold start | ~0ms | 250ms+ |
| Max execution | 30s (Vercel) | 300s (Vercel) |
| Bundle size limit | 4MB (Vercel) | 250MB |
| Node.js APIs | Subset only | Full |
| npm packages | Limited (no native deps) | All |
| Best for | Auth checks, redirects, A/B tests, geolocation | DB queries, file I/O, heavy computation |

### Edge middleware (runs before every matched request)

```ts
// middleware.ts (project root)
import { NextRequest, NextResponse } from "next/server";

export const config = {
  matcher: [
    // Skip static files and API routes you don't need to process
    "/((?!_next/static|_next/image|favicon.ico|api/public).*)",
  ],
};

export function middleware(request: NextRequest) {
  // Geo-based redirect (fast at the edge)
  const country = request.geo?.country;
  if (country === "AT" && !request.nextUrl.pathname.startsWith("/at")) {
    return NextResponse.redirect(new URL("/at" + request.nextUrl.pathname, request.url));
  }

  // Auth token check (don't do full validation here — just existence)
  const token = request.cookies.get("session")?.value;
  if (!token && request.nextUrl.pathname.startsWith("/dashboard")) {
    return NextResponse.redirect(new URL("/login", request.url));
  }

  // A/B test via cookie
  const response = NextResponse.next();
  if (!request.cookies.get("ab-variant")) {
    response.cookies.set("ab-variant", Math.random() > 0.5 ? "a" : "b", {
      maxAge: 60 * 60 * 24 * 30,
    });
  }

  return response;
}
```

### Edge API route

```ts
// app/api/og/route.ts
export const runtime = "edge"; // Opt into edge runtime

export async function GET(request: Request) {
  const { searchParams } = new URL(request.url);
  const title = searchParams.get("title") ?? "Default";

  // Light computation only — no heavy node modules
  return new Response(JSON.stringify({ title }), {
    headers: { "Content-Type": "application/json" },
  });
}
```

**DO:**
- Use edge for middleware (auth redirects, geolocation, headers, rewrites)
- Keep middleware fast (< 10ms) — it runs on every matched request
- Use `matcher` to skip unnecessary routes

**DON'T:**
- Use edge for Supabase queries with the `pg` driver (needs Node.js — use `@supabase/supabase-js` which works on edge)
- Import heavy npm packages in middleware
- Do database writes in middleware

---

## 10. Prefetching

### Link prefetch (automatic in viewport)

```tsx
import Link from "next/link";

// DO: Default prefetch (prefetches static shell in production)
<Link href="/about">About</Link>

// Disable for rarely-visited links to save bandwidth
<Link href="/terms" prefetch={false}>Terms</Link>

// Full page prefetch (including dynamic data) — Next 15
<Link href="/dashboard" prefetch={true}>Dashboard</Link>
```

### Programmatic prefetch

```tsx
"use client";

import { useRouter } from "next/navigation";

export function HoverCard({ href, children }: { href: string; children: React.ReactNode }) {
  const router = useRouter();

  return (
    <div
      onMouseEnter={() => router.prefetch(href)} // Prefetch on hover intent
      onClick={() => router.push(href)}
      role="link"
      tabIndex={0}
    >
      {children}
    </div>
  );
}
```

### Data prefetching in RSC

```tsx
// app/dashboard/page.tsx
import { Suspense } from "react";

// Parallel data fetching — both start immediately
async function DashboardPage() {
  // DO: Kick off all fetches in parallel at the page level
  const kpiPromise = getKPIs();
  const chartPromise = getChartData();
  const recentPromise = getRecentActivity();

  return (
    <div>
      <Suspense fallback={<KPISkeleton />}>
        <KPIs dataPromise={kpiPromise} />
      </Suspense>
      <Suspense fallback={<ChartSkeleton />}>
        <Chart dataPromise={chartPromise} />
      </Suspense>
      <Suspense fallback={<ActivitySkeleton />}>
        <Activity dataPromise={recentPromise} />
      </Suspense>
    </div>
  );
}

// Child component awaits the promise
async function KPIs({ dataPromise }: { dataPromise: Promise<KPIData> }) {
  const data = await dataPromise; // Suspends until resolved
  return <KPIGrid data={data} />;
}
```

### Preloading patterns with `cache()`

```tsx
// lib/queries.ts
import { cache } from "react";

// DO: Deduplicate requests across components in the same render
export const getUser = cache(async (id: string) => {
  const res = await fetch(`/api/users/${id}`);
  return res.json();
});

// Both components call getUser("123") but only one fetch executes
// app/dashboard/page.tsx
async function Dashboard() {
  return (
    <>
      <Header />      {/* calls getUser("123") */}
      <Sidebar />     {/* calls getUser("123") — deduped */}
    </>
  );
}
```

**DO:**
- Let Next.js handle `<Link>` prefetching automatically — it's smart about viewport detection
- Use `router.prefetch()` on hover/focus for high-intent navigation
- Start data fetches as early as possible in RSCs (don't waterfall)
- Use `cache()` to deduplicate identical fetches in the same render tree

**DON'T:**
- Prefetch everything — it wastes bandwidth on mobile
- Create fetch waterfalls (parent fetches, renders child, child fetches)
- Forget `<Suspense>` boundaries — without them, the entire page waits for the slowest fetch
