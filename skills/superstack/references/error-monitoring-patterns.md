# Error Monitoring Patterns

## 1. Custom Error Classes

Base error class hierarchy for consistent error handling across the stack.

```typescript
// lib/errors.ts

export enum ErrorCode {
  VALIDATION_ERROR = 'VALIDATION_ERROR',
  AUTH_ERROR = 'AUTH_ERROR',
  NOT_FOUND = 'NOT_FOUND',
  FORBIDDEN = 'FORBIDDEN',
  RATE_LIMITED = 'RATE_LIMITED',
  INTERNAL_ERROR = 'INTERNAL_ERROR',
  EXTERNAL_SERVICE_ERROR = 'EXTERNAL_SERVICE_ERROR',
  CONFLICT = 'CONFLICT',
}

export class AppError extends Error {
  public readonly code: ErrorCode;
  public readonly statusCode: number;
  public readonly isOperational: boolean;
  public readonly userMessage: string;
  public readonly context?: Record<string, unknown>;

  constructor(params: {
    message: string;
    code: ErrorCode;
    statusCode: number;
    isOperational?: boolean;
    userMessage?: string;
    context?: Record<string, unknown>;
    cause?: Error;
  }) {
    super(params.message, { cause: params.cause });
    this.name = this.constructor.name;
    this.code = params.code;
    this.statusCode = params.statusCode;
    this.isOperational = params.isOperational ?? true;
    this.userMessage = params.userMessage ?? 'An unexpected error occurred.';
    this.context = params.context;
    Error.captureStackTrace(this, this.constructor);
  }

  toJSON() {
    return {
      error: {
        code: this.code,
        message: this.userMessage,
        ...(process.env.NODE_ENV === 'development' && {
          devMessage: this.message,
          stack: this.stack,
          context: this.context,
        }),
      },
    };
  }
}

export class HttpError extends AppError {
  constructor(statusCode: number, message: string, userMessage?: string) {
    super({
      message,
      code: ErrorCode.INTERNAL_ERROR,
      statusCode,
      userMessage,
    });
  }
}

export class ValidationError extends AppError {
  public readonly fields: Record<string, string[]>;

  constructor(fields: Record<string, string[]>, message?: string) {
    super({
      message: message ?? `Validation failed: ${Object.keys(fields).join(', ')}`,
      code: ErrorCode.VALIDATION_ERROR,
      statusCode: 400,
      userMessage: 'Please check your input and try again.',
      context: { fields },
    });
    this.fields = fields;
  }

  toJSON() {
    return {
      error: {
        code: this.code,
        message: this.userMessage,
        fields: this.fields,
      },
    };
  }
}

export class AuthError extends AppError {
  constructor(message = 'Authentication required') {
    super({
      message,
      code: ErrorCode.AUTH_ERROR,
      statusCode: 401,
      userMessage: 'Please sign in to continue.',
    });
  }
}

export class ForbiddenError extends AppError {
  constructor(message = 'Insufficient permissions') {
    super({
      message,
      code: ErrorCode.FORBIDDEN,
      statusCode: 403,
      userMessage: 'You do not have permission to perform this action.',
    });
  }
}

export class NotFoundError extends AppError {
  constructor(resource: string, id?: string) {
    super({
      message: `${resource} not found${id ? `: ${id}` : ''}`,
      code: ErrorCode.NOT_FOUND,
      statusCode: 404,
      userMessage: `The requested ${resource.toLowerCase()} was not found.`,
      context: { resource, id },
    });
  }
}

export class RateLimitError extends AppError {
  constructor(retryAfter?: number) {
    super({
      message: 'Rate limit exceeded',
      code: ErrorCode.RATE_LIMITED,
      statusCode: 429,
      userMessage: 'Too many requests. Please try again later.',
      context: { retryAfter },
    });
  }
}

export class ExternalServiceError extends AppError {
  constructor(service: string, cause?: Error) {
    super({
      message: `External service error: ${service}`,
      code: ErrorCode.EXTERNAL_SERVICE_ERROR,
      statusCode: 502,
      userMessage: 'A third-party service is temporarily unavailable.',
      context: { service },
      cause,
    });
  }
}
```

---

## 2. User-Facing vs Dev Errors

Never leak internal details to users. Every error has two messages.

```typescript
// The AppError base class handles this via `message` (dev) and `userMessage` (user-facing).
// Serialization strips dev info in production.

// Usage in API routes:
export function serializeError(error: unknown): { status: number; body: object } {
  if (error instanceof AppError) {
    return {
      status: error.statusCode,
      body: error.toJSON(),
    };
  }

  // Unknown errors — never expose internals
  console.error('Unhandled error:', error);
  return {
    status: 500,
    body: {
      error: {
        code: 'INTERNAL_ERROR',
        message: 'An unexpected error occurred.',
      },
    },
  };
}

// In logging, always include full detail:
function logError(error: unknown, requestContext?: Record<string, unknown>) {
  if (error instanceof AppError) {
    logger.error({
      err: error,
      code: error.code,
      statusCode: error.statusCode,
      isOperational: error.isOperational,
      context: { ...error.context, ...requestContext },
    }, error.message);
  } else {
    logger.error({ err: error, ...requestContext }, 'Unhandled error');
  }
}
```

**Rules:**
- `userMessage` — safe to show in UI, generic, no stack traces
- `message` — for logs/Sentry, includes technical detail
- `context` — structured metadata for debugging (tenant ID, user ID, request params)
- `isOperational` — `true` = expected errors (validation, auth), `false` = bugs (crash, restart)

---

## 3. Sentry Setup

### Installation

```bash
npx @sentry/wizard@latest -i nextjs
```

### Configuration

```typescript
// sentry.client.config.ts
import * as Sentry from '@sentry/nextjs';

Sentry.init({
  dsn: process.env.NEXT_PUBLIC_SENTRY_DSN,
  environment: process.env.NEXT_PUBLIC_VERCEL_ENV ?? 'development',
  release: process.env.NEXT_PUBLIC_VERCEL_GIT_COMMIT_SHA,

  tracesSampleRate: process.env.NODE_ENV === 'production' ? 0.1 : 1.0,
  replaysSessionSampleRate: 0.01,
  replaysOnErrorSampleRate: 1.0,

  integrations: [
    Sentry.replayIntegration({
      maskAllText: true,
      blockAllMedia: true,
    }),
  ],

  beforeSend(event) {
    // Scrub PII
    if (event.request?.cookies) {
      delete event.request.cookies;
    }
    return event;
  },

  ignoreErrors: [
    'ResizeObserver loop',
    'Network request failed',
    'Load failed',
    /^AbortError/,
  ],
});
```

```typescript
// sentry.server.config.ts
import * as Sentry from '@sentry/nextjs';

Sentry.init({
  dsn: process.env.SENTRY_DSN,
  environment: process.env.VERCEL_ENV ?? 'development',
  release: process.env.VERCEL_GIT_COMMIT_SHA,
  tracesSampleRate: 0.1,

  beforeSend(event) {
    // Add custom tags
    if (event.contexts?.app) {
      event.tags = {
        ...event.tags,
        service: 'api',
      };
    }
    return event;
  },
});
```

### Tunnel Route (Bypass Ad Blockers)

```typescript
// app/api/monitoring/route.ts
import { NextRequest } from 'next/server';

// Tunnel Sentry requests through your own domain to bypass ad blockers
export async function POST(request: NextRequest) {
  const envelope = await request.text();
  const pieces = envelope.split('\n');

  const header = JSON.parse(pieces[0]);
  const dsn = new URL(header.dsn);
  const projectId = dsn.pathname.replace('/', '');

  const sentryUrl = `https://${dsn.hostname}/api/${projectId}/envelope/`;

  const response = await fetch(sentryUrl, {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-sentry-envelope' },
    body: envelope,
  });

  return new Response(response.body, { status: response.status });
}
```

```typescript
// sentry.client.config.ts — add tunnel option
Sentry.init({
  dsn: process.env.NEXT_PUBLIC_SENTRY_DSN,
  tunnel: '/api/monitoring', // Route through your domain
  // ...rest of config
});
```

### Source Maps (next.config.ts)

```typescript
// next.config.ts
import { withSentryConfig } from '@sentry/nextjs';

const nextConfig = {
  // your config
};

export default withSentryConfig(nextConfig, {
  org: 'your-org',
  project: 'your-project',
  authToken: process.env.SENTRY_AUTH_TOKEN,
  silent: true,
  hideSourceMaps: true, // Don't expose source maps publicly
  widenClientFileUpload: true,
});
```

---

## 4. Error Boundaries

```typescript
// components/error-boundary.tsx
'use client';

import * as Sentry from '@sentry/nextjs';
import { Component, type ErrorInfo, type ReactNode } from 'react';
import { Button } from '@/components/ui/button';

interface Props {
  children: ReactNode;
  fallback?: ReactNode;
  onError?: (error: Error, errorInfo: ErrorInfo) => void;
  level?: 'page' | 'section' | 'widget';
}

interface State {
  hasError: boolean;
  error: Error | null;
}

export class ErrorBoundary extends Component<Props, State> {
  state: State = { hasError: false, error: null };

  static getDerivedStateFromError(error: Error): State {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, errorInfo: ErrorInfo) {
    Sentry.withScope((scope) => {
      scope.setTag('errorBoundary', this.props.level ?? 'unknown');
      scope.setContext('componentStack', {
        stack: errorInfo.componentStack,
      });
      Sentry.captureException(error);
    });

    this.props.onError?.(error, errorInfo);
  }

  handleReset = () => {
    this.setState({ hasError: false, error: null });
  };

  render() {
    if (this.state.hasError) {
      if (this.props.fallback) return this.props.fallback;

      return (
        <div className="flex flex-col items-center justify-center gap-4 p-8">
          <h2 className="text-lg font-semibold">Something went wrong</h2>
          <p className="text-sm text-muted-foreground">
            {this.props.level === 'page'
              ? 'This page encountered an error. Please try refreshing.'
              : 'This section encountered an error.'}
          </p>
          <div className="flex gap-2">
            <Button variant="outline" onClick={this.handleReset}>
              Try Again
            </Button>
            {this.props.level === 'page' && (
              <Button onClick={() => window.location.reload()}>
                Refresh Page
              </Button>
            )}
          </div>
        </div>
      );
    }

    return this.props.children;
  }
}
```

```typescript
// app/error.tsx — Next.js App Router error boundary
'use client';

import * as Sentry from '@sentry/nextjs';
import { useEffect } from 'react';
import { Button } from '@/components/ui/button';

export default function Error({
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
    <div className="flex min-h-[50vh] flex-col items-center justify-center gap-4">
      <h2 className="text-xl font-semibold">Something went wrong</h2>
      <p className="text-muted-foreground">
        An error occurred while loading this page.
      </p>
      <Button onClick={reset}>Try Again</Button>
    </div>
  );
}
```

**Boundary placement strategy:**
- Page-level boundary (`app/error.tsx`) — catches route-level crashes
- Section-level — wrap independent dashboard sections so one crash doesn't take down the whole page
- Widget-level — wrap third-party embeds, charts, or experimental features

---

## 5. Alerting Rules

### Sentry Alert Configuration

```yaml
# Recommended Sentry alert rules

# 1. New issue alert — immediate notification
- name: "New Production Error"
  conditions:
    - type: first_seen_event
  filters:
    - environment: production
  actions:
    - type: slack
      channel: "#alerts-errors"
    - type: email
      targetType: team

# 2. Spike alert — sudden increase
- name: "Error Spike"
  conditions:
    - type: event_frequency
      value: 100
      interval: 1h
  actions:
    - type: slack
      channel: "#alerts-critical"
    - type: pagerduty

# 3. Regression alert — previously resolved errors
- name: "Error Regression"
  conditions:
    - type: regression_event
  actions:
    - type: slack
      channel: "#alerts-errors"

# 4. Performance degradation
- name: "Slow Transactions"
  conditions:
    - type: transaction_duration
      value: 5000  # ms
      percentile: p95
  actions:
    - type: slack
      channel: "#alerts-performance"
```

### Alert Fatigue Prevention

- **Group related errors** — use Sentry fingerprinting to collapse duplicates
- **Set thresholds** — don't alert on single occurrences of non-critical errors
- **Use digest mode** — batch low-priority alerts into daily/weekly summaries
- **Severity tiers:**
  - P0 (Critical): PagerDuty + Slack — auth broken, data loss, full outage
  - P1 (High): Slack immediate — feature broken for many users
  - P2 (Medium): Slack batched — edge cases, degraded experience
  - P3 (Low): Weekly review — cosmetic, rare

```typescript
// Custom fingerprinting in Sentry
Sentry.init({
  beforeSend(event) {
    // Group all validation errors together
    if (event.tags?.errorCode === 'VALIDATION_ERROR') {
      event.fingerprint = ['validation-error', event.tags.endpoint ?? 'unknown'];
    }
    return event;
  },
});
```

---

## 6. Error Budgets

### Concepts

- **SLI (Service Level Indicator):** measurable metric (e.g., error rate, latency p99)
- **SLO (Service Level Objective):** target for the SLI (e.g., 99.9% success rate)
- **Error Budget:** the allowed failure (e.g., 0.1% = ~43 minutes downtime/month)

### Practical Implementation

```typescript
// lib/slo.ts
interface SLOConfig {
  name: string;
  target: number; // e.g., 0.999
  window: 'rolling-7d' | 'rolling-30d' | 'calendar-month';
}

const SLOs: SLOConfig[] = [
  { name: 'api-availability', target: 0.999, window: 'rolling-30d' },
  { name: 'api-latency-p99', target: 0.99, window: 'rolling-7d' },
  { name: 'checkout-success', target: 0.995, window: 'calendar-month' },
];

// Track in your monitoring system
function calculateErrorBudget(
  totalRequests: number,
  failedRequests: number,
  target: number
): {
  budgetTotal: number;
  budgetConsumed: number;
  budgetRemaining: number;
  percentConsumed: number;
} {
  const budgetTotal = totalRequests * (1 - target);
  const budgetConsumed = failedRequests;
  const budgetRemaining = Math.max(0, budgetTotal - budgetConsumed);
  const percentConsumed = budgetTotal > 0 ? (budgetConsumed / budgetTotal) * 100 : 100;

  return { budgetTotal, budgetConsumed, budgetRemaining, percentConsumed };
}
```

### Decision Framework

| Budget Consumed | Action |
|---|---|
| 0-50% | Ship features freely |
| 50-75% | Increase review rigor, add feature flags |
| 75-90% | Freeze non-critical features, focus on reliability |
| 90-100% | Full feature freeze, all hands on stability |
| >100% | SLO violated — postmortem, remediation plan |

---

## 7. Structured Logging

### Pino Setup

```typescript
// lib/logger.ts
import pino from 'pino';
import { randomUUID } from 'crypto';

export const logger = pino({
  level: process.env.LOG_LEVEL ?? 'info',
  formatters: {
    level(label) {
      return { level: label };
    },
  },
  timestamp: pino.stdTimeFunctions.isoTime,
  redact: {
    paths: ['req.headers.authorization', 'req.headers.cookie', '*.password', '*.token'],
    censor: '[REDACTED]',
  },
  ...(process.env.NODE_ENV === 'development' && {
    transport: {
      target: 'pino-pretty',
      options: {
        colorize: true,
        translateTime: 'HH:MM:ss',
        ignore: 'pid,hostname',
      },
    },
  }),
});

// Create child logger with request context
export function createRequestLogger(req?: {
  method?: string;
  url?: string;
  headers?: Record<string, string>;
}) {
  const correlationId =
    req?.headers?.['x-correlation-id'] ?? randomUUID();

  return logger.child({
    correlationId,
    method: req?.method,
    url: req?.url,
  });
}

// Log levels:
// trace — ultra-verbose debug info (never in prod)
// debug — development debugging
// info  — normal operations (request served, job completed)
// warn  — recoverable issues (rate limit approaching, deprecated usage)
// error — failures requiring attention
// fatal — app crash, immediate action required
```

### Correlation IDs in Middleware

```typescript
// middleware.ts
import { NextRequest, NextResponse } from 'next/server';
import { randomUUID } from 'crypto';

export function middleware(request: NextRequest) {
  const correlationId =
    request.headers.get('x-correlation-id') ?? randomUUID();

  const requestHeaders = new Headers(request.headers);
  requestHeaders.set('x-correlation-id', correlationId);

  const response = NextResponse.next({
    request: { headers: requestHeaders },
  });
  response.headers.set('x-correlation-id', correlationId);

  return response;
}
```

### Logging Patterns

```typescript
// DO: structured, actionable
logger.info({ userId, action: 'subscription_created', planId }, 'User subscribed');
logger.error({ err, orderId, paymentProvider: 'stripe' }, 'Payment processing failed');

// DON'T: unstructured, missing context
logger.info('User subscribed to plan');
logger.error('Payment failed: ' + error.message);
```

---

## 8. API Error Handling

### Consistent Error Response Format

```typescript
// All API errors follow this shape:
interface ApiErrorResponse {
  error: {
    code: string;        // Machine-readable: 'VALIDATION_ERROR'
    message: string;     // User-friendly message
    fields?: Record<string, string[]>;  // Validation errors per field
    requestId?: string;  // For support tickets
  };
}

// Success responses:
interface ApiSuccessResponse<T> {
  data: T;
  meta?: {
    page?: number;
    totalPages?: number;
    totalCount?: number;
  };
}
```

### Error Handler Middleware

```typescript
// lib/api/error-handler.ts
import { NextRequest, NextResponse } from 'next/server';
import * as Sentry from '@sentry/nextjs';
import { AppError, ErrorCode } from '@/lib/errors';
import { createRequestLogger } from '@/lib/logger';
import { ZodError } from 'zod';

type RouteHandler = (
  req: NextRequest,
  context?: { params: Record<string, string> }
) => Promise<NextResponse>;

export function withErrorHandler(handler: RouteHandler): RouteHandler {
  return async (req, context) => {
    const log = createRequestLogger({
      method: req.method,
      url: req.url,
      headers: Object.fromEntries(req.headers),
    });

    try {
      return await handler(req, context);
    } catch (error) {
      // Zod validation errors
      if (error instanceof ZodError) {
        const fields: Record<string, string[]> = {};
        for (const issue of error.issues) {
          const path = issue.path.join('.');
          fields[path] = fields[path] ?? [];
          fields[path].push(issue.message);
        }
        return NextResponse.json(
          {
            error: {
              code: ErrorCode.VALIDATION_ERROR,
              message: 'Validation failed.',
              fields,
            },
          },
          { status: 400 }
        );
      }

      // Known application errors
      if (error instanceof AppError) {
        log.warn({ err: error, code: error.code }, error.message);

        if (!error.isOperational) {
          Sentry.captureException(error);
        }

        return NextResponse.json(error.toJSON(), { status: error.statusCode });
      }

      // Unknown errors — always report
      log.error({ err: error }, 'Unhandled API error');
      Sentry.captureException(error);

      return NextResponse.json(
        {
          error: {
            code: ErrorCode.INTERNAL_ERROR,
            message: 'An unexpected error occurred.',
          },
        },
        { status: 500 }
      );
    }
  };
}
```

### Usage

```typescript
// app/api/projects/[id]/route.ts
import { withErrorHandler } from '@/lib/api/error-handler';
import { NotFoundError, ValidationError } from '@/lib/errors';
import { z } from 'zod';

const updateSchema = z.object({
  name: z.string().min(1).max(100),
  description: z.string().max(500).optional(),
});

export const PATCH = withErrorHandler(async (req, context) => {
  const body = await req.json();
  const data = updateSchema.parse(body); // Throws ZodError if invalid

  const project = await db.project.findUnique({
    where: { id: context!.params.id },
  });

  if (!project) {
    throw new NotFoundError('Project', context!.params.id);
  }

  const updated = await db.project.update({
    where: { id: project.id },
    data,
  });

  return NextResponse.json({ data: updated });
});
```

---

## 9. Client-Side Error Tracking

```typescript
// lib/client-error-tracking.ts
import * as Sentry from '@sentry/nextjs';

export function initClientErrorTracking() {
  // Catch unhandled promise rejections
  if (typeof window !== 'undefined') {
    window.addEventListener('unhandledrejection', (event) => {
      Sentry.captureException(event.reason, {
        tags: { errorType: 'unhandledrejection' },
      });
    });

    // Catch global errors not caught by React
    window.addEventListener('error', (event) => {
      // Ignore cross-origin script errors (no useful info)
      if (event.message === 'Script error.') return;

      Sentry.captureException(event.error ?? event.message, {
        tags: { errorType: 'window.onerror' },
        contexts: {
          errorEvent: {
            filename: event.filename,
            lineno: event.lineno,
            colno: event.colno,
          },
        },
      });
    });
  }
}

// Track API fetch errors from the client
export async function fetchWithErrorTracking<T>(
  url: string,
  options?: RequestInit
): Promise<T> {
  try {
    const response = await fetch(url, options);

    if (!response.ok) {
      const errorBody = await response.json().catch(() => ({}));

      const error = new Error(
        `API Error ${response.status}: ${errorBody?.error?.message ?? response.statusText}`
      );

      // Only report 5xx to Sentry (4xx are expected)
      if (response.status >= 500) {
        Sentry.captureException(error, {
          tags: {
            apiEndpoint: url,
            statusCode: response.status.toString(),
          },
        });
      }

      throw error;
    }

    return response.json();
  } catch (error) {
    if (error instanceof TypeError && error.message === 'Failed to fetch') {
      // Network error — user is offline or server unreachable
      Sentry.captureMessage('Network request failed', {
        level: 'warning',
        tags: { url, errorType: 'network' },
      });
    }
    throw error;
  }
}
```

### Error Boundary Cascade

```tsx
// Layout with nested error boundaries
// app/dashboard/layout.tsx
import { ErrorBoundary } from '@/components/error-boundary';

export default function DashboardLayout({ children }: { children: React.ReactNode }) {
  return (
    <ErrorBoundary level="page">
      <div className="flex">
        <ErrorBoundary level="section" fallback={<SimpleSidebar />}>
          <Sidebar />
        </ErrorBoundary>
        <main className="flex-1">
          <ErrorBoundary level="section">
            {children}
          </ErrorBoundary>
        </main>
      </div>
    </ErrorBoundary>
  );
}
```

---

## 10. Debug Workflow

### From Alert to Fix

```
Alert fired
  |
  v
1. ACKNOWLEDGE — claim the alert, set severity, notify team
  |
  v
2. ASSESS — check Sentry for:
   - Error count + affected users
   - First seen vs. regression
   - Stack trace + breadcrumbs
   - Release correlation (did a deploy cause it?)
  |
  v
3. REPRODUCE — use Sentry replay, logs, correlation ID
   - Check structured logs with correlation ID
   - Reproduce locally or in staging
   - If not reproducible, add more logging and wait
  |
  v
4. DIAGNOSE — identify root cause
   - Is it a code bug, data issue, or external service?
   - Check recent PRs/deploys for changes
   - Review the error context from AppError
  |
  v
5. FIX — implement + test
   - Write a failing test that reproduces the bug
   - Fix the code
   - Add/improve error handling if the error was unhandled
  |
  v
6. VERIFY — deploy and confirm
   - Deploy to staging first
   - Monitor Sentry for the issue to stop recurring
   - Check error budget impact
  |
  v
7. CLOSE — resolve the Sentry issue
   - Write a brief postmortem for P0/P1
   - Update runbooks if needed
   - Close the alert
```

### Postmortem Template (for P0/P1)

```markdown
## Incident: [Title]
**Severity:** P0/P1
**Duration:** [start] — [end]
**Impact:** [X users affected, Y requests failed]

### Timeline
- HH:MM — Alert fired
- HH:MM — Acknowledged by [name]
- HH:MM — Root cause identified
- HH:MM — Fix deployed
- HH:MM — Confirmed resolved

### Root Cause
[What exactly went wrong and why]

### Fix
[What was changed]

### Action Items
- [ ] [Preventive measure 1]
- [ ] [Preventive measure 2]
- [ ] [Monitoring improvement]
```

### Useful Sentry Search Queries

```
# Find errors for a specific user
user.id:user_123

# Find errors from a specific release
release:abc123

# Find unhandled errors only
error.unhandled:true

# Find errors by custom tag
errorCode:VALIDATION_ERROR

# Find errors by transaction
transaction:/api/projects/*
```
