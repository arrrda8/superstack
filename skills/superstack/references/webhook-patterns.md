# Webhook Patterns Reference

Production-ready webhook patterns for Next.js applications.

---

## 1. Receiving Webhooks

### Next.js Route Handler

```typescript
// app/api/webhooks/stripe/route.ts
import { NextRequest, NextResponse } from "next/server";

export async function POST(req: NextRequest) {
  // Get raw body for signature verification
  const rawBody = await req.text();
  const signature = req.headers.get("stripe-signature");

  if (!signature) {
    return NextResponse.json({ error: "Missing signature" }, { status: 400 });
  }

  try {
    const event = verifyAndParseWebhook(rawBody, signature);
    await processWebhookEvent(event);
    return NextResponse.json({ received: true }, { status: 200 });
  } catch (error) {
    console.error("Webhook error:", error);
    return NextResponse.json({ error: "Webhook failed" }, { status: 400 });
  }
}
```

### Raw body parsing

```typescript
// Next.js App Router gives you the raw body via req.text() or req.arrayBuffer()
// No special config needed (unlike Pages Router)

export async function POST(req: NextRequest) {
  // For text-based signatures (most webhooks)
  const rawBody = await req.text();

  // For binary payloads
  const rawBuffer = Buffer.from(await req.arrayBuffer());

  // Parse JSON after verification
  const payload = JSON.parse(rawBody);
}
```

### Async processing pattern

```typescript
// For long-running webhook processing: acknowledge immediately, process async
export async function POST(req: NextRequest) {
  const rawBody = await req.text();
  const signature = req.headers.get("x-webhook-signature")!;

  // Verify signature synchronously
  if (!verifySignature(rawBody, signature)) {
    return NextResponse.json({ error: "Invalid signature" }, { status: 401 });
  }

  const payload = JSON.parse(rawBody);

  // Queue for async processing (don't await)
  // Option 1: Database queue
  await db.webhookEvents.create({
    data: {
      source: "stripe",
      eventType: payload.type,
      payload,
      status: "pending",
    },
  });

  // Option 2: Upstash QStash for background processing
  // await qstash.publishJSON({
  //   url: `${process.env.APP_URL}/api/webhooks/process`,
  //   body: payload,
  // });

  // Respond immediately (within webhook timeout)
  return NextResponse.json({ received: true }, { status: 200 });
}
```

---

## 2. Signature Verification

### HMAC-SHA256 verification (generic)

```typescript
import { createHmac, timingSafeEqual } from "crypto";

function verifyHmacSignature(
  payload: string,
  signature: string,
  secret: string
): boolean {
  const expected = createHmac("sha256", secret).update(payload).digest("hex");

  // Timing-safe comparison prevents timing attacks
  try {
    return timingSafeEqual(
      Buffer.from(signature),
      Buffer.from(expected)
    );
  } catch {
    return false; // Different lengths
  }
}
```

### Stripe signature verification

```typescript
import Stripe from "stripe";

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!);

export async function POST(req: NextRequest) {
  const rawBody = await req.text();
  const signature = req.headers.get("stripe-signature")!;

  let event: Stripe.Event;
  try {
    event = stripe.webhooks.constructEvent(
      rawBody,
      signature,
      process.env.STRIPE_WEBHOOK_SECRET!
    );
  } catch (err) {
    console.error("Stripe signature verification failed:", err);
    return NextResponse.json({ error: "Invalid signature" }, { status: 400 });
  }

  // Process verified event
  switch (event.type) {
    case "checkout.session.completed":
      await handleCheckoutComplete(event.data.object as Stripe.Checkout.Session);
      break;
    case "invoice.payment_succeeded":
      await handlePaymentSuccess(event.data.object as Stripe.Invoice);
      break;
    case "customer.subscription.deleted":
      await handleSubscriptionCanceled(event.data.object as Stripe.Subscription);
      break;
  }

  return NextResponse.json({ received: true });
}
```

### GitHub signature verification

```typescript
import { createHmac, timingSafeEqual } from "crypto";

function verifyGitHubSignature(payload: string, signature: string): boolean {
  const expected = `sha256=${createHmac("sha256", process.env.GITHUB_WEBHOOK_SECRET!)
    .update(payload)
    .digest("hex")}`;

  return timingSafeEqual(Buffer.from(signature), Buffer.from(expected));
}

export async function POST(req: NextRequest) {
  const rawBody = await req.text();
  const signature = req.headers.get("x-hub-signature-256");

  if (!signature || !verifyGitHubSignature(rawBody, signature)) {
    return NextResponse.json({ error: "Invalid signature" }, { status: 401 });
  }

  const event = req.headers.get("x-github-event");
  const payload = JSON.parse(rawBody);

  switch (event) {
    case "push":
      await handlePush(payload);
      break;
    case "pull_request":
      await handlePR(payload);
      break;
    case "issues":
      await handleIssue(payload);
      break;
  }

  return NextResponse.json({ ok: true });
}
```

---

## 3. Idempotency

### Idempotency key storage

```typescript
// Prevent duplicate processing using a unique event identifier

async function processWebhookIdempotent(eventId: string, handler: () => Promise<void>) {
  // Check if already processed (Redis for speed)
  const processed = await redis.get(`webhook:processed:${eventId}`);
  if (processed) {
    console.log(`Webhook ${eventId} already processed, skipping`);
    return;
  }

  // Set processing flag with TTL (prevents race conditions)
  const acquired = await redis.set(`webhook:processing:${eventId}`, "1", {
    ex: 300, // 5 min lock
    nx: true, // only if not exists
  });

  if (!acquired) {
    console.log(`Webhook ${eventId} is being processed by another worker`);
    return;
  }

  try {
    await handler();
    // Mark as processed with long TTL
    await redis.set(`webhook:processed:${eventId}`, "1", { ex: 86400 * 7 }); // 7 days
  } catch (error) {
    // Remove processing flag so it can be retried
    await redis.del(`webhook:processing:${eventId}`);
    throw error;
  }
}

// Usage
export async function POST(req: NextRequest) {
  const payload = await verifyAndParse(req);

  await processWebhookIdempotent(payload.id, async () => {
    await handleStripeEvent(payload);
  });

  return NextResponse.json({ received: true });
}
```

### Database upsert pattern

```typescript
// Use database upsert for durable idempotency
async function processOrderWebhook(event: WebhookEvent) {
  // Upsert ensures idempotency at the database level
  const result = await db.webhookEvents.upsert({
    where: { eventId: event.id },
    create: {
      eventId: event.id,
      source: "stripe",
      type: event.type,
      payload: event,
      status: "processing",
      receivedAt: new Date(),
    },
    update: {}, // No-op if already exists
  });

  // Only process if we just created it
  if (result.status !== "processing") {
    return; // Already processed or processing
  }

  try {
    await handleEvent(event);
    await db.webhookEvents.update({
      where: { eventId: event.id },
      data: { status: "completed", processedAt: new Date() },
    });
  } catch (error) {
    await db.webhookEvents.update({
      where: { eventId: event.id },
      data: {
        status: "failed",
        error: error instanceof Error ? error.message : "Unknown error",
        failedAt: new Date(),
      },
    });
    throw error;
  }
}
```

---

## 4. Retry Handling

### Responding with correct status codes

```typescript
export async function POST(req: NextRequest) {
  try {
    const payload = await verifyAndParse(req);

    // 200: Successfully processed — don't retry
    await processEvent(payload);
    return NextResponse.json({ received: true }, { status: 200 });
  } catch (error) {
    if (error instanceof SignatureVerificationError) {
      // 401: Bad signature — don't retry (not recoverable)
      return NextResponse.json({ error: "Invalid signature" }, { status: 401 });
    }

    if (error instanceof ValidationError) {
      // 400: Bad payload — don't retry (not recoverable)
      return NextResponse.json({ error: "Invalid payload" }, { status: 400 });
    }

    // 500: Internal error — sender should retry
    console.error("Webhook processing failed:", error);
    return NextResponse.json({ error: "Processing failed" }, { status: 500 });
  }
}

// Status code guide for webhook receivers:
// 2xx -> Success, don't retry
// 4xx -> Client error, don't retry (except 408, 429)
// 5xx -> Server error, retry with backoff
```

### Timeout handling

```typescript
// Webhook senders typically have 5-30 second timeouts
// For long processing: acknowledge fast, process async

export async function POST(req: NextRequest) {
  const payload = await verifyAndParse(req);

  // Quick validation only
  if (!isValidEvent(payload)) {
    return NextResponse.json({ error: "Invalid event" }, { status: 400 });
  }

  // Queue for background processing
  await db.webhookQueue.create({
    data: {
      eventId: payload.id,
      payload,
      status: "queued",
      attempts: 0,
    },
  });

  // Respond within timeout
  return NextResponse.json({ received: true, queued: true }, { status: 202 });
}

// Background worker (cron or separate process)
async function processWebhookQueue() {
  const pending = await db.webhookQueue.findMany({
    where: { status: "queued" },
    orderBy: { createdAt: "asc" },
    take: 10,
  });

  for (const item of pending) {
    try {
      await processEvent(item.payload);
      await db.webhookQueue.update({
        where: { id: item.id },
        data: { status: "completed" },
      });
    } catch (error) {
      const attempts = item.attempts + 1;
      await db.webhookQueue.update({
        where: { id: item.id },
        data: {
          status: attempts >= 5 ? "dead" : "queued",
          attempts,
          lastError: error instanceof Error ? error.message : "Unknown",
          nextRetryAt: new Date(Date.now() + Math.pow(2, attempts) * 1000),
        },
      });
    }
  }
}
```

---

## 5. Dead Letter Queue

### Failed webhook storage

```typescript
// Schema for webhook dead letter queue
// CREATE TABLE webhook_dead_letters (
//   id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
//   source TEXT NOT NULL,
//   event_id TEXT NOT NULL,
//   event_type TEXT NOT NULL,
//   payload JSONB NOT NULL,
//   error TEXT,
//   attempts INTEGER DEFAULT 0,
//   first_received_at TIMESTAMPTZ NOT NULL,
//   last_attempted_at TIMESTAMPTZ,
//   resolved_at TIMESTAMPTZ,
//   status TEXT DEFAULT 'failed' CHECK (status IN ('failed', 'retrying', 'resolved', 'ignored'))
// );

async function moveToDeadLetter(event: WebhookEvent, error: Error) {
  await db.webhookDeadLetters.create({
    data: {
      source: event.source,
      eventId: event.id,
      eventType: event.type,
      payload: event,
      error: error.message,
      attempts: event.attempts,
      firstReceivedAt: event.receivedAt,
      lastAttemptedAt: new Date(),
    },
  });

  // Alert the team
  await sendAlert({
    channel: "webhook-failures",
    message: `Webhook failed after ${event.attempts} attempts: ${event.type} from ${event.source}`,
    metadata: { eventId: event.id, error: error.message },
  });
}
```

### Manual retry API

```typescript
// app/api/admin/webhooks/retry/route.ts
export async function POST(req: NextRequest) {
  // Admin auth check
  const session = await getAdminSession(req);
  if (!session) return NextResponse.json({ error: "Unauthorized" }, { status: 401 });

  const { deadLetterId } = await req.json();

  const deadLetter = await db.webhookDeadLetters.findUnique({
    where: { id: deadLetterId },
  });

  if (!deadLetter) {
    return NextResponse.json({ error: "Not found" }, { status: 404 });
  }

  try {
    await db.webhookDeadLetters.update({
      where: { id: deadLetterId },
      data: { status: "retrying", lastAttemptedAt: new Date() },
    });

    await processEvent(deadLetter.payload);

    await db.webhookDeadLetters.update({
      where: { id: deadLetterId },
      data: { status: "resolved", resolvedAt: new Date() },
    });

    return NextResponse.json({ success: true });
  } catch (error) {
    await db.webhookDeadLetters.update({
      where: { id: deadLetterId },
      data: {
        status: "failed",
        attempts: deadLetter.attempts + 1,
        error: error instanceof Error ? error.message : "Unknown",
      },
    });
    return NextResponse.json({ error: "Retry failed" }, { status: 500 });
  }
}
```

---

## 6. Sending Webhooks

### Event system

```typescript
// lib/webhooks/sender.ts
interface WebhookSubscription {
  id: string;
  url: string;
  secret: string;
  events: string[];
  active: boolean;
}

async function emitWebhookEvent(eventType: string, data: unknown) {
  // Find all subscriptions for this event
  const subscriptions = await db.webhookSubscriptions.findMany({
    where: {
      active: true,
      events: { has: eventType },
    },
  });

  // Deliver to each subscriber
  const deliveries = subscriptions.map((sub) =>
    deliverWebhook(sub, eventType, data)
  );

  await Promise.allSettled(deliveries);
}

// Usage
await emitWebhookEvent("order.created", { orderId: "ord-123", total: 99.99 });
```

### HTTP delivery with retry

```typescript
import { createHmac } from "crypto";

async function deliverWebhook(
  subscription: WebhookSubscription,
  eventType: string,
  data: unknown,
  attempt: number = 1
) {
  const payload = JSON.stringify({
    id: crypto.randomUUID(),
    type: eventType,
    data,
    timestamp: new Date().toISOString(),
  });

  const signature = createHmac("sha256", subscription.secret)
    .update(payload)
    .digest("hex");

  const deliveryId = crypto.randomUUID();

  try {
    const response = await fetch(subscription.url, {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        "X-Webhook-Signature": `sha256=${signature}`,
        "X-Webhook-Id": deliveryId,
        "X-Webhook-Timestamp": new Date().toISOString(),
      },
      body: payload,
      signal: AbortSignal.timeout(10_000), // 10s timeout
    });

    // Log delivery
    await db.webhookDeliveries.create({
      data: {
        id: deliveryId,
        subscriptionId: subscription.id,
        eventType,
        payload,
        statusCode: response.status,
        success: response.ok,
        attempt,
      },
    });

    if (!response.ok && attempt < 5) {
      // Schedule retry with exponential backoff
      const delayMs = Math.pow(2, attempt) * 1000; // 2s, 4s, 8s, 16s
      setTimeout(() => deliverWebhook(subscription, eventType, data, attempt + 1), delayMs);
    }
  } catch (error) {
    await db.webhookDeliveries.create({
      data: {
        id: deliveryId,
        subscriptionId: subscription.id,
        eventType,
        payload,
        statusCode: 0,
        success: false,
        error: error instanceof Error ? error.message : "Network error",
        attempt,
      },
    });

    if (attempt < 5) {
      const delayMs = Math.pow(2, attempt) * 1000;
      setTimeout(() => deliverWebhook(subscription, eventType, data, attempt + 1), delayMs);
    }
  }
}
```

### Webhook registration API

```typescript
// app/api/webhooks/subscriptions/route.ts
import { createHmac, randomBytes } from "crypto";

export async function POST(req: NextRequest) {
  const session = await getSession(req);
  if (!session) return NextResponse.json({ error: "Unauthorized" }, { status: 401 });

  const { url, events } = await req.json();

  // Generate signing secret
  const secret = `whsec_${randomBytes(32).toString("hex")}`;

  const subscription = await db.webhookSubscriptions.create({
    data: {
      orgId: session.orgId,
      url,
      secret,
      events,
      active: true,
    },
  });

  // Return secret only once — user must store it
  return NextResponse.json({
    id: subscription.id,
    url: subscription.url,
    secret, // Show only on creation
    events: subscription.events,
  });
}

// List subscriptions (secret is hidden)
export async function GET(req: NextRequest) {
  const session = await getSession(req);
  if (!session) return NextResponse.json({ error: "Unauthorized" }, { status: 401 });

  const subscriptions = await db.webhookSubscriptions.findMany({
    where: { orgId: session.orgId },
    select: { id: true, url: true, events: true, active: true, createdAt: true },
  });

  return NextResponse.json(subscriptions);
}
```

---

## 7. Security

### IP allowlisting

```typescript
// Middleware or route-level IP check
const ALLOWED_IPS: Record<string, string[]> = {
  stripe: [
    "3.18.12.63", "3.130.192.231", "13.235.14.237",
    "13.235.122.149", "18.211.135.69", "35.154.171.200",
    "52.15.183.38", "54.187.174.169", "54.187.205.235",
    "54.187.216.72",
  ],
  github: ["140.82.112.0/20", "185.199.108.0/22", "192.30.252.0/22"],
};

function isAllowedIP(source: string, ip: string): boolean {
  const allowed = ALLOWED_IPS[source];
  if (!allowed) return true; // No restriction for unknown sources

  return allowed.some((allowedIp) => {
    if (allowedIp.includes("/")) {
      return isInCIDR(ip, allowedIp); // Implement CIDR check
    }
    return ip === allowedIp;
  });
}

export async function POST(req: NextRequest) {
  const ip = req.headers.get("x-forwarded-for")?.split(",")[0]?.trim();
  if (!ip || !isAllowedIP("stripe", ip)) {
    return NextResponse.json({ error: "Forbidden" }, { status: 403 });
  }
  // Process webhook...
}
```

### Replay attack prevention (timestamp validation)

```typescript
function verifyTimestamp(timestampHeader: string, toleranceSeconds: number = 300): boolean {
  const timestamp = parseInt(timestampHeader, 10);
  const now = Math.floor(Date.now() / 1000);
  const diff = Math.abs(now - timestamp);

  // Reject if older than tolerance (default 5 minutes)
  return diff <= toleranceSeconds;
}

// Stripe-style: timestamp is part of the signed payload
function verifyStripeWebhook(payload: string, sigHeader: string, secret: string) {
  const parts = sigHeader.split(",");
  const timestamp = parts.find((p) => p.startsWith("t="))?.slice(2);
  const signature = parts.find((p) => p.startsWith("v1="))?.slice(3);

  if (!timestamp || !signature) throw new Error("Invalid signature header");

  // Check timestamp freshness
  if (!verifyTimestamp(timestamp)) {
    throw new Error("Webhook timestamp too old — possible replay attack");
  }

  // Verify signature includes timestamp (prevents replay)
  const signedPayload = `${timestamp}.${payload}`;
  const expected = createHmac("sha256", secret).update(signedPayload).digest("hex");

  if (!timingSafeEqual(Buffer.from(signature), Buffer.from(expected))) {
    throw new Error("Invalid signature");
  }
}
```

### Secret rotation

```typescript
// Support multiple secrets during rotation period
async function verifyWithRotation(
  payload: string,
  signature: string,
  secrets: string[]
): Promise<boolean> {
  for (const secret of secrets) {
    const expected = createHmac("sha256", secret).update(payload).digest("hex");
    try {
      if (timingSafeEqual(Buffer.from(signature), Buffer.from(expected))) {
        return true;
      }
    } catch {
      continue;
    }
  }
  return false;
}

// Usage: support current + previous secret during rotation
const secrets = [
  process.env.WEBHOOK_SECRET_CURRENT!,
  process.env.WEBHOOK_SECRET_PREVIOUS!, // Remove after rotation period
].filter(Boolean);

const valid = await verifyWithRotation(rawBody, signature, secrets);
```

---

## 8. Testing

### ngrok for local development

```bash
# Install ngrok
brew install ngrok

# Expose local Next.js dev server
ngrok http 3000

# Use the HTTPS URL for webhook registration
# https://abc123.ngrok-free.app/api/webhooks/stripe
```

### Stripe CLI for local testing

```bash
# Listen for Stripe webhooks locally
stripe listen --forward-to localhost:3000/api/webhooks/stripe

# Trigger specific events
stripe trigger checkout.session.completed
stripe trigger invoice.payment_succeeded
```

### Mock webhook payloads for tests

```typescript
// tests/webhooks/stripe.test.ts
import { createHmac } from "crypto";
import { POST } from "@/app/api/webhooks/stripe/route";

function createMockStripeRequest(payload: object) {
  const body = JSON.stringify(payload);
  const timestamp = Math.floor(Date.now() / 1000);
  const secret = process.env.STRIPE_WEBHOOK_SECRET!;

  const signedPayload = `${timestamp}.${body}`;
  const signature = createHmac("sha256", secret).update(signedPayload).digest("hex");

  return new Request("http://localhost:3000/api/webhooks/stripe", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "stripe-signature": `t=${timestamp},v1=${signature}`,
    },
    body,
  });
}

describe("Stripe Webhook", () => {
  it("processes checkout.session.completed", async () => {
    const req = createMockStripeRequest({
      id: "evt_test_123",
      type: "checkout.session.completed",
      data: {
        object: {
          id: "cs_test_123",
          customer: "cus_test_123",
          subscription: "sub_test_123",
          payment_status: "paid",
        },
      },
    });

    const res = await POST(req as any);
    expect(res.status).toBe(200);
  });

  it("rejects invalid signature", async () => {
    const req = new Request("http://localhost:3000/api/webhooks/stripe", {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        "stripe-signature": "t=123,v1=invalid",
      },
      body: JSON.stringify({ type: "test" }),
    });

    const res = await POST(req as any);
    expect(res.status).toBe(400);
  });
});
```

---

## 9. Common Integrations

### Stripe webhook handler (complete)

```typescript
// app/api/webhooks/stripe/route.ts
import Stripe from "stripe";
import { NextRequest, NextResponse } from "next/server";

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!);

const relevantEvents = new Set([
  "checkout.session.completed",
  "customer.subscription.created",
  "customer.subscription.updated",
  "customer.subscription.deleted",
  "invoice.payment_succeeded",
  "invoice.payment_failed",
]);

export async function POST(req: NextRequest) {
  const rawBody = await req.text();
  const signature = req.headers.get("stripe-signature")!;

  let event: Stripe.Event;
  try {
    event = stripe.webhooks.constructEvent(
      rawBody,
      signature,
      process.env.STRIPE_WEBHOOK_SECRET!
    );
  } catch {
    return NextResponse.json({ error: "Invalid signature" }, { status: 400 });
  }

  if (!relevantEvents.has(event.type)) {
    return NextResponse.json({ received: true });
  }

  // Idempotency check
  const existing = await db.stripeEvents.findUnique({ where: { eventId: event.id } });
  if (existing) return NextResponse.json({ received: true });

  await db.stripeEvents.create({
    data: { eventId: event.id, type: event.type, processed: false },
  });

  try {
    switch (event.type) {
      case "checkout.session.completed": {
        const session = event.data.object as Stripe.Checkout.Session;
        await db.users.update({
          where: { stripeCustomerId: session.customer as string },
          data: {
            subscriptionId: session.subscription as string,
            plan: "pro",
            subscriptionStatus: "active",
          },
        });
        break;
      }
      case "customer.subscription.deleted": {
        const sub = event.data.object as Stripe.Subscription;
        await db.users.update({
          where: { stripeCustomerId: sub.customer as string },
          data: { plan: "free", subscriptionStatus: "canceled" },
        });
        break;
      }
      case "invoice.payment_failed": {
        const invoice = event.data.object as Stripe.Invoice;
        await db.users.update({
          where: { stripeCustomerId: invoice.customer as string },
          data: { subscriptionStatus: "past_due" },
        });
        // Send notification
        await sendPaymentFailedEmail(invoice.customer as string);
        break;
      }
    }

    await db.stripeEvents.update({
      where: { eventId: event.id },
      data: { processed: true },
    });
  } catch (error) {
    console.error(`Failed to process Stripe event ${event.id}:`, error);
    return NextResponse.json({ error: "Processing failed" }, { status: 500 });
  }

  return NextResponse.json({ received: true });
}
```

### GitHub webhook handler

```typescript
// app/api/webhooks/github/route.ts
import { createHmac, timingSafeEqual } from "crypto";

export async function POST(req: NextRequest) {
  const rawBody = await req.text();
  const signature = req.headers.get("x-hub-signature-256");
  const event = req.headers.get("x-github-event");
  const deliveryId = req.headers.get("x-github-delivery");

  if (!signature || !event || !deliveryId) {
    return NextResponse.json({ error: "Missing headers" }, { status: 400 });
  }

  const expected = `sha256=${createHmac("sha256", process.env.GITHUB_WEBHOOK_SECRET!)
    .update(rawBody)
    .digest("hex")}`;

  if (!timingSafeEqual(Buffer.from(signature), Buffer.from(expected))) {
    return NextResponse.json({ error: "Invalid signature" }, { status: 401 });
  }

  const payload = JSON.parse(rawBody);

  switch (event) {
    case "push":
      if (payload.ref === "refs/heads/main") {
        await triggerDeployment(payload.repository.full_name);
      }
      break;
    case "issues":
      if (payload.action === "opened") {
        await notifyTeam(`New issue: ${payload.issue.title}`);
      }
      break;
    case "pull_request":
      if (payload.action === "opened") {
        await runCodeReview(payload.pull_request);
      }
      break;
  }

  return NextResponse.json({ ok: true });
}
```

### Supabase Database Webhooks

```typescript
// Supabase sends webhooks on database changes via pg_net or Database Webhooks

// app/api/webhooks/supabase/route.ts
export async function POST(req: NextRequest) {
  // Verify using shared secret
  const authHeader = req.headers.get("authorization");
  if (authHeader !== `Bearer ${process.env.SUPABASE_WEBHOOK_SECRET}`) {
    return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
  }

  const payload = await req.json();

  // Supabase webhook payload structure:
  // { type: "INSERT" | "UPDATE" | "DELETE", table: string, record: object, old_record: object }

  switch (payload.table) {
    case "orders":
      if (payload.type === "INSERT") {
        await sendOrderConfirmation(payload.record);
        revalidateTag("orders");
      }
      break;
    case "users":
      if (payload.type === "UPDATE") {
        await redis.del(`user:${payload.record.id}`);
        revalidateTag(`user-${payload.record.id}`);
      }
      break;
  }

  return NextResponse.json({ ok: true });
}
```

---

## 10. Monitoring

### Webhook delivery dashboard data

```typescript
// app/api/admin/webhooks/stats/route.ts
export async function GET(req: NextRequest) {
  const session = await getAdminSession(req);
  if (!session) return NextResponse.json({ error: "Unauthorized" }, { status: 401 });

  const [total, successful, failed, avgLatency] = await Promise.all([
    db.webhookEvents.count({
      where: { receivedAt: { gte: subDays(new Date(), 7) } },
    }),
    db.webhookEvents.count({
      where: {
        receivedAt: { gte: subDays(new Date(), 7) },
        status: "completed",
      },
    }),
    db.webhookEvents.count({
      where: {
        receivedAt: { gte: subDays(new Date(), 7) },
        status: "failed",
      },
    }),
    db.$queryRaw`
      SELECT AVG(EXTRACT(EPOCH FROM (processed_at - received_at))) as avg_seconds
      FROM webhook_events
      WHERE received_at >= NOW() - INTERVAL '7 days'
      AND status = 'completed'
    `,
  ]);

  const successRate = total > 0 ? (successful / total) * 100 : 0;

  return NextResponse.json({
    period: "7d",
    total,
    successful,
    failed,
    successRate: `${successRate.toFixed(1)}%`,
    avgLatencyMs: Math.round((avgLatency as any)[0]?.avg_seconds * 1000),
  });
}
```

### Structured logging for webhook events

```typescript
// lib/webhooks/logger.ts
interface WebhookLog {
  eventId: string;
  source: string;
  type: string;
  status: "received" | "processing" | "completed" | "failed";
  duration?: number;
  error?: string;
}

function logWebhook(log: WebhookLog) {
  console.log(JSON.stringify({
    ...log,
    timestamp: new Date().toISOString(),
    service: "webhook-handler",
  }));
}

// Usage in webhook handler
export async function POST(req: NextRequest) {
  const start = Date.now();
  const payload = await verifyAndParse(req);

  logWebhook({
    eventId: payload.id,
    source: "stripe",
    type: payload.type,
    status: "received",
  });

  try {
    await processEvent(payload);
    logWebhook({
      eventId: payload.id,
      source: "stripe",
      type: payload.type,
      status: "completed",
      duration: Date.now() - start,
    });
  } catch (error) {
    logWebhook({
      eventId: payload.id,
      source: "stripe",
      type: payload.type,
      status: "failed",
      duration: Date.now() - start,
      error: error instanceof Error ? error.message : "Unknown",
    });
    throw error;
  }

  return NextResponse.json({ received: true });
}
```

### Alerting on failures

```typescript
// lib/webhooks/alerting.ts
const FAILURE_THRESHOLD = 5; // Alert after 5 failures in window
const WINDOW_MINUTES = 15;

async function checkWebhookHealth(source: string) {
  const recentFailures = await db.webhookEvents.count({
    where: {
      source,
      status: "failed",
      receivedAt: { gte: subMinutes(new Date(), WINDOW_MINUTES) },
    },
  });

  if (recentFailures >= FAILURE_THRESHOLD) {
    const alertKey = `alert:webhook:${source}`;
    const alreadyAlerted = await redis.get(alertKey);

    if (!alreadyAlerted) {
      await sendSlackAlert({
        channel: "#engineering-alerts",
        text: `Webhook alert: ${recentFailures} failures from ${source} in last ${WINDOW_MINUTES} minutes`,
        severity: "high",
      });

      // Don't alert again for 30 minutes
      await redis.set(alertKey, "1", { ex: 1800 });
    }
  }
}
```
