# Background Jobs & Task Queues

Reference for async job processing, scheduled tasks, and background work in modern full-stack apps.

---

## 1. Decision Matrix

| Criteria | Inngest | Trigger.dev | BullMQ | Vercel Cron |
|---|---|---|---|---|
| **Environment** | Serverless (Vercel, Netlify, etc.) | Serverless + self-hosted | Self-hosted (Redis required) | Vercel only |
| **Complexity** | Medium-High (step functions, fan-out) | Medium-High (long-running, integrations) | Medium (queues, workers) | Low (simple scheduled endpoints) |
| **Long-running tasks** | Via step functions (each step < timeout) | Native support (hours/days) | Native support | No (serverless timeout applies) |
| **Retry/backoff** | Built-in, configurable | Built-in, configurable | Built-in, configurable | Manual (re-invoke on failure) |
| **Cron/scheduling** | Yes | Yes | Via separate scheduler | Yes (vercel.json) |
| **Dashboard** | Yes (cloud) | Yes (cloud + self-hosted) | Via Bull Board (self-hosted) | Vercel dashboard (basic) |
| **Pricing** | Free tier, then usage-based | Free tier, then usage-based | Free (self-hosted) | Free with Vercel plan |
| **Best for** | Event-driven workflows, serverless | Complex jobs, long-running, integrations | High-throughput, full control | Simple scheduled tasks |

### When to use what

- **Simple cron (daily cleanup, weekly report)** -> Vercel Cron
- **Event-driven workflows (user signs up -> send email -> wait 3 days -> send follow-up)** -> Inngest
- **Long-running tasks (video processing, large data imports)** -> Trigger.dev
- **High-throughput queue (thousands of jobs/sec, self-hosted)** -> BullMQ

---

## 2. Inngest

### Installation & Setup

```bash
bun add inngest
```

```ts
// src/lib/inngest/client.ts
import { Inngest } from "inngest";

export const inngest = new Inngest({
  id: "my-app",
  // Optional: event schemas for type safety
});
```

### Serve endpoint (Next.js App Router)

```ts
// src/app/api/inngest/route.ts
import { serve } from "inngest/next";
import { inngest } from "@/lib/inngest/client";
import { syncDataFunction } from "@/lib/inngest/functions/sync-data";
import { sendWelcomeEmail } from "@/lib/inngest/functions/send-welcome-email";

export const { GET, POST, PUT } = serve({
  client: inngest,
  functions: [syncDataFunction, sendWelcomeEmail],
});
```

### Event-driven function

```ts
// src/lib/inngest/functions/send-welcome-email.ts
import { inngest } from "../client";

export const sendWelcomeEmail = inngest.createFunction(
  {
    id: "send-welcome-email",
    retries: 3,
  },
  { event: "user/created" },
  async ({ event, step }) => {
    // Step 1: Send welcome email
    await step.run("send-email", async () => {
      await sendEmail({
        to: event.data.email,
        template: "welcome",
        data: { name: event.data.name },
      });
    });

    // Step 2: Wait 3 days
    await step.sleep("wait-3-days", "3d");

    // Step 3: Send follow-up
    await step.run("send-followup", async () => {
      await sendEmail({
        to: event.data.email,
        template: "followup",
      });
    });
  }
);
```

### Sending events

```ts
// From anywhere in your app
import { inngest } from "@/lib/inngest/client";

await inngest.send({
  name: "user/created",
  data: {
    userId: user.id,
    email: user.email,
    name: user.name,
  },
});
```

### Cron schedule

```ts
export const dailyCleanup = inngest.createFunction(
  { id: "daily-cleanup" },
  { cron: "0 3 * * *" }, // 3 AM daily
  async ({ step }) => {
    await step.run("cleanup-expired-sessions", async () => {
      await db.session.deleteMany({
        where: { expiresAt: { lt: new Date() } },
      });
    });

    await step.run("cleanup-temp-files", async () => {
      await cleanupTempStorage();
    });
  }
);
```

### Fan-out pattern

```ts
export const processAllUsers = inngest.createFunction(
  { id: "process-all-users" },
  { event: "batch/process-users" },
  async ({ step }) => {
    const users = await step.run("fetch-users", async () => {
      return await db.user.findMany({ where: { active: true } });
    });

    // Fan out: send an event per user (processed in parallel)
    const events = users.map((user) => ({
      name: "user/process-single" as const,
      data: { userId: user.id },
    }));

    await step.sendEvent("fan-out", events);
  }
);
```

### Step function patterns

```ts
// Parallel steps
const [result1, result2] = await Promise.all([
  step.run("task-a", async () => fetchA()),
  step.run("task-b", async () => fetchB()),
]);

// Wait for external event
const approval = await step.waitForEvent("wait-for-approval", {
  event: "order/approved",
  match: "data.orderId",
  timeout: "7d",
});

if (!approval) {
  // Timed out — no approval received
  await step.run("notify-timeout", async () => {
    await notifyAdmin("Order approval timed out");
  });
}
```

---

## 3. Trigger.dev

### Installation & Setup

```bash
bun add @trigger.dev/sdk
npx trigger.dev@latest init
```

```ts
// src/trigger/client.ts
import { TriggerClient } from "@trigger.dev/sdk";

export const client = new TriggerClient({
  id: "my-app",
  apiKey: process.env.TRIGGER_API_KEY!,
  apiUrl: process.env.TRIGGER_API_URL, // optional for self-hosted
});
```

### Task definition (v3)

```ts
// src/trigger/tasks/generate-report.ts
import { task } from "@trigger.dev/sdk/v3";

export const generateReport = task({
  id: "generate-report",
  // Long-running: up to 5 minutes
  maxDuration: 300,
  retry: {
    maxAttempts: 3,
    factor: 2,
    minTimeoutInMs: 1000,
    maxTimeoutInMs: 30000,
  },
  run: async (payload: { reportId: string; userId: string }) => {
    const data = await fetchReportData(payload.reportId);
    const pdf = await generatePDF(data);
    const url = await uploadToStorage(pdf);

    await db.report.update({
      where: { id: payload.reportId },
      data: { status: "completed", url },
    });

    return { url };
  },
});
```

### Triggering tasks

```ts
import { generateReport } from "@/trigger/tasks/generate-report";

// Trigger and forget
await generateReport.trigger({
  reportId: "rpt_123",
  userId: "usr_456",
});

// Trigger and wait for result
const result = await generateReport.triggerAndWait({
  reportId: "rpt_123",
  userId: "usr_456",
});
console.log(result.url);

// Batch trigger
await generateReport.batchTrigger(
  reportIds.map((id) => ({ payload: { reportId: id, userId } }))
);
```

### Scheduled tasks (cron)

```ts
import { schedules } from "@trigger.dev/sdk/v3";

export const weeklyDigest = schedules.task({
  id: "weekly-digest",
  cron: "0 9 * * 1", // Monday 9 AM
  run: async () => {
    const users = await db.user.findMany({ where: { digestEnabled: true } });
    for (const user of users) {
      await sendDigestEmail(user);
    }
  },
});
```

---

## 4. Vercel Cron

### Configuration

```json
// vercel.json
{
  "crons": [
    {
      "path": "/api/cron/cleanup",
      "schedule": "0 3 * * *"
    },
    {
      "path": "/api/cron/weekly-report",
      "schedule": "0 9 * * 1"
    }
  ]
}
```

### Route handler as cron endpoint

```ts
// src/app/api/cron/cleanup/route.ts
import { NextResponse } from "next/server";

export async function GET(request: Request) {
  // Verify the request is from Vercel Cron
  const authHeader = request.headers.get("authorization");
  if (authHeader !== `Bearer ${process.env.CRON_SECRET}`) {
    return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
  }

  try {
    const deleted = await db.session.deleteMany({
      where: { expiresAt: { lt: new Date() } },
    });

    return NextResponse.json({
      success: true,
      deletedSessions: deleted.count,
    });
  } catch (error) {
    console.error("Cron cleanup failed:", error);
    return NextResponse.json(
      { error: "Cleanup failed" },
      { status: 500 }
    );
  }
}
```

### Limitations

- **Hobby plan**: 2 cron jobs, once per day minimum
- **Pro plan**: 40 cron jobs, once per minute minimum
- **Execution timeout**: Same as serverless function timeout (10s hobby, 60s pro, 300s enterprise)
- **No retry**: If the endpoint fails, it won't retry until the next scheduled run
- **No payload**: Cannot pass dynamic data to cron endpoints
- **No chaining**: Cannot trigger one cron from another natively

---

## 5. Common Patterns

### Email sending queue

```ts
// Inngest example
export const sendEmailJob = inngest.createFunction(
  {
    id: "send-email",
    retries: 5,
    concurrency: {
      limit: 10, // Max 10 concurrent email sends
    },
  },
  { event: "email/send" },
  async ({ event, step }) => {
    const { to, subject, template, data } = event.data;

    await step.run("send", async () => {
      await resend.emails.send({
        from: "noreply@example.com",
        to,
        subject,
        react: getEmailTemplate(template, data),
      });
    });

    await step.run("log", async () => {
      await db.emailLog.create({
        data: { to, subject, template, sentAt: new Date() },
      });
    });
  }
);
```

### Image processing

```ts
export const processImage = inngest.createFunction(
  {
    id: "process-image",
    retries: 3,
    concurrency: { limit: 5 },
  },
  { event: "image/uploaded" },
  async ({ event, step }) => {
    const { imageId, url } = event.data;

    const optimized = await step.run("optimize", async () => {
      const buffer = await fetch(url).then((r) => r.arrayBuffer());
      return await sharp(Buffer.from(buffer))
        .resize(1200, 1200, { fit: "inside" })
        .webp({ quality: 80 })
        .toBuffer();
    });

    const uploadedUrl = await step.run("upload", async () => {
      return await uploadToR2(optimized, `images/${imageId}.webp`);
    });

    await step.run("update-db", async () => {
      await db.image.update({
        where: { id: imageId },
        data: { processedUrl: uploadedUrl, status: "ready" },
      });
    });
  }
);
```

### Report generation

```ts
export const generateMonthlyReport = inngest.createFunction(
  { id: "monthly-report" },
  { cron: "0 6 1 * *" }, // 1st of month at 6 AM
  async ({ step }) => {
    const orgs = await step.run("fetch-orgs", () =>
      db.organization.findMany({ where: { active: true } })
    );

    // Fan out per org
    await step.sendEvent(
      "fan-out-reports",
      orgs.map((org) => ({
        name: "report/generate-single" as const,
        data: { orgId: org.id, month: new Date().getMonth() },
      }))
    );
  }
);
```

### Data sync

```ts
export const syncExternalData = inngest.createFunction(
  {
    id: "sync-external-data",
    retries: 3,
    concurrency: { limit: 2 }, // API rate limit friendly
  },
  { cron: "*/15 * * * *" }, // Every 15 minutes
  async ({ step }) => {
    const lastSync = await step.run("get-last-sync", () =>
      db.syncState.findFirst({ orderBy: { createdAt: "desc" } })
    );

    const newData = await step.run("fetch-external", () =>
      externalApi.getUpdatedSince(lastSync?.timestamp ?? new Date(0))
    );

    if (newData.length > 0) {
      await step.run("upsert-data", () =>
        db.$transaction(
          newData.map((item) =>
            db.record.upsert({
              where: { externalId: item.id },
              create: mapToLocal(item),
              update: mapToLocal(item),
            })
          )
        )
      );
    }

    await step.run("update-sync-state", () =>
      db.syncState.create({ data: { timestamp: new Date(), count: newData.length } })
    );
  }
);
```

### Cleanup jobs

```ts
export const cleanupJob = inngest.createFunction(
  { id: "cleanup" },
  { cron: "0 2 * * *" }, // 2 AM daily
  async ({ step }) => {
    // Delete expired sessions
    await step.run("expired-sessions", () =>
      db.session.deleteMany({ where: { expiresAt: { lt: new Date() } } })
    );

    // Delete unverified accounts older than 7 days
    await step.run("unverified-accounts", () =>
      db.user.deleteMany({
        where: {
          emailVerified: false,
          createdAt: { lt: new Date(Date.now() - 7 * 24 * 60 * 60 * 1000) },
        },
      })
    );

    // Clean temp storage
    await step.run("temp-storage", () => cleanTempFiles("7d"));
  }
);
```

---

## 6. Retry Strategies

### Exponential backoff (Inngest)

```ts
inngest.createFunction(
  {
    id: "resilient-task",
    retries: 5, // Will retry up to 5 times
    // Default: exponential backoff starting at ~5s
  },
  { event: "task/run" },
  async ({ event, step, attempt }) => {
    console.log(`Attempt ${attempt}`); // 0-indexed
    await step.run("api-call", async () => {
      const res = await fetch("https://api.example.com/data");
      if (!res.ok) throw new Error(`API returned ${res.status}`);
      return res.json();
    });
  }
);
```

### Custom retry (Trigger.dev)

```ts
export const resilientTask = task({
  id: "resilient-task",
  retry: {
    maxAttempts: 5,
    factor: 2,            // Exponential factor
    minTimeoutInMs: 1000,  // Start at 1s
    maxTimeoutInMs: 60000, // Cap at 60s
    randomize: true,       // Add jitter
  },
  run: async (payload) => {
    // Task logic
  },
});
```

### Non-retryable errors

```ts
import { NonRetriableError } from "inngest";

inngest.createFunction(
  { id: "process-payment", retries: 3 },
  { event: "payment/process" },
  async ({ event, step }) => {
    await step.run("charge", async () => {
      try {
        await stripe.charges.create({ /* ... */ });
      } catch (error) {
        if (error.code === "card_declined") {
          // Don't retry — card is declined
          throw new NonRetriableError("Card declined", { cause: error });
        }
        // Other errors will be retried
        throw error;
      }
    });
  }
);
```

### Dead letter / failure handling

```ts
inngest.createFunction(
  {
    id: "important-task",
    retries: 5,
    onFailure: async ({ error, event, step }) => {
      // All retries exhausted — handle permanent failure
      await step.run("notify-team", async () => {
        await slack.send({
          channel: "#alerts",
          text: `Job ${event.data.jobId} permanently failed: ${error.message}`,
        });
      });

      await step.run("update-status", async () => {
        await db.job.update({
          where: { id: event.data.jobId },
          data: { status: "failed", error: error.message },
        });
      });
    },
  },
  { event: "task/important" },
  async ({ event, step }) => {
    // Main task logic
  }
);
```

### Idempotent jobs

```ts
inngest.createFunction(
  { id: "process-order" },
  { event: "order/created" },
  async ({ event, step }) => {
    // Use idempotency key to prevent double processing
    const idempotencyKey = `order-${event.data.orderId}`;

    await step.run("check-processed", async () => {
      const existing = await db.processedJob.findUnique({
        where: { key: idempotencyKey },
      });
      if (existing) {
        // Already processed — skip silently
        return;
      }
    });

    await step.run("process", async () => {
      await processOrder(event.data.orderId);
    });

    await step.run("mark-processed", async () => {
      await db.processedJob.create({
        data: { key: idempotencyKey, processedAt: new Date() },
      });
    });
  }
);
```

---

## 7. Job Monitoring

### Inngest dashboard

Inngest provides a cloud dashboard at `app.inngest.com` showing:
- Function runs (success/failure/in-progress)
- Event history
- Step-by-step execution timeline
- Error details and stack traces
- Cron schedule overview

### Failed job alerts

```ts
// Inngest onFailure handler -> Slack alert
onFailure: async ({ error, event, step }) => {
  await step.run("alert", async () => {
    await fetch(process.env.SLACK_WEBHOOK_URL!, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        text: `Job failed: ${event.name}\nError: ${error.message}\nEvent ID: ${event.id}`,
      }),
    });
  });
};
```

### Custom monitoring with Supabase

```ts
// Log job execution to Supabase for custom dashboards
async function logJobExecution(job: {
  functionId: string;
  status: "started" | "completed" | "failed";
  duration?: number;
  error?: string;
}) {
  await supabase.from("job_executions").insert({
    function_id: job.functionId,
    status: job.status,
    duration_ms: job.duration,
    error_message: job.error,
    executed_at: new Date().toISOString(),
  });
}
```

---

## 8. Rate Limiting

### Job concurrency limits (Inngest)

```ts
inngest.createFunction(
  {
    id: "api-sync",
    concurrency: {
      limit: 5, // Max 5 concurrent runs
    },
    // OR: per-key concurrency
    // concurrency: {
    //   limit: 1,
    //   key: "event.data.userId", // 1 concurrent per user
    // },
  },
  { event: "sync/start" },
  async ({ event, step }) => {
    // Only 5 of these will run at the same time
  }
);
```

### Rate limiting with Inngest

```ts
inngest.createFunction(
  {
    id: "external-api-call",
    rateLimit: {
      limit: 100,
      period: "1m", // 100 calls per minute
      key: "event.data.apiKey", // Per API key
    },
  },
  { event: "api/call" },
  async ({ event, step }) => {
    await step.run("call-api", async () => {
      return await externalApi.call(event.data);
    });
  }
);
```

### Throttling pattern

```ts
inngest.createFunction(
  {
    id: "bulk-email",
    throttle: {
      limit: 50,
      period: "1s", // Max 50 per second
      burst: 10,    // Allow burst of 10 extra
    },
  },
  { event: "email/bulk-send" },
  async ({ event, step }) => {
    await step.run("send", async () => {
      await resend.emails.send({ /* ... */ });
    });
  }
);
```

---

## 9. Long-Running Tasks

### Progress tracking

```ts
export const importData = inngest.createFunction(
  { id: "import-data" },
  { event: "data/import" },
  async ({ event, step }) => {
    const { importId, fileUrl } = event.data;

    const rows = await step.run("parse-file", async () => {
      const file = await fetch(fileUrl).then((r) => r.text());
      return parseCSV(file);
    });

    const totalRows = rows.length;
    const CHUNK_SIZE = 100;

    for (let i = 0; i < totalRows; i += CHUNK_SIZE) {
      const chunk = rows.slice(i, i + CHUNK_SIZE);

      await step.run(`process-chunk-${i}`, async () => {
        await db.record.createMany({ data: chunk.map(mapRow) });

        // Update progress
        await db.import.update({
          where: { id: importId },
          data: {
            processedRows: Math.min(i + CHUNK_SIZE, totalRows),
            totalRows,
            progress: Math.round(((i + CHUNK_SIZE) / totalRows) * 100),
          },
        });
      });
    }

    await step.run("finalize", async () => {
      await db.import.update({
        where: { id: importId },
        data: { status: "completed", completedAt: new Date() },
      });
    });
  }
);
```

### Cancellation

```ts
// Inngest: cancel via event
inngest.createFunction(
  {
    id: "long-process",
    cancelOn: [
      {
        event: "process/cancel",
        match: "data.processId", // Cancel when matching event received
      },
    ],
  },
  { event: "process/start" },
  async ({ event, step }) => {
    // If "process/cancel" with same processId is sent, this function stops
    for (const chunk of chunks) {
      await step.run(`process-${chunk.id}`, () => processChunk(chunk));
    }
  }
);

// Send cancel event
await inngest.send({
  name: "process/cancel",
  data: { processId: "proc_123" },
});
```

### Timeout handling

```ts
// Trigger.dev: set max duration
export const longTask = task({
  id: "long-task",
  maxDuration: 600, // 10 minutes max
  run: async (payload) => {
    const startTime = Date.now();
    const TIMEOUT = 9 * 60 * 1000; // 9 min safety margin

    for (const item of payload.items) {
      if (Date.now() - startTime > TIMEOUT) {
        // Save progress and re-trigger with remaining items
        await longTask.trigger({
          ...payload,
          items: payload.items.slice(payload.items.indexOf(item)),
        });
        return { status: "continued", processedUntil: item.id };
      }

      await processItem(item);
    }

    return { status: "completed" };
  },
});
```

---

## 10. Testing

### Testing job handlers in isolation

```ts
// src/lib/inngest/functions/__tests__/send-welcome-email.test.ts
import { describe, it, expect, vi } from "vitest";

// Extract the handler logic into a pure function
import { handleSendWelcomeEmail } from "../send-welcome-email";

describe("sendWelcomeEmail handler", () => {
  it("sends email with correct template", async () => {
    const mockSendEmail = vi.fn().mockResolvedValue({ id: "msg_123" });

    await handleSendWelcomeEmail({
      email: "test@example.com",
      name: "Test User",
      sendEmail: mockSendEmail,
    });

    expect(mockSendEmail).toHaveBeenCalledWith({
      to: "test@example.com",
      template: "welcome",
      data: { name: "Test User" },
    });
  });
});
```

### Separation pattern for testability

```ts
// src/lib/inngest/functions/send-welcome-email.ts

// Pure handler — easy to test
export async function handleSendWelcomeEmail(params: {
  email: string;
  name: string;
  sendEmail?: typeof defaultSendEmail;
}) {
  const send = params.sendEmail ?? defaultSendEmail;
  await send({
    to: params.email,
    template: "welcome",
    data: { name: params.name },
  });
}

// Inngest wrapper — thin glue
export const sendWelcomeEmail = inngest.createFunction(
  { id: "send-welcome-email", retries: 3 },
  { event: "user/created" },
  async ({ event, step }) => {
    await step.run("send-email", () =>
      handleSendWelcomeEmail({
        email: event.data.email,
        name: event.data.name,
      })
    );
  }
);
```

### Mocking Inngest step functions

```ts
import { describe, it, expect, vi } from "vitest";

// Mock step object
function createMockStep() {
  return {
    run: vi.fn((_name: string, fn: () => any) => fn()),
    sleep: vi.fn().mockResolvedValue(undefined),
    sendEvent: vi.fn().mockResolvedValue(undefined),
    waitForEvent: vi.fn().mockResolvedValue(null),
  };
}

describe("complex workflow", () => {
  it("processes order and sends notification", async () => {
    const step = createMockStep();
    const event = {
      data: { orderId: "ord_123", userId: "usr_456" },
    };

    await orderWorkflowHandler({ event, step } as any);

    expect(step.run).toHaveBeenCalledTimes(3);
    expect(step.sendEvent).toHaveBeenCalledWith(
      "notify",
      expect.objectContaining({ name: "notification/send" })
    );
  });
});
```

### Integration tests with Inngest Dev Server

```ts
// Start Inngest dev server: npx inngest-cli@latest dev
// Then run integration tests against it

import { describe, it, expect } from "vitest";
import { inngest } from "@/lib/inngest/client";

describe("integration: email workflow", () => {
  it("processes user/created event end-to-end", async () => {
    // Send event to dev server
    await inngest.send({
      name: "user/created",
      data: {
        userId: "test_123",
        email: "integration@test.com",
        name: "Integration Test",
      },
    });

    // Poll for result (the dev server processes it)
    await new Promise((resolve) => setTimeout(resolve, 2000));

    // Verify side effects
    const emailLog = await db.emailLog.findFirst({
      where: { to: "integration@test.com" },
    });
    expect(emailLog).toBeDefined();
    expect(emailLog?.template).toBe("welcome");
  });
});
```
