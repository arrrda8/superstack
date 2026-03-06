# DevOps Reference — Superstack

## 1. Sentry Error Monitoring Setup

### Installation

```bash
bunx @sentry/wizard@latest -i nextjs
```

This auto-creates config files. Verify and adjust:

### Client Config (sentry.client.config.ts)

```typescript
import * as Sentry from "@sentry/nextjs";

Sentry.init({
  dsn: process.env.NEXT_PUBLIC_SENTRY_DSN,
  environment: process.env.NEXT_PUBLIC_VERCEL_ENV || "development",
  tracesSampleRate: process.env.NODE_ENV === "production" ? 0.2 : 1.0,
  replaysSessionSampleRate: 0.1,
  replaysOnErrorSampleRate: 1.0,
  integrations: [
    Sentry.replayIntegration({
      maskAllText: true,
      blockAllMedia: true,
    }),
  ],
  // Filter noise
  ignoreErrors: [
    "ResizeObserver loop",
    "Non-Error promise rejection",
    /Loading chunk \d+ failed/,
  ],
});
```

### Server Config (sentry.server.config.ts)

```typescript
import * as Sentry from "@sentry/nextjs";

Sentry.init({
  dsn: process.env.SENTRY_DSN,
  environment: process.env.VERCEL_ENV || "development",
  tracesSampleRate: 0.2,
});
```

### Global Error Boundary (app/global-error.tsx)

```typescript
"use client";

import * as Sentry from "@sentry/nextjs";
import { useEffect } from "react";

export default function GlobalError({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  useEffect(() => {
    Sentry.captureException(error);
  }, [error]);

  return (
    <html>
      <body>
        <h2>Something went wrong</h2>
        <button onClick={() => reset()}>Try again</button>
      </body>
    </html>
  );
}
```

### CRITICAL: Tunnel Route to Bypass Adblockers

Adblockers block requests to `sentry.io`. Use a tunnel route.

```typescript
// app/api/tunnel/route.ts
import { NextRequest, NextResponse } from "next/server";

const SENTRY_HOST = "o123456.ingest.sentry.io"; // Replace with your org
const SENTRY_PROJECT_IDS = ["your-project-id"];

export async function POST(req: NextRequest) {
  const envelope = await req.text();
  const header = envelope.split("\n")[0];

  const dsn = JSON.parse(header).dsn as string;
  const { host, pathname } = new URL(dsn);

  const projectId = pathname.replace("/", "");

  if (host !== SENTRY_HOST || !SENTRY_PROJECT_IDS.includes(projectId)) {
    return NextResponse.json({ error: "Invalid DSN" }, { status: 400 });
  }

  const sentryUrl = `https://${SENTRY_HOST}/api/${projectId}/envelope/`;

  const response = await fetch(sentryUrl, {
    method: "POST",
    body: envelope,
    headers: { "Content-Type": "application/x-sentry-envelope" },
  });

  return NextResponse.json({}, { status: response.status });
}
```

In `sentry.client.config.ts` add: `tunnel: "/api/tunnel"`

---

## 2. Structured Logging with Pino

Pino > Winston for Next.js: faster serialization, JSON-native, lower overhead.

### Installation

```bash
bun add pino pino-pretty
```

### Logger Setup (lib/logger.ts)

```typescript
import pino from "pino";

const isDev = process.env.NODE_ENV === "development";

export const logger = pino({
  level: process.env.LOG_LEVEL || (isDev ? "debug" : "info"),
  ...(isDev
    ? {
        transport: {
          target: "pino-pretty",
          options: { colorize: true, translateTime: "HH:MM:ss" },
        },
      }
    : {}),
  // Production: plain JSON for log aggregators (Datadog, Loki, etc.)
  formatters: {
    level: (label) => ({ level: label }),
  },
  redact: {
    paths: ["password", "token", "authorization", "cookie", "creditCard", "ssn"],
    censor: "[REDACTED]",
  },
});

// Child logger with request context
export function createRequestLogger(requestId: string, userId?: string) {
  return logger.child({ requestId, userId });
}
```

### Usage in API Routes

```typescript
import { createRequestLogger } from "@/lib/logger";
import { randomUUID } from "crypto";

export async function POST(req: Request) {
  const requestId = randomUUID();
  const log = createRequestLogger(requestId);

  log.info({ path: "/api/orders" }, "Request received");

  try {
    const result = await processOrder();
    log.info({ orderId: result.id }, "Order created");
    return Response.json(result);
  } catch (error) {
    log.error({ err: error }, "Order creation failed");
    return Response.json({ error: "Internal error" }, { status: 500 });
  }
}
```

### NEVER Log

- Passwords or password hashes
- API tokens or secrets
- PII (email, phone, address) in production
- Credit card numbers
- Session tokens or JWTs

---

## 3. Health Check Endpoint

### Route Handler (app/api/health/route.ts)

```typescript
import { NextResponse } from "next/server";
import { createClient } from "@supabase/supabase-js";

const startTime = Date.now();

async function checkDatabase(): Promise<{ ok: boolean; latencyMs: number }> {
  const start = performance.now();
  try {
    const supabase = createClient(
      process.env.SUPABASE_URL!,
      process.env.SUPABASE_SERVICE_ROLE_KEY!
    );
    await supabase.from("_health").select("1").limit(1).single();
    return { ok: true, latencyMs: Math.round(performance.now() - start) };
  } catch {
    return { ok: false, latencyMs: Math.round(performance.now() - start) };
  }
}

function checkMemory(): { ok: boolean; usageMB: number; limitMB: number } {
  const usage = process.memoryUsage();
  const usageMB = Math.round(usage.heapUsed / 1024 / 1024);
  const limitMB = 512; // Adjust per deployment
  return { ok: usageMB < limitMB * 0.9, usageMB, limitMB };
}

export async function GET() {
  const db = await checkDatabase();
  const memory = checkMemory();

  const allHealthy = db.ok && memory.ok;

  return NextResponse.json(
    {
      status: allHealthy ? "healthy" : "degraded",
      uptime: Math.round((Date.now() - startTime) / 1000),
      version: process.env.NEXT_PUBLIC_APP_VERSION || "unknown",
      checks: { database: db, memory },
    },
    { status: allHealthy ? 200 : 503 }
  );
}
```

### Docker HEALTHCHECK

```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD curl -f http://localhost:3000/api/health || exit 1
```

### External Monitoring

- **UptimeRobot** (free tier: 50 monitors, 5-min interval)
- **BetterUptime** (better UI, incident pages, 3-min interval)
- Alert channels: Slack webhook, email, SMS for critical

---

## 4. Docker Multi-Stage Build

### Dockerfile

```dockerfile
# Stage 1: Dependencies
FROM node:20-alpine AS deps
RUN apk add --no-cache libc6-compat
WORKDIR /app
COPY package.json bun.lockb ./
RUN npm install -g bun && bun install --frozen-lockfile

# Stage 2: Build
FROM node:20-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
ENV NEXT_TELEMETRY_DISABLED=1
RUN npm install -g bun && bun run build

# Stage 3: Runner (production)
FROM node:20-alpine AS runner
WORKDIR /app

ENV NODE_ENV=production
ENV NEXT_TELEMETRY_DISABLED=1

RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

USER nextjs
EXPOSE 3000
ENV PORT=3000
ENV HOSTNAME="0.0.0.0"

CMD ["node", "server.js"]
```

### next.config.ts (required for standalone)

```typescript
const nextConfig = {
  output: "standalone",
};
export default nextConfig;
```

### docker-compose.yml

```yaml
version: "3.9"
services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "3000:3000"
    env_file: .env.production
    depends_on:
      redis:
        condition: service_healthy
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5

volumes:
  redis_data:
```

---

## 5. Redis Caching Patterns

### Cache-Aside Pattern

```typescript
import { Redis } from "@upstash/redis";

const redis = Redis.fromEnv(); // Uses UPSTASH_REDIS_REST_URL + TOKEN

export async function getCached<T>(
  key: string,
  fetcher: () => Promise<T>,
  ttlSeconds: number = 300
): Promise<T> {
  const cached = await redis.get<T>(key);
  if (cached !== null) return cached;

  const fresh = await fetcher();
  await redis.set(key, fresh, { ex: ttlSeconds });
  return fresh;
}

// Usage
const product = await getCached(
  `product:${id}`,
  () => db.query.products.findFirst({ where: eq(products.id, id) }),
  600 // 10 minutes
);
```

### Tag-Based Invalidation

```typescript
export async function invalidateByTag(tag: string) {
  const keys = await redis.smembers(`tag:${tag}`);
  if (keys.length > 0) {
    await redis.del(...keys);
    await redis.del(`tag:${tag}`);
  }
}

export async function setWithTags<T>(
  key: string,
  value: T,
  tags: string[],
  ttlSeconds: number
) {
  await redis.set(key, value, { ex: ttlSeconds });
  for (const tag of tags) {
    await redis.sadd(`tag:${tag}`, key);
  }
}
```

### TTL Strategies

| Content Type       | TTL        | Reason                          |
| ------------------ | ---------- | ------------------------------- |
| Static pages       | 1 hour     | Rarely changes                  |
| Product listings   | 5 minutes  | Price/stock may change          |
| User session data  | 30 minutes | Must stay fresh                 |
| API rate limit     | 1 minute   | Short window                    |
| Search results     | 15 minutes | Balance freshness vs. load      |

---

## 6. Database Migrations

### Drizzle (Recommended for Superstack)

```bash
bun add drizzle-orm drizzle-kit
```

```bash
# Generate migration from schema changes
bunx drizzle-kit generate

# Apply migrations (CI/CD and production)
bunx drizzle-kit migrate

# Push schema directly (development only — no migration files)
bunx drizzle-kit push
```

### Breaking Change Strategy (Multi-Step)

Never rename/drop columns directly. Use a multi-step approach:

1. **Step 1 (Deploy A):** Add new column, keep old column
2. **Step 2 (Deploy B):** Backfill data from old → new column
3. **Step 3 (Deploy C):** Update code to read from new column only
4. **Step 4 (Deploy D):** Drop old column

### CI/CD Integration

```yaml
# GitHub Actions step — run BEFORE deployment
- name: Run database migrations
  run: bunx drizzle-kit migrate
  env:
    DATABASE_URL: ${{ secrets.DATABASE_URL }}
```

---

## 7. Secret Management

### .env.example Template

```bash
# === App ===
NEXT_PUBLIC_APP_URL=http://localhost:3000
NEXT_PUBLIC_APP_VERSION=0.1.0

# === Database ===
DATABASE_URL=postgresql://user:password@localhost:5432/mydb
SUPABASE_URL=https://xxx.supabase.co
SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=

# === Auth ===
NEXTAUTH_SECRET=
NEXTAUTH_URL=http://localhost:3000

# === External Services ===
SENTRY_DSN=
STRIPE_SECRET_KEY=
STRIPE_WEBHOOK_SECRET=
RESEND_API_KEY=
UPSTASH_REDIS_REST_URL=
UPSTASH_REDIS_REST_TOKEN=
```

### Rules

- NEVER commit `.env` files — only `.env.example` with empty values
- Use Vercel Environment Variables for production (scoped per environment)
- Rotate secrets quarterly; immediately if compromised
- For teams: use **Infisical** (self-hosted, open source) or **Doppler** (managed)

### .gitignore entries

```
.env
.env.local
.env.production
.env*.local
```
