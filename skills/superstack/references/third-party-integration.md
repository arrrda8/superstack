# Third-Party Integration Patterns

## Table of Contents
- [1. API Client Wrapper](#1-api-client-wrapper)
- [2. Circuit Breaker](#2-circuit-breaker)
- [3. OAuth2 Refresh Flow](#3-oauth2-refresh-flow)
- [4. Retry with Exponential Backoff](#4-retry-with-exponential-backoff)
- [5. Webhook Integration](#5-webhook-integration)
- [6. API Key Management](#6-api-key-management)
- [7. Rate Limit Handling](#7-rate-limit-handling)
- [8. Data Sync Patterns](#8-data-sync-patterns)
- [9. Error Mapping](#9-error-mapping)
- [10. Common Integrations](#10-common-integrations)
  - [Stripe](#stripe)
  - [Resend (Email)](#resend-email)
  - [Supabase](#supabase)
  - [OpenAI / Anthropic](#openai--anthropic)
  - [Google APIs](#google-apis)

Reference for integrating external APIs, handling auth flows, managing rate limits, syncing data, and wiring up common services (Stripe, Resend, Supabase, OpenAI/Anthropic, Google APIs).

---

## 1. API Client Wrapper

Typed, reusable API client class with base URL config, auth headers, and request/response interceptors.

```tsx
// lib/api-client.ts
type RequestInterceptor = (config: RequestInit & { url: string }) => RequestInit & { url: string };
type ResponseInterceptor = (response: Response) => Response | Promise<Response>;

interface ApiClientConfig {
  baseUrl: string;
  headers?: Record<string, string>;
  requestInterceptors?: RequestInterceptor[];
  responseInterceptors?: ResponseInterceptor[];
}

class ApiClient {
  private baseUrl: string;
  private defaultHeaders: Record<string, string>;
  private requestInterceptors: RequestInterceptor[];
  private responseInterceptors: ResponseInterceptor[];

  constructor(config: ApiClientConfig) {
    this.baseUrl = config.baseUrl.replace(/\/$/, "");
    this.defaultHeaders = {
      "Content-Type": "application/json",
      ...config.headers,
    };
    this.requestInterceptors = config.requestInterceptors ?? [];
    this.responseInterceptors = config.responseInterceptors ?? [];
  }

  private async request<T>(
    method: string,
    path: string,
    body?: unknown,
    options?: RequestInit
  ): Promise<T> {
    let config: RequestInit & { url: string } = {
      url: `${this.baseUrl}${path}`,
      method,
      headers: { ...this.defaultHeaders, ...(options?.headers as Record<string, string>) },
      ...(body ? { body: JSON.stringify(body) } : {}),
      ...options,
    };

    // Apply request interceptors
    for (const interceptor of this.requestInterceptors) {
      config = interceptor(config);
    }

    const { url, ...fetchOptions } = config;
    let response = await fetch(url, fetchOptions);

    // Apply response interceptors
    for (const interceptor of this.responseInterceptors) {
      response = await interceptor(response);
    }

    if (!response.ok) {
      const errorBody = await response.text();
      throw new ApiError(response.status, errorBody, path);
    }

    return response.json() as Promise<T>;
  }

  async get<T>(path: string, options?: RequestInit) {
    return this.request<T>("GET", path, undefined, options);
  }
  async post<T>(path: string, body?: unknown, options?: RequestInit) {
    return this.request<T>("POST", path, body, options);
  }
  async put<T>(path: string, body?: unknown, options?: RequestInit) {
    return this.request<T>("PUT", path, body, options);
  }
  async patch<T>(path: string, body?: unknown, options?: RequestInit) {
    return this.request<T>("PATCH", path, body, options);
  }
  async delete<T>(path: string, options?: RequestInit) {
    return this.request<T>("DELETE", path, undefined, options);
  }
}

class ApiError extends Error {
  constructor(
    public status: number,
    public body: string,
    public path: string
  ) {
    super(`API Error ${status} at ${path}: ${body}`);
    this.name = "ApiError";
  }
}

// Usage
const stripeClient = new ApiClient({
  baseUrl: "https://api.stripe.com/v1",
  headers: {
    Authorization: `Bearer ${process.env.STRIPE_SECRET_KEY}`,
  },
  requestInterceptors: [
    (config) => {
      // Log outgoing requests
      console.log(`[API] ${config.method} ${config.url}`);
      return config;
    },
  ],
  responseInterceptors: [
    async (response) => {
      if (response.status === 429) {
        // Handle rate limit - see section 7
      }
      return response;
    },
  ],
});
```

---

## 2. Circuit Breaker

Prevent cascading failures when a third-party service is down. Track failures, open the circuit after a threshold, and periodically test with half-open state.

```tsx
// lib/circuit-breaker.ts
type CircuitState = "CLOSED" | "OPEN" | "HALF_OPEN";

interface CircuitBreakerOptions {
  failureThreshold: number;    // failures before opening
  resetTimeout: number;        // ms before trying half-open
  halfOpenMaxAttempts: number;  // successful calls to close again
  fallback?: <T>() => T;       // fallback response when open
}

class CircuitBreaker {
  private state: CircuitState = "CLOSED";
  private failureCount = 0;
  private successCount = 0;
  private lastFailureTime = 0;
  private options: CircuitBreakerOptions;

  constructor(options: CircuitBreakerOptions) {
    this.options = options;
  }

  async execute<T>(fn: () => Promise<T>): Promise<T> {
    if (this.state === "OPEN") {
      if (Date.now() - this.lastFailureTime >= this.options.resetTimeout) {
        this.state = "HALF_OPEN";
        this.successCount = 0;
      } else {
        if (this.options.fallback) {
          return this.options.fallback();
        }
        throw new CircuitOpenError("Circuit breaker is OPEN");
      }
    }

    try {
      const result = await fn();

      if (this.state === "HALF_OPEN") {
        this.successCount++;
        if (this.successCount >= this.options.halfOpenMaxAttempts) {
          this.state = "CLOSED";
          this.failureCount = 0;
        }
      } else {
        this.failureCount = 0;
      }

      return result;
    } catch (error) {
      this.failureCount++;
      this.lastFailureTime = Date.now();

      if (this.failureCount >= this.options.failureThreshold) {
        this.state = "OPEN";
      }

      throw error;
    }
  }

  getState(): CircuitState {
    return this.state;
  }

  reset() {
    this.state = "CLOSED";
    this.failureCount = 0;
    this.successCount = 0;
  }
}

class CircuitOpenError extends Error {
  constructor(message: string) {
    super(message);
    this.name = "CircuitOpenError";
  }
}

// Usage
const paymentCircuit = new CircuitBreaker({
  failureThreshold: 5,
  resetTimeout: 30_000,       // 30s before half-open
  halfOpenMaxAttempts: 3,
  fallback: () => ({ status: "unavailable", message: "Payment service temporarily unavailable" }),
});

const result = await paymentCircuit.execute(() =>
  stripeClient.post("/charges", { amount: 2000, currency: "eur" })
);
```

---

## 3. OAuth2 Refresh Flow

Access token + refresh token flow with automatic refresh on 401 and secure token storage in httpOnly cookies.

```tsx
// lib/oauth2.ts
interface TokenPair {
  accessToken: string;
  refreshToken: string;
  expiresAt: number; // Unix timestamp ms
}

// --- Server-side token management (Route Handlers) ---

// app/api/auth/callback/route.ts
import { cookies } from "next/headers";

export async function GET(request: Request) {
  const { searchParams } = new URL(request.url);
  const code = searchParams.get("code");

  // Exchange authorization code for tokens
  const tokenResponse = await fetch("https://provider.com/oauth/token", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      grant_type: "authorization_code",
      code,
      client_id: process.env.OAUTH_CLIENT_ID,
      client_secret: process.env.OAUTH_CLIENT_SECRET,
      redirect_uri: `${process.env.NEXT_PUBLIC_APP_URL}/api/auth/callback`,
    }),
  });

  const tokens = await tokenResponse.json();

  const cookieStore = await cookies();

  // Store tokens in httpOnly cookies - never expose to client JS
  cookieStore.set("access_token", tokens.access_token, {
    httpOnly: true,
    secure: true,
    sameSite: "lax",
    maxAge: tokens.expires_in,
    path: "/",
  });

  cookieStore.set("refresh_token", tokens.refresh_token, {
    httpOnly: true,
    secure: true,
    sameSite: "lax",
    maxAge: 30 * 24 * 60 * 60, // 30 days
    path: "/",
  });

  return Response.redirect(`${process.env.NEXT_PUBLIC_APP_URL}/dashboard`);
}

// lib/oauth2-refresh.ts
async function refreshAccessToken(refreshToken: string): Promise<TokenPair> {
  const response = await fetch("https://provider.com/oauth/token", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      grant_type: "refresh_token",
      refresh_token: refreshToken,
      client_id: process.env.OAUTH_CLIENT_ID,
      client_secret: process.env.OAUTH_CLIENT_SECRET,
    }),
  });

  if (!response.ok) {
    throw new Error("Token refresh failed - user must re-authenticate");
  }

  const data = await response.json();
  return {
    accessToken: data.access_token,
    refreshToken: data.refresh_token ?? refreshToken,
    expiresAt: Date.now() + data.expires_in * 1000,
  };
}

// Auto-refresh interceptor for ApiClient
function createOAuth2Interceptor(getRefreshToken: () => string) {
  let isRefreshing = false;
  let refreshPromise: Promise<TokenPair> | null = null;

  return async (response: Response): Promise<Response> => {
    if (response.status !== 401) return response;

    if (!isRefreshing) {
      isRefreshing = true;
      refreshPromise = refreshAccessToken(getRefreshToken());
    }

    try {
      const tokens = await refreshPromise!;
      isRefreshing = false;

      // Retry original request with new token
      const retryResponse = await fetch(response.url, {
        headers: { Authorization: `Bearer ${tokens.accessToken}` },
      });
      return retryResponse;
    } catch {
      isRefreshing = false;
      throw new Error("Authentication expired");
    }
  };
}
```

---

## 4. Retry with Exponential Backoff

Retry utility with exponential backoff, jitter, max retries, and configurable retry-able status codes.

```tsx
// lib/retry.ts
interface RetryOptions {
  maxRetries: number;
  baseDelay: number;           // ms
  maxDelay: number;            // ms
  jitter: boolean;
  retryableStatuses: number[]; // HTTP status codes that trigger retry
  onRetry?: (attempt: number, error: Error, delay: number) => void;
}

const DEFAULT_RETRY_OPTIONS: RetryOptions = {
  maxRetries: 3,
  baseDelay: 1000,
  maxDelay: 30_000,
  jitter: true,
  retryableStatuses: [408, 429, 500, 502, 503, 504],
};

async function withRetry<T>(
  fn: () => Promise<T>,
  options: Partial<RetryOptions> = {}
): Promise<T> {
  const opts = { ...DEFAULT_RETRY_OPTIONS, ...options };
  let lastError: Error | undefined;

  for (let attempt = 0; attempt <= opts.maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error as Error;

      // Check if error is retryable
      const isRetryable =
        error instanceof ApiError &&
        opts.retryableStatuses.includes(error.status);

      if (!isRetryable || attempt === opts.maxRetries) {
        throw error;
      }

      // Calculate delay: exponential backoff with optional jitter
      let delay = Math.min(
        opts.baseDelay * Math.pow(2, attempt),
        opts.maxDelay
      );

      if (opts.jitter) {
        delay = delay * (0.5 + Math.random() * 0.5); // 50-100% of calculated delay
      }

      // If 429 with Retry-After header, use that instead
      if (
        error instanceof ApiError &&
        error.status === 429
      ) {
        const retryAfter = parseRetryAfterHeader(error);
        if (retryAfter) delay = retryAfter;
      }

      opts.onRetry?.(attempt + 1, lastError, delay);

      await sleep(delay);
    }
  }

  throw lastError;
}

function parseRetryAfterHeader(error: ApiError): number | null {
  // Retry-After can be seconds or HTTP-date
  // Parse from error body or headers if available
  try {
    const parsed = JSON.parse(error.body);
    if (parsed.retry_after) return parsed.retry_after * 1000;
  } catch {}
  return null;
}

function sleep(ms: number): Promise<void> {
  return new Promise((resolve) => setTimeout(resolve, ms));
}

// Usage
const data = await withRetry(
  () => apiClient.get("/unstable-endpoint"),
  {
    maxRetries: 5,
    baseDelay: 500,
    onRetry: (attempt, error, delay) => {
      console.warn(`Retry ${attempt} after ${delay}ms: ${error.message}`);
    },
  }
);
```

---

## 5. Webhook Integration

Receive and verify webhooks from third-party services. Patterns for Stripe, GitHub, and generic HMAC.

```tsx
// app/api/webhooks/stripe/route.ts
import Stripe from "stripe";

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!);
const endpointSecret = process.env.STRIPE_WEBHOOK_SECRET!;

export async function POST(request: Request) {
  const body = await request.text();
  const signature = request.headers.get("stripe-signature")!;

  let event: Stripe.Event;

  try {
    event = stripe.webhooks.constructEvent(body, signature, endpointSecret);
  } catch (err) {
    console.error("Webhook signature verification failed:", err);
    return Response.json({ error: "Invalid signature" }, { status: 400 });
  }

  // Process event idempotently
  try {
    await processWebhookEvent(event);
  } catch (err) {
    console.error("Webhook processing failed:", err);
    // Return 200 anyway to prevent Stripe from retrying
    // Log the error and handle via dead-letter queue
  }

  return Response.json({ received: true });
}

async function processWebhookEvent(event: Stripe.Event) {
  // Idempotency: check if already processed
  const existing = await db.webhookEvent.findUnique({
    where: { eventId: event.id },
  });
  if (existing) return; // Already processed

  switch (event.type) {
    case "checkout.session.completed":
      await handleCheckoutComplete(event.data.object);
      break;
    case "customer.subscription.updated":
      await handleSubscriptionUpdate(event.data.object);
      break;
    case "invoice.payment_failed":
      await handlePaymentFailed(event.data.object);
      break;
  }

  // Mark as processed
  await db.webhookEvent.create({
    data: { eventId: event.id, type: event.type, processedAt: new Date() },
  });
}

// --- Generic HMAC verification ---
// app/api/webhooks/generic/route.ts
import { createHmac, timingSafeEqual } from "crypto";

function verifyHmacSignature(
  payload: string,
  signature: string,
  secret: string,
  algorithm: string = "sha256"
): boolean {
  const expected = createHmac(algorithm, secret).update(payload).digest("hex");

  // Constant-time comparison to prevent timing attacks
  return timingSafeEqual(
    Buffer.from(signature),
    Buffer.from(expected)
  );
}

// --- GitHub webhook verification ---
function verifyGitHubWebhook(payload: string, signature: string): boolean {
  const expected = `sha256=${createHmac("sha256", process.env.GITHUB_WEBHOOK_SECRET!)
    .update(payload)
    .digest("hex")}`;

  return timingSafeEqual(
    Buffer.from(signature),
    Buffer.from(expected)
  );
}
```

---

## 6. API Key Management

Server-side only key handling, environment variables, key rotation, and scoped permissions.

```tsx
// lib/api-keys.ts

// RULE 1: Never import API keys in client components
// RULE 2: Never include keys in client bundles
// RULE 3: Use environment variables without NEXT_PUBLIC_ prefix

// --- Key configuration ---
// .env.local (NEVER committed)
// STRIPE_SECRET_KEY=sk_live_...
// STRIPE_PUBLISHABLE_KEY=pk_live_...  (this one is ok for client)
// OPENAI_API_KEY=sk-...
// RESEND_API_KEY=re_...

// --- Server-only key access ---
// lib/keys.server.ts
import "server-only"; // Prevents accidental import in client components

interface ApiKeyConfig {
  key: string;
  scopes: string[];
  rotatedAt: Date;
  expiresAt?: Date;
}

function getApiKey(service: string): string {
  const keyMap: Record<string, string | undefined> = {
    stripe: process.env.STRIPE_SECRET_KEY,
    openai: process.env.OPENAI_API_KEY,
    resend: process.env.RESEND_API_KEY,
    anthropic: process.env.ANTHROPIC_API_KEY,
    supabase_service: process.env.SUPABASE_SERVICE_ROLE_KEY,
  };

  const key = keyMap[service];
  if (!key) {
    throw new Error(`Missing API key for service: ${service}. Check environment variables.`);
  }

  return key;
}

// --- Key rotation pattern ---
// Support both current and previous key during rotation window
function getApiKeyWithFallback(service: string): { current: string; previous?: string } {
  const current = process.env[`${service.toUpperCase()}_API_KEY`];
  const previous = process.env[`${service.toUpperCase()}_API_KEY_PREVIOUS`];

  if (!current) {
    throw new Error(`Missing current API key for ${service}`);
  }

  return { current, previous };
}

async function callWithKeyRotation<T>(
  service: string,
  fn: (key: string) => Promise<T>
): Promise<T> {
  const { current, previous } = getApiKeyWithFallback(service);

  try {
    return await fn(current);
  } catch (error) {
    if (error instanceof ApiError && error.status === 401 && previous) {
      console.warn(`Current ${service} key failed, trying previous key`);
      return await fn(previous);
    }
    throw error;
  }
}

// --- Scoped API keys (for multi-tenant) ---
// Store per-tenant API keys encrypted in database
import { createCipheriv, createDecipheriv, randomBytes } from "crypto";

const ENCRYPTION_KEY = Buffer.from(process.env.ENCRYPTION_KEY!, "hex"); // 32 bytes
const IV_LENGTH = 16;

function encryptApiKey(key: string): string {
  const iv = randomBytes(IV_LENGTH);
  const cipher = createCipheriv("aes-256-gcm", ENCRYPTION_KEY, iv);
  let encrypted = cipher.update(key, "utf8", "hex");
  encrypted += cipher.final("hex");
  const authTag = cipher.getAuthTag().toString("hex");
  return `${iv.toString("hex")}:${authTag}:${encrypted}`;
}

function decryptApiKey(encrypted: string): string {
  const [ivHex, authTagHex, data] = encrypted.split(":");
  const iv = Buffer.from(ivHex, "hex");
  const authTag = Buffer.from(authTagHex, "hex");
  const decipher = createDecipheriv("aes-256-gcm", ENCRYPTION_KEY, iv);
  decipher.setAuthTag(authTag);
  let decrypted = decipher.update(data, "hex", "utf8");
  decrypted += decipher.final("utf8");
  return decrypted;
}
```

---

## 7. Rate Limit Handling

Respect X-RateLimit headers, implement client-side queuing, backpressure, and 429 retry-after.

```tsx
// lib/rate-limiter.ts
interface RateLimitState {
  remaining: number;
  limit: number;
  resetAt: number; // Unix timestamp ms
}

class RateLimitedClient {
  private queue: Array<{
    fn: () => Promise<unknown>;
    resolve: (value: unknown) => void;
    reject: (error: unknown) => void;
  }> = [];
  private processing = false;
  private rateLimitState: RateLimitState | null = null;

  constructor(private client: ApiClient) {}

  async request<T>(fn: () => Promise<T>): Promise<T> {
    return new Promise<T>((resolve, reject) => {
      this.queue.push({
        fn: fn as () => Promise<unknown>,
        resolve: resolve as (value: unknown) => void,
        reject,
      });
      this.processQueue();
    });
  }

  private async processQueue() {
    if (this.processing || this.queue.length === 0) return;
    this.processing = true;

    while (this.queue.length > 0) {
      // Check rate limit state
      if (this.rateLimitState && this.rateLimitState.remaining <= 1) {
        const waitTime = this.rateLimitState.resetAt - Date.now();
        if (waitTime > 0) {
          console.log(`Rate limit approaching, waiting ${waitTime}ms`);
          await sleep(waitTime);
        }
      }

      const item = this.queue.shift()!;

      try {
        const result = await item.fn();
        item.resolve(result);
      } catch (error) {
        if (error instanceof ApiError && error.status === 429) {
          // Re-queue the request
          this.queue.unshift(item);

          const retryAfter = this.parseRetryAfter(error);
          console.warn(`Rate limited. Retrying after ${retryAfter}ms`);
          await sleep(retryAfter);
        } else {
          item.reject(error);
        }
      }
    }

    this.processing = false;
  }

  // Call this from a response interceptor to update state
  updateFromHeaders(headers: Headers) {
    const remaining = headers.get("x-ratelimit-remaining");
    const limit = headers.get("x-ratelimit-limit");
    const reset = headers.get("x-ratelimit-reset");

    if (remaining && limit && reset) {
      this.rateLimitState = {
        remaining: parseInt(remaining, 10),
        limit: parseInt(limit, 10),
        resetAt: parseInt(reset, 10) * 1000, // Convert to ms
      };
    }
  }

  private parseRetryAfter(error: ApiError): number {
    try {
      const parsed = JSON.parse(error.body);
      if (parsed.retry_after) return parsed.retry_after * 1000;
    } catch {}
    return 60_000; // Default 60s
  }
}

// Usage
const rateLimited = new RateLimitedClient(apiClient);

// All calls go through the queue
const users = await rateLimited.request(() => apiClient.get("/users"));
const posts = await rateLimited.request(() => apiClient.get("/posts"));
```

---

## 8. Data Sync Patterns

Periodic sync (cron), event-driven sync (webhooks), conflict resolution, and last-write-wins.

```tsx
// lib/sync/types.ts
interface SyncRecord {
  id: string;
  externalId: string;
  source: string;
  data: Record<string, unknown>;
  syncedAt: Date;
  checksum: string;
}

interface SyncResult {
  created: number;
  updated: number;
  deleted: number;
  conflicts: SyncConflict[];
  errors: SyncError[];
}

interface SyncConflict {
  id: string;
  localData: Record<string, unknown>;
  remoteData: Record<string, unknown>;
  resolution: "local_wins" | "remote_wins" | "manual";
}

// --- Periodic sync (cron-based) ---
// app/api/cron/sync-contacts/route.ts
import { createHash } from "crypto";

export async function GET(request: Request) {
  // Verify cron secret (Vercel Cron or custom)
  const authHeader = request.headers.get("authorization");
  if (authHeader !== `Bearer ${process.env.CRON_SECRET}`) {
    return Response.json({ error: "Unauthorized" }, { status: 401 });
  }

  const result = await syncContacts();
  return Response.json(result);
}

async function syncContacts(): Promise<SyncResult> {
  const result: SyncResult = { created: 0, updated: 0, deleted: 0, conflicts: [], errors: [] };

  // Fetch all remote contacts
  const remoteContacts = await externalApi.get<Contact[]>("/contacts");

  // Fetch all local synced records
  const localRecords = await db.syncRecord.findMany({
    where: { source: "external-crm" },
  });
  const localMap = new Map(localRecords.map((r) => [r.externalId, r]));

  for (const remote of remoteContacts) {
    const checksum = createHash("md5").update(JSON.stringify(remote)).digest("hex");
    const local = localMap.get(remote.id);

    if (!local) {
      // New record - create locally
      await db.contact.create({ data: mapToLocal(remote) });
      await db.syncRecord.create({
        data: { externalId: remote.id, source: "external-crm", checksum, syncedAt: new Date() },
      });
      result.created++;
    } else if (local.checksum !== checksum) {
      // Changed - update with last-write-wins
      await db.contact.update({
        where: { externalId: remote.id },
        data: mapToLocal(remote),
      });
      await db.syncRecord.update({
        where: { id: local.id },
        data: { checksum, syncedAt: new Date() },
      });
      result.updated++;
    }

    localMap.delete(remote.id);
  }

  // Remaining in localMap were deleted remotely
  for (const [externalId] of localMap) {
    await db.contact.delete({ where: { externalId } });
    result.deleted++;
  }

  return result;
}

// --- Event-driven sync (webhook-based) ---
// app/api/webhooks/crm/route.ts
export async function POST(request: Request) {
  const event = await request.json();

  switch (event.type) {
    case "contact.created":
    case "contact.updated":
      await upsertContact(event.data);
      break;
    case "contact.deleted":
      await db.contact.delete({ where: { externalId: event.data.id } });
      break;
  }

  return Response.json({ received: true });
}

// --- Conflict resolution strategies ---
type ConflictStrategy = "last_write_wins" | "remote_wins" | "local_wins" | "merge";

function resolveConflict(
  local: Record<string, unknown>,
  remote: Record<string, unknown>,
  strategy: ConflictStrategy
): Record<string, unknown> {
  switch (strategy) {
    case "last_write_wins": {
      const localTime = (local.updatedAt as Date).getTime();
      const remoteTime = (remote.updatedAt as Date).getTime();
      return remoteTime >= localTime ? remote : local;
    }
    case "remote_wins":
      return remote;
    case "local_wins":
      return local;
    case "merge":
      // Field-level merge: remote wins per-field on conflict
      return { ...local, ...remote };
  }
}

// vercel.json - cron config
// {
//   "crons": [
//     { "path": "/api/cron/sync-contacts", "schedule": "0 */6 * * *" }
//   ]
// }
```

---

## 9. Error Mapping

Map third-party errors to internal error types, provide user-friendly messages, and log the original error.

```tsx
// lib/errors.ts

// --- Internal error types ---
class AppError extends Error {
  constructor(
    public code: string,
    public userMessage: string,
    public statusCode: number = 500,
    public context?: Record<string, unknown>
  ) {
    super(userMessage);
    this.name = "AppError";
  }
}

// Specific error classes
class PaymentError extends AppError {
  constructor(code: string, userMessage: string, context?: Record<string, unknown>) {
    super(code, userMessage, 402, context);
    this.name = "PaymentError";
  }
}

class ExternalServiceError extends AppError {
  constructor(service: string, userMessage: string, context?: Record<string, unknown>) {
    super(`${service}_error`, userMessage, 502, context);
    this.name = "ExternalServiceError";
  }
}

class ValidationError extends AppError {
  constructor(message: string, context?: Record<string, unknown>) {
    super("validation_error", message, 400, context);
    this.name = "ValidationError";
  }
}

// --- Error mapping per provider ---
interface ErrorMapping {
  match: (error: unknown) => boolean;
  map: (error: unknown) => AppError;
}

const stripeErrorMappings: ErrorMapping[] = [
  {
    match: (e) => e instanceof Error && e.message.includes("card_declined"),
    map: () => new PaymentError("card_declined", "Your card was declined. Please try a different payment method."),
  },
  {
    match: (e) => e instanceof Error && e.message.includes("expired_card"),
    map: () => new PaymentError("expired_card", "Your card has expired. Please update your payment method."),
  },
  {
    match: (e) => e instanceof Error && e.message.includes("insufficient_funds"),
    map: () => new PaymentError("insufficient_funds", "Insufficient funds. Please try a different payment method."),
  },
  {
    match: (e) => e instanceof ApiError && e.status === 429,
    map: () => new ExternalServiceError("stripe", "Payment service is busy. Please try again in a moment."),
  },
];

function mapExternalError(
  error: unknown,
  service: string,
  mappings: ErrorMapping[]
): AppError {
  // Log the original error for debugging
  console.error(`[${service}] Original error:`, error);

  for (const mapping of mappings) {
    if (mapping.match(error)) {
      return mapping.map(error);
    }
  }

  // Default fallback
  return new ExternalServiceError(
    service,
    "Something went wrong. Please try again later.",
    {
      originalMessage: error instanceof Error ? error.message : String(error),
    }
  );
}

// --- Usage in API routes ---
// app/api/payments/route.ts
export async function POST(request: Request) {
  try {
    const body = await request.json();
    const result = await processPayment(body);
    return Response.json(result);
  } catch (error) {
    const appError = mapExternalError(error, "stripe", stripeErrorMappings);

    return Response.json(
      {
        error: {
          code: appError.code,
          message: appError.userMessage,
        },
      },
      { status: appError.statusCode }
    );
  }
}

// --- Centralized error handler for route handlers ---
type RouteHandler = (request: Request) => Promise<Response>;

function withErrorHandling(handler: RouteHandler): RouteHandler {
  return async (request: Request) => {
    try {
      return await handler(request);
    } catch (error) {
      if (error instanceof AppError) {
        return Response.json(
          { error: { code: error.code, message: error.userMessage } },
          { status: error.statusCode }
        );
      }

      console.error("Unhandled error:", error);
      return Response.json(
        { error: { code: "internal_error", message: "An unexpected error occurred." } },
        { status: 500 }
      );
    }
  };
}
```

---

## 10. Common Integrations

Setup patterns for Stripe, Resend, Supabase, OpenAI/Anthropic, and Google APIs.

### Stripe

```tsx
// lib/stripe.ts
import Stripe from "stripe";
import "server-only";

export const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, {
  apiVersion: "2024-12-18.acacia",
  typescript: true,
});

// Create checkout session
export async function createCheckoutSession(params: {
  priceId: string;
  userId: string;
  successUrl: string;
  cancelUrl: string;
}) {
  return stripe.checkout.sessions.create({
    mode: "subscription",
    payment_method_types: ["card"],
    line_items: [{ price: params.priceId, quantity: 1 }],
    success_url: params.successUrl,
    cancel_url: params.cancelUrl,
    client_reference_id: params.userId,
    metadata: { userId: params.userId },
  });
}

// Create billing portal session
export async function createPortalSession(customerId: string, returnUrl: string) {
  return stripe.billingPortal.sessions.create({
    customer: customerId,
    return_url: returnUrl,
  });
}
```

### Resend (Email)

```tsx
// lib/resend.ts
import { Resend } from "resend";
import "server-only";

const resend = new Resend(process.env.RESEND_API_KEY);

interface SendEmailParams {
  to: string | string[];
  subject: string;
  react: React.ReactElement; // Use React Email templates
}

export async function sendEmail({ to, subject, react }: SendEmailParams) {
  const { data, error } = await resend.emails.send({
    from: "App <noreply@yourdomain.com>",
    to: Array.isArray(to) ? to : [to],
    subject,
    react,
  });

  if (error) {
    throw mapExternalError(error, "resend", []);
  }

  return data;
}
```

### Supabase

```tsx
// lib/supabase/server.ts
import { createServerClient } from "@supabase/ssr";
import { cookies } from "next/headers";

export async function createClient() {
  const cookieStore = await cookies();

  return createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() {
          return cookieStore.getAll();
        },
        setAll(cookiesToSet) {
          try {
            cookiesToSet.forEach(({ name, value, options }) =>
              cookieStore.set(name, value, options)
            );
          } catch {
            // Cookies can only be set in Server Actions or Route Handlers
          }
        },
      },
    }
  );
}

// Admin client (service role - bypasses RLS)
// lib/supabase/admin.ts
import { createClient } from "@supabase/supabase-js";
import "server-only";

export const supabaseAdmin = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!, // Never expose this
  { auth: { autoRefreshToken: false, persistSession: false } }
);
```

### OpenAI / Anthropic

```tsx
// lib/ai/openai.ts
import OpenAI from "openai";
import "server-only";

export const openai = new OpenAI({
  apiKey: process.env.OPENAI_API_KEY,
});

// Streaming completion
export async function streamCompletion(prompt: string) {
  return openai.chat.completions.create({
    model: "gpt-4o",
    messages: [{ role: "user", content: prompt }],
    stream: true,
  });
}

// lib/ai/anthropic.ts
import Anthropic from "@anthropic-ai/sdk";
import "server-only";

export const anthropic = new Anthropic({
  apiKey: process.env.ANTHROPIC_API_KEY,
});

export async function streamMessage(prompt: string) {
  return anthropic.messages.stream({
    model: "claude-sonnet-4-20250514",
    max_tokens: 4096,
    messages: [{ role: "user", content: prompt }],
  });
}

// --- AI SDK (Vercel) - unified interface ---
// lib/ai/index.ts
import { openai } from "@ai-sdk/openai";
import { anthropic } from "@ai-sdk/anthropic";
import { streamText } from "ai";

export async function chat(messages: Array<{ role: string; content: string }>) {
  return streamText({
    model: anthropic("claude-sonnet-4-20250514"), // or openai("gpt-4o")
    messages,
  });
}
```

### Google APIs

```tsx
// lib/google.ts
import { google } from "googleapis";
import "server-only";

// Service account auth (server-to-server)
const auth = new google.auth.GoogleAuth({
  credentials: JSON.parse(process.env.GOOGLE_SERVICE_ACCOUNT_KEY!),
  scopes: ["https://www.googleapis.com/auth/analytics.readonly"],
});

export const analytics = google.analyticsdata({ version: "v1beta", auth });

// OAuth2 for user-scoped access
export function createOAuth2Client() {
  return new google.auth.OAuth2(
    process.env.GOOGLE_CLIENT_ID,
    process.env.GOOGLE_CLIENT_SECRET,
    `${process.env.NEXT_PUBLIC_APP_URL}/api/auth/google/callback`
  );
}

// Usage: Google Analytics data
export async function getPageViews(propertyId: string, startDate: string, endDate: string) {
  const response = await analytics.properties.runReport({
    property: `properties/${propertyId}`,
    requestBody: {
      dateRanges: [{ startDate, endDate }],
      metrics: [{ name: "screenPageViews" }],
      dimensions: [{ name: "pagePath" }],
    },
  });

  return response.data.rows;
}
```
