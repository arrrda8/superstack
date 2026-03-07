# Caching Strategies Reference

Production-ready caching patterns for Next.js full-stack applications.

---

## 1. Next.js Cache Layers

Next.js has four cache layers that interact in a specific order:

### Request Memoization (per-request, server only)

Deduplicates identical `fetch` calls within a single render pass.

```typescript
// Both components call the same fetch — only ONE request is made
async function getUser(id: string) {
  // Automatically memoized within the same request
  const res = await fetch(`https://api.example.com/users/${id}`);
  return res.json();
}

// Component A
const user = await getUser("123"); // fetch fires

// Component B (same render)
const user = await getUser("123"); // returns memoized result
```

- Scope: single server request (RSC render)
- Applies to: `fetch` with same URL + options, or `React.cache()` for non-fetch
- Lifetime: dies after response is sent

```typescript
import { cache } from "react";

// For non-fetch functions, use React.cache()
export const getUser = cache(async (id: string) => {
  const { data } = await supabase.from("users").select("*").eq("id", id).single();
  return data;
});
```

### Data Cache (persistent, server only)

Caches fetch results across requests and deployments.

```typescript
// Cached indefinitely (default in App Router)
const res = await fetch("https://api.example.com/data");

// Revalidate every 60 seconds (ISR for data)
const res = await fetch("https://api.example.com/data", {
  next: { revalidate: 60 },
});

// Opt out of caching
const res = await fetch("https://api.example.com/data", {
  cache: "no-store",
});

// Tag-based cache control
const res = await fetch("https://api.example.com/products", {
  next: { tags: ["products"] },
});
```

### Full Route Cache (persistent, server only)

Caches the rendered HTML + RSC payload for static routes at build time.

```typescript
// Statically cached at build time (no dynamic functions used)
export default async function ProductsPage() {
  const products = await getProducts(); // cached fetch
  return <ProductList products={products} />;
}

// Opting out: use dynamic functions
export const dynamic = "force-dynamic";

// Or use segment config
export const revalidate = 3600; // revalidate every hour
```

### Router Cache (client-side, in-memory)

Caches RSC payloads on the client during navigation.

```typescript
// Router Cache durations (Next.js 15+):
// - Dynamic pages: 0 seconds (not cached)
// - Static pages: 5 minutes
// - Prefetched (Link): 5 minutes (static), 0 (dynamic)

// Force refresh of Router Cache
import { useRouter } from "next/navigation";

const router = useRouter();
router.refresh(); // invalidates Router Cache for current route
```

### How layers interact

```
Client Request
  -> Router Cache (client) — hit? return cached RSC payload
  -> Full Route Cache (server) — hit? return cached HTML + RSC payload
  -> Render RSC
    -> Data Cache — hit? return cached fetch data
    -> Request Memoization — dedupe same fetches within render
    -> Origin fetch
```

---

## 2. ISR / SSG / SSR / PPR Decision Tree

```
Is the page content the same for all users?
|
+-- YES: Is the data completely static (never changes)?
|   |
|   +-- YES --> SSG (Static Site Generation)
|   |           export const dynamic = "force-static"
|   |
|   +-- NO: How often does data change?
|       |
|       +-- Predictable interval --> ISR (Incremental Static Regeneration)
|       |                            export const revalidate = 3600
|       |
|       +-- On specific events --> ISR + On-demand Revalidation
|                                  revalidatePath() / revalidateTag()
|
+-- NO (personalized): Is part of the page static?
    |
    +-- YES --> PPR (Partial Prerendering) [Next.js 15+]
    |           Static shell + dynamic Suspense boundaries
    |
    +-- NO: Does SEO matter?
        |
        +-- YES --> SSR (Server-Side Rendering)
        |           export const dynamic = "force-dynamic"
        |
        +-- NO --> Client-side fetching (TanStack Query / SWR)
```

### SSG — Static Site Generation

```typescript
// app/blog/[slug]/page.tsx
export async function generateStaticParams() {
  const posts = await getPosts();
  return posts.map((post) => ({ slug: post.slug }));
}

export default async function BlogPost({ params }: { params: { slug: string } }) {
  const post = await getPost(params.slug);
  return <Article post={post} />;
}
```

### ISR — Incremental Static Regeneration

```typescript
// Time-based revalidation
export const revalidate = 3600; // rebuild every hour

export default async function PricingPage() {
  const prices = await getPrices();
  return <PricingTable prices={prices} />;
}
```

### PPR — Partial Prerendering (Next.js 15+)

```typescript
// next.config.ts
const config: NextConfig = {
  experimental: { ppr: true },
};

// app/dashboard/page.tsx
import { Suspense } from "react";

export default function DashboardPage() {
  return (
    <div>
      {/* Static shell — prerendered at build */}
      <Header />
      <Sidebar />

      {/* Dynamic — streamed on request */}
      <Suspense fallback={<DashboardSkeleton />}>
        <DashboardContent />
      </Suspense>
    </div>
  );
}
```

---

## 3. revalidatePath / revalidateTag

### On-demand revalidation with revalidatePath

```typescript
// app/actions/product.ts
"use server";

import { revalidatePath } from "next/cache";

export async function updateProduct(id: string, data: ProductData) {
  await db.products.update({ where: { id }, data });

  // Revalidate specific page
  revalidatePath(`/products/${id}`);

  // Revalidate listing page
  revalidatePath("/products");

  // Revalidate layout (and all child pages)
  revalidatePath("/products", "layout");

  // Revalidate everything
  revalidatePath("/", "layout");
}
```

### Tag-based cache invalidation with revalidateTag

```typescript
// Fetching with tags
async function getProducts() {
  const res = await fetch("https://api.example.com/products", {
    next: { tags: ["products"] },
  });
  return res.json();
}

async function getProduct(id: string) {
  const res = await fetch(`https://api.example.com/products/${id}`, {
    next: { tags: ["products", `product-${id}`] },
  });
  return res.json();
}

// Invalidation in Server Action
"use server";

import { revalidateTag } from "next/cache";

export async function updateProduct(id: string, data: ProductData) {
  await db.products.update({ where: { id }, data });

  // Invalidate only this product's cache
  revalidateTag(`product-${id}`);
}

export async function bulkUpdateProducts() {
  await db.products.updateMany(/* ... */);

  // Invalidate ALL product caches
  revalidateTag("products");
}
```

### Webhook-triggered revalidation

```typescript
// app/api/revalidate/route.ts
import { revalidateTag } from "next/cache";
import { NextRequest, NextResponse } from "next/server";

export async function POST(req: NextRequest) {
  const secret = req.headers.get("x-revalidation-secret");
  if (secret !== process.env.REVALIDATION_SECRET) {
    return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
  }

  const { tag } = await req.json();
  revalidateTag(tag);

  return NextResponse.json({ revalidated: true, tag });
}
```

---

## 4. TanStack Query Cache

### Basic configuration

```typescript
// lib/query-client.ts
import { QueryClient } from "@tanstack/react-query";

export function makeQueryClient() {
  return new QueryClient({
    defaultOptions: {
      queries: {
        staleTime: 60 * 1000,       // Data considered fresh for 60s
        gcTime: 5 * 60 * 1000,      // Garbage collect after 5 min unused
        refetchOnWindowFocus: false, // Disable for dashboards
        retry: 2,
      },
    },
  });
}
```

### Key timing concepts

```typescript
// staleTime: how long data is considered "fresh"
// - While fresh: no refetch on mount/focus/reconnect
// - After stale: refetch in background, show stale data immediately

// gcTime (formerly cacheTime): how long INACTIVE data stays in memory
// - Starts counting when last observer unmounts
// - After gcTime: data is garbage collected

useQuery({
  queryKey: ["dashboard", "stats"],
  queryFn: fetchDashboardStats,
  staleTime: 30 * 1000,       // fresh for 30s (don't refetch)
  gcTime: 10 * 60 * 1000,     // keep in memory 10 min after unmount
});
```

### placeholderData for instant UX

```typescript
import { keepPreviousData } from "@tanstack/react-query";

// Keep previous page data visible while loading next page
useQuery({
  queryKey: ["products", { page, filters }],
  queryFn: () => fetchProducts({ page, filters }),
  placeholderData: keepPreviousData, // show old data during load
});

// Or provide static placeholder
useQuery({
  queryKey: ["user", userId],
  queryFn: () => fetchUser(userId),
  placeholderData: { name: "Loading...", email: "" },
});
```

### Prefetching in RSC (server-side hydration)

```typescript
// app/products/page.tsx
import { dehydrate, HydrationBoundary, QueryClient } from "@tanstack/react-query";

export default async function ProductsPage() {
  const queryClient = new QueryClient();

  await queryClient.prefetchQuery({
    queryKey: ["products"],
    queryFn: fetchProducts,
  });

  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      <ProductList />
    </HydrationBoundary>
  );
}

// components/ProductList.tsx (client component)
"use client";

export function ProductList() {
  // This will use the prefetched data — no loading state on first render
  const { data } = useQuery({
    queryKey: ["products"],
    queryFn: fetchProducts,
  });
  return <div>{/* render products */}</div>;
}
```

### Cache invalidation

```typescript
import { useQueryClient, useMutation } from "@tanstack/react-query";

function useUpdateProduct() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (data: ProductUpdate) => updateProduct(data),
    onSuccess: (result, variables) => {
      // Invalidate and refetch
      queryClient.invalidateQueries({ queryKey: ["products"] });

      // Or update cache directly (optimistic)
      queryClient.setQueryData(["product", variables.id], result);
    },
  });
}

// Optimistic update pattern
useMutation({
  mutationFn: updateTodo,
  onMutate: async (newTodo) => {
    await queryClient.cancelQueries({ queryKey: ["todos"] });
    const previous = queryClient.getQueryData(["todos"]);
    queryClient.setQueryData(["todos"], (old: Todo[]) =>
      old.map((t) => (t.id === newTodo.id ? { ...t, ...newTodo } : t))
    );
    return { previous };
  },
  onError: (_err, _newTodo, context) => {
    queryClient.setQueryData(["todos"], context?.previous);
  },
  onSettled: () => {
    queryClient.invalidateQueries({ queryKey: ["todos"] });
  },
});
```

---

## 5. Redis / Upstash

### Upstash Redis setup

```typescript
// lib/redis.ts
import { Redis } from "@upstash/redis";

export const redis = new Redis({
  url: process.env.UPSTASH_REDIS_REST_URL!,
  token: process.env.UPSTASH_REDIS_REST_TOKEN!,
});
```

### KV store for computed data

```typescript
// Cache expensive computation
async function getAnalytics(orgId: string): Promise<Analytics> {
  const cacheKey = `analytics:${orgId}`;

  // Try cache first
  const cached = await redis.get<Analytics>(cacheKey);
  if (cached) return cached;

  // Compute fresh
  const analytics = await computeExpensiveAnalytics(orgId);

  // Cache with TTL
  await redis.set(cacheKey, analytics, { ex: 300 }); // 5 min TTL

  return analytics;
}

// Invalidate on data change
async function onDataUpdate(orgId: string) {
  await redis.del(`analytics:${orgId}`);
}
```

### Session cache

```typescript
interface SessionData {
  userId: string;
  orgId: string;
  role: string;
  permissions: string[];
}

async function getSession(sessionId: string): Promise<SessionData | null> {
  return redis.get<SessionData>(`session:${sessionId}`);
}

async function setSession(sessionId: string, data: SessionData) {
  await redis.set(`session:${sessionId}`, data, { ex: 86400 }); // 24h
}

async function destroySession(sessionId: string) {
  await redis.del(`session:${sessionId}`);
}
```

### Rate limiting

```typescript
// lib/rate-limit.ts
import { Ratelimit } from "@upstash/ratelimit";
import { redis } from "./redis";

// 10 requests per 10 seconds sliding window
export const ratelimit = new Ratelimit({
  redis,
  limiter: Ratelimit.slidingWindow(10, "10 s"),
  analytics: true,
});

// Usage in Route Handler
export async function POST(req: NextRequest) {
  const ip = req.headers.get("x-forwarded-for") ?? "127.0.0.1";
  const { success, limit, remaining, reset } = await ratelimit.limit(ip);

  if (!success) {
    return NextResponse.json(
      { error: "Rate limit exceeded" },
      {
        status: 429,
        headers: {
          "X-RateLimit-Limit": limit.toString(),
          "X-RateLimit-Remaining": remaining.toString(),
          "X-RateLimit-Reset": reset.toString(),
        },
      }
    );
  }

  // Process request...
}
```

### TTL patterns

```typescript
// Short TTL: frequently changing data
await redis.set("live-score:match-123", score, { ex: 10 }); // 10s

// Medium TTL: computed aggregations
await redis.set("dashboard:org-456", stats, { ex: 300 }); // 5 min

// Long TTL: rarely changing reference data
await redis.set("config:feature-flags", flags, { ex: 3600 }); // 1 hour

// Conditional TTL with NX (only set if not exists)
await redis.set("lock:job-789", "processing", { ex: 60, nx: true });
```

---

## 6. CDN Caching

### Vercel Edge Cache with Cache-Control headers

```typescript
// app/api/products/route.ts
export async function GET() {
  const products = await getProducts();

  return NextResponse.json(products, {
    headers: {
      // Cache on CDN for 60s, serve stale for 300s while revalidating
      "Cache-Control": "public, s-maxage=60, stale-while-revalidate=300",
    },
  });
}
```

### Cache-Control header breakdown

```
Cache-Control: public, s-maxage=60, stale-while-revalidate=300

public           — CDN can cache this
s-maxage=60      — CDN cache TTL (overrides max-age for shared caches)
max-age=0        — Browser should not cache (or set a value)
stale-while-revalidate=300 — Serve stale for up to 300s while fetching fresh
no-store         — Never cache anywhere
no-cache         — Cache but revalidate every time
private          — Only browser can cache (user-specific data)
immutable        — Content never changes (use with hashed filenames)
```

### Static assets on Vercel

```typescript
// next.config.ts — static assets get immutable caching automatically
// /_next/static/* -> Cache-Control: public, max-age=31536000, immutable

// Custom headers for specific paths
const config: NextConfig = {
  async headers() {
    return [
      {
        source: "/api/public/:path*",
        headers: [
          {
            key: "Cache-Control",
            value: "public, s-maxage=3600, stale-while-revalidate=86400",
          },
        ],
      },
    ];
  },
};
```

### Surrogate keys (tag-based CDN invalidation on Vercel)

```typescript
// Vercel supports tag-based purging via revalidateTag
// When you call revalidateTag("products"), Vercel purges the CDN cache
// for all responses tagged with "products"
```

---

## 7. SWR Pattern (Stale-While-Revalidate)

### The concept

```
1. Request comes in
2. Cache has data? Return it immediately (even if stale)
3. Fetch fresh data in background
4. Update cache with fresh data
5. Next request gets fresh data

User experience: instant response, eventual consistency
```

### Implementation with fetch

```typescript
// Using Cache-Control header
const res = await fetch("/api/data", {
  headers: {
    "Cache-Control": "s-maxage=60, stale-while-revalidate=300",
    // For 60s: serve from cache
    // From 60-360s: serve stale, revalidate in background
    // After 360s: cache miss, fetch fresh
  },
});
```

### Implementation with TanStack Query

```typescript
// TanStack Query implements SWR by default
const { data } = useQuery({
  queryKey: ["products"],
  queryFn: fetchProducts,
  staleTime: 60_000, // fresh for 60s — no background refetch
  // After 60s: data is "stale"
  // On next access: return stale data immediately + refetch in background
  // When refetch completes: update UI seamlessly
});
```

### SWR with Upstash Redis

```typescript
async function getWithSWR<T>(
  key: string,
  fetcher: () => Promise<T>,
  ttl: number = 300,
  staleWindow: number = 600
): Promise<T> {
  const cached = await redis.get<{ data: T; timestamp: number }>(key);

  if (cached) {
    const age = Date.now() - cached.timestamp;

    if (age < ttl * 1000) {
      // Fresh — return immediately
      return cached.data;
    }

    if (age < (ttl + staleWindow) * 1000) {
      // Stale — return cached, revalidate in background
      // Fire and forget — don't await
      fetcher().then((fresh) =>
        redis.set(key, { data: fresh, timestamp: Date.now() }, { ex: ttl + staleWindow })
      );
      return cached.data;
    }
  }

  // Cache miss or expired — fetch and cache
  const fresh = await fetcher();
  await redis.set(key, { data: fresh, timestamp: Date.now() }, { ex: ttl + staleWindow });
  return fresh;
}
```

---

## 8. Cache Invalidation Strategies

### Event-driven invalidation

```typescript
// Server Action triggers cache invalidation
"use server";

import { revalidateTag, revalidatePath } from "next/cache";

export async function createOrder(data: OrderInput) {
  const order = await db.orders.create({ data });

  // Invalidate related caches
  revalidateTag("orders");
  revalidateTag(`user-orders-${data.userId}`);
  revalidatePath("/admin/orders");

  // Invalidate Redis cache
  await redis.del(`analytics:${data.orgId}`);
  await redis.del(`user-stats:${data.userId}`);

  // Invalidate TanStack Query (client-side, via response)
  return { success: true, order, invalidate: ["orders", "stats"] };
}
```

### Webhook-driven invalidation

```typescript
// app/api/webhooks/cms/route.ts
export async function POST(req: NextRequest) {
  const payload = await req.json();

  switch (payload.event) {
    case "entry.update":
      revalidateTag(`content-${payload.entry.slug}`);
      break;
    case "entry.publish":
      revalidateTag("content");
      revalidatePath("/blog");
      break;
    case "entry.delete":
      revalidateTag("content");
      revalidatePath("/blog");
      break;
  }

  return NextResponse.json({ ok: true });
}
```

### Time-based + event-based hybrid

```typescript
// Use short TTL as safety net + event-based invalidation for freshness
async function getCatalog(orgId: string) {
  const res = await fetch(`${API_URL}/catalog/${orgId}`, {
    next: {
      revalidate: 300,                    // Safety net: 5 min max staleness
      tags: [`catalog-${orgId}`],         // Event-based: invalidate on change
    },
  });
  return res.json();
}

// On catalog update: immediate invalidation
export async function updateCatalog(orgId: string, data: CatalogData) {
  await db.catalogs.update({ where: { orgId }, data });
  revalidateTag(`catalog-${orgId}`); // Immediate freshness
}
```

---

## 9. Browser Caching

### Service Worker cache (offline support)

```typescript
// public/sw.js
const CACHE_NAME = "app-v1";
const STATIC_ASSETS = ["/", "/offline", "/manifest.json"];

self.addEventListener("install", (event) => {
  event.waitUntil(
    caches.open(CACHE_NAME).then((cache) => cache.addAll(STATIC_ASSETS))
  );
});

self.addEventListener("fetch", (event) => {
  // Network first, fall back to cache
  event.respondWith(
    fetch(event.request)
      .then((response) => {
        const clone = response.clone();
        caches.open(CACHE_NAME).then((cache) => cache.put(event.request, clone));
        return response;
      })
      .catch(() => caches.match(event.request))
  );
});
```

### localStorage / sessionStorage

```typescript
// Utility for typed storage with TTL
function setWithExpiry<T>(key: string, value: T, ttlMs: number) {
  const item = {
    value,
    expiry: Date.now() + ttlMs,
  };
  localStorage.setItem(key, JSON.stringify(item));
}

function getWithExpiry<T>(key: string): T | null {
  const raw = localStorage.getItem(key);
  if (!raw) return null;

  const item = JSON.parse(raw) as { value: T; expiry: number };
  if (Date.now() > item.expiry) {
    localStorage.removeItem(key);
    return null;
  }
  return item.value;
}

// Usage
setWithExpiry("user-preferences", { theme: "dark", locale: "de" }, 7 * 24 * 60 * 60 * 1000);
const prefs = getWithExpiry<UserPreferences>("user-preferences");
```

### IndexedDB for large datasets

```typescript
// Using idb library for cleaner API
import { openDB } from "idb";

const dbPromise = openDB("app-store", 1, {
  upgrade(db) {
    db.createObjectStore("products", { keyPath: "id" });
    db.createObjectStore("sync-queue", { keyPath: "id", autoIncrement: true });
  },
});

// Cache large dataset locally
async function cacheProducts(products: Product[]) {
  const db = await dbPromise;
  const tx = db.transaction("products", "readwrite");
  await Promise.all(products.map((p) => tx.store.put(p)));
  await tx.done;
}

async function getCachedProducts(): Promise<Product[]> {
  const db = await dbPromise;
  return db.getAll("products");
}
```

---

## 10. Anti-Patterns

### Over-caching

```typescript
// BAD: caching user-specific data on CDN
return NextResponse.json(userData, {
  headers: { "Cache-Control": "public, s-maxage=3600" }, // Leaks user data!
});

// GOOD: user-specific data is private
return NextResponse.json(userData, {
  headers: { "Cache-Control": "private, max-age=60" },
});
```

### Cache stampede (thundering herd)

```typescript
// BAD: cache expires, 1000 concurrent users all hit the origin
async function getData() {
  const cached = await redis.get("popular-data");
  if (!cached) {
    const data = await expensiveQuery(); // 1000 concurrent calls!
    await redis.set("popular-data", data, { ex: 60 });
    return data;
  }
  return cached;
}

// GOOD: use a lock to prevent stampede
async function getDataSafe() {
  const cached = await redis.get("popular-data");
  if (cached) return cached;

  // Try to acquire lock
  const locked = await redis.set("lock:popular-data", "1", { ex: 10, nx: true });
  if (!locked) {
    // Another process is refreshing — wait and retry
    await new Promise((r) => setTimeout(r, 100));
    return redis.get("popular-data");
  }

  try {
    const data = await expensiveQuery();
    await redis.set("popular-data", data, { ex: 60 });
    return data;
  } finally {
    await redis.del("lock:popular-data");
  }
}
```

### Stale data bugs

```typescript
// BAD: caching without considering write-through
async function updateUserName(userId: string, name: string) {
  await db.users.update({ where: { id: userId }, data: { name } });
  // Forgot to invalidate cache! User sees old name.
}

// GOOD: always invalidate related caches after writes
async function updateUserName(userId: string, name: string) {
  await db.users.update({ where: { id: userId }, data: { name } });

  // Invalidate all layers
  revalidateTag(`user-${userId}`);
  await redis.del(`user:${userId}`);
  // Client-side: queryClient.invalidateQueries({ queryKey: ["user", userId] })
}
```

### Forgetting to invalidate

```typescript
// Create a cache invalidation registry to avoid missing invalidations
const CACHE_DEPENDENCIES: Record<string, string[]> = {
  "product:update": ["products", "catalog", "search-index"],
  "order:create": ["orders", "analytics", "inventory"],
  "user:update": ["user-profile", "team-members"],
};

async function invalidateForEvent(event: string, context: Record<string, string>) {
  const tags = CACHE_DEPENDENCIES[event] ?? [];
  for (const tag of tags) {
    // Replace placeholders: "user-profile" -> "user-profile:user-123"
    const resolvedTag = context.id ? `${tag}:${context.id}` : tag;
    revalidateTag(resolvedTag);
  }
}

// Usage
await invalidateForEvent("product:update", { id: "prod-123" });
```

### Caching errors

```typescript
// BAD: caching error responses
const res = await fetch(url, { next: { revalidate: 3600 } });
const data = await res.json(); // If API returned 500, you cached the error for 1 hour

// GOOD: only cache successful responses
async function fetchWithCacheGuard(url: string) {
  const res = await fetch(url, { cache: "no-store" }); // Don't auto-cache
  if (!res.ok) throw new Error(`Fetch failed: ${res.status}`);

  const data = await res.json();
  // Manually cache only good data
  await redis.set(`cache:${url}`, data, { ex: 3600 });
  return data;
}
```
