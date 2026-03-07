# API Design Patterns (2026)

## Table of Contents

- [tRPC vs REST](#trpc-vs-rest)
  - [Decision Matrix](#decision-matrix)
  - [When to Use Each](#when-to-use-each)
  - [Migration Path (REST to tRPC)](#migration-path-rest-to-trpc)
- [Next.js Route Handlers](#nextjs-route-handlers)
  - [GET with Query Parameters](#get-with-query-parameters)
  - [POST with Request Validation](#post-with-request-validation)
  - [PUT/PATCH with Dynamic Route](#putpatch-with-dynamic-route)
- [Cursor Pagination](#cursor-pagination)
  - [Response Format](#response-format)
  - [Supabase Cursor Implementation](#supabase-cursor-implementation)
  - [Client-Side Infinite Scroll](#client-side-infinite-scroll)
- [Filtering & Sorting](#filtering-sorting)
  - [Query Parameter Design](#query-parameter-design)
  - [Supabase Filter Builder](#supabase-filter-builder)
- [Rate Limiting](#rate-limiting)
  - [Upstash Ratelimit Setup](#upstash-ratelimit-setup)
  - [Rate Limit Middleware](#rate-limit-middleware)
  - [Usage in Route Handler](#usage-in-route-handler)
- [OpenAPI Generation](#openapi-generation)
  - [Using fumadocs (Recommended for 2026)](#using-fumadocs-recommended-for-2026)
  - [Using next-swagger-doc](#using-next-swagger-doc)
  - [JSDoc Annotations for Route Handlers](#jsdoc-annotations-for-route-handlers)
- [Error Responses](#error-responses)
  - [RFC 7807 Problem Details Format](#rfc-7807-problem-details-format)
  - [Usage](#usage)
- [API Versioning](#api-versioning)
  - [URL Versioning (Recommended)](#url-versioning-recommended)
  - [Header Versioning (Alternative)](#header-versioning-alternative)
  - [Breaking Change Strategy](#breaking-change-strategy)
- [Middleware Patterns](#middleware-patterns)
  - [Auth Middleware](#auth-middleware)
  - [Composable Middleware](#composable-middleware)
  - [CORS in next.config](#cors-in-nextconfig)
- [Server Actions vs API Routes](#server-actions-vs-api-routes)
  - [Decision Matrix](#decision-matrix)
  - [Server Action Pattern](#server-action-pattern)
  - [Using Server Action in Component](#using-server-action-in-component)
  - [Security Considerations for Server Actions](#security-considerations-for-server-actions)

## tRPC vs REST

### Decision Matrix

| Factor | tRPC | REST (Route Handlers) |
|---|---|---|
| Client | Only your own Next.js app | Multiple clients, mobile, third-party |
| Type safety | End-to-end automatic | Manual (Zod + typed responses) |
| Learning curve | Moderate (routers, procedures) | Low (standard HTTP) |
| OpenAPI spec | Possible via plugin | Native with swagger tools |
| Caching | Custom, less granular | Standard HTTP caching, CDN-friendly |
| File uploads | Awkward (needs workaround) | Native multipart support |
| WebSocket | Built-in subscriptions | Manual setup |
| Bundle size | Adds ~15KB | Zero additional |

### When to Use Each

- **tRPC** -> Internal dashboard, SaaS app consumed only by your Next.js frontend, rapid prototyping
- **REST** -> Public API, mobile app backend, webhook receivers, third-party integrations
- **Both** -> tRPC for internal, REST route handlers for public/webhook endpoints

### Migration Path (REST to tRPC)

```typescript
// 1. Install
// bun add @trpc/server @trpc/client @trpc/react-query @trpc/next @tanstack/react-query

// 2. Create router (server/trpc.ts)
import { initTRPC, TRPCError } from "@trpc/server";
import superjson from "superjson";
import { auth } from "@/lib/auth";

const t = initTRPC.context<{ session: Session | null }>().create({
  transformer: superjson,
});

export const router = t.router;
export const publicProcedure = t.procedure;
export const protectedProcedure = t.procedure.use(({ ctx, next }) => {
  if (!ctx.session) {
    throw new TRPCError({ code: "UNAUTHORIZED" });
  }
  return next({ ctx: { session: ctx.session } });
});
```

---

## Next.js Route Handlers

### GET with Query Parameters

```typescript
// app/api/projects/route.ts
import { NextRequest, NextResponse } from "next/server";
import { z } from "zod";
import { supabase } from "@/lib/supabase-admin";

const querySchema = z.object({
  cursor: z.string().optional(),
  limit: z.coerce.number().min(1).max(100).default(20),
  status: z.enum(["active", "archived", "draft"]).optional(),
  sort: z.enum(["created_at", "updated_at", "name"]).default("created_at"),
  order: z.enum(["asc", "desc"]).default("desc"),
});

export async function GET(request: NextRequest) {
  try {
    const params = Object.fromEntries(request.nextUrl.searchParams);
    const parsed = querySchema.safeParse(params);

    if (!parsed.success) {
      return NextResponse.json(
        {
          type: "https://api.example.com/errors/validation",
          title: "Validation Error",
          status: 400,
          detail: "Invalid query parameters",
          errors: parsed.error.flatten().fieldErrors,
        },
        { status: 400 }
      );
    }

    const { cursor, limit, status, sort, order } = parsed.data;

    let query = supabase
      .from("projects")
      .select("id, name, status, created_at, updated_at")
      .order(sort, { ascending: order === "asc" })
      .limit(limit + 1); // fetch one extra to determine hasMore

    if (status) query = query.eq("status", status);
    if (cursor) {
      query = order === "desc"
        ? query.lt(sort, cursor)
        : query.gt(sort, cursor);
    }

    const { data, error } = await query;
    if (error) throw error;

    const hasMore = data.length > limit;
    const items = hasMore ? data.slice(0, -1) : data;
    const nextCursor = hasMore ? items[items.length - 1]?.[sort] : null;

    return NextResponse.json({
      data: items,
      pagination: {
        nextCursor,
        hasMore,
        limit,
      },
    });
  } catch (error) {
    return NextResponse.json(
      { type: "https://api.example.com/errors/internal", title: "Internal Error", status: 500 },
      { status: 500 }
    );
  }
}
```

### POST with Request Validation

```typescript
// app/api/projects/route.ts
const createProjectSchema = z.object({
  name: z.string().min(1).max(100),
  description: z.string().max(500).optional(),
  status: z.enum(["active", "draft"]).default("draft"),
  teamId: z.string().uuid(),
});

export async function POST(request: NextRequest) {
  try {
    const body = await request.json();
    const parsed = createProjectSchema.safeParse(body);

    if (!parsed.success) {
      return NextResponse.json(
        {
          type: "https://api.example.com/errors/validation",
          title: "Validation Error",
          status: 422,
          detail: "Request body validation failed",
          errors: parsed.error.flatten().fieldErrors,
        },
        { status: 422 }
      );
    }

    const { data, error } = await supabase
      .from("projects")
      .insert(parsed.data)
      .select()
      .single();

    if (error) throw error;

    return NextResponse.json({ data }, { status: 201 });
  } catch (error) {
    return NextResponse.json(
      { type: "https://api.example.com/errors/internal", title: "Internal Error", status: 500 },
      { status: 500 }
    );
  }
}
```

### PUT/PATCH with Dynamic Route

```typescript
// app/api/projects/[id]/route.ts
import { NextRequest, NextResponse } from "next/server";
import { z } from "zod";

const updateProjectSchema = z.object({
  name: z.string().min(1).max(100).optional(),
  description: z.string().max(500).optional(),
  status: z.enum(["active", "archived", "draft"]).optional(),
});

export async function PATCH(
  request: NextRequest,
  { params }: { params: Promise<{ id: string }> }
) {
  const { id } = await params;

  if (!z.string().uuid().safeParse(id).success) {
    return NextResponse.json(
      { type: "https://api.example.com/errors/validation", title: "Invalid ID", status: 400 },
      { status: 400 }
    );
  }

  const body = await request.json();
  const parsed = updateProjectSchema.safeParse(body);
  if (!parsed.success) {
    return NextResponse.json(
      { type: "https://api.example.com/errors/validation", title: "Validation Error", status: 422, errors: parsed.error.flatten().fieldErrors },
      { status: 422 }
    );
  }

  const { data, error } = await supabase
    .from("projects")
    .update(parsed.data)
    .eq("id", id)
    .select()
    .single();

  if (error?.code === "PGRST116") {
    return NextResponse.json(
      { type: "https://api.example.com/errors/not-found", title: "Not Found", status: 404 },
      { status: 404 }
    );
  }

  return NextResponse.json({ data });
}

export async function DELETE(
  _request: NextRequest,
  { params }: { params: Promise<{ id: string }> }
) {
  const { id } = await params;

  const { error } = await supabase.from("projects").delete().eq("id", id);

  if (error) {
    return NextResponse.json(
      { type: "https://api.example.com/errors/internal", title: "Delete Failed", status: 500 },
      { status: 500 }
    );
  }

  return new NextResponse(null, { status: 204 });
}
```

---

## Cursor Pagination

### Response Format

```typescript
type PaginatedResponse<T> = {
  data: T[];
  pagination: {
    nextCursor: string | null;
    hasMore: boolean;
    limit: number;
    total?: number; // optional, expensive for large tables
  };
};
```

### Supabase Cursor Implementation

```typescript
async function fetchPaginated<T>(
  table: string,
  options: {
    cursor?: string;
    limit?: number;
    sortColumn?: string;
    sortOrder?: "asc" | "desc";
    filters?: Record<string, string>;
  }
): Promise<PaginatedResponse<T>> {
  const { cursor, limit = 20, sortColumn = "created_at", sortOrder = "desc", filters = {} } = options;

  let query = supabase
    .from(table)
    .select("*", { count: "planned" }) // "planned" is cheaper than "exact"
    .order(sortColumn, { ascending: sortOrder === "asc" })
    .limit(limit + 1);

  // Apply filters
  for (const [key, value] of Object.entries(filters)) {
    query = query.eq(key, value);
  }

  // Apply cursor
  if (cursor) {
    query = sortOrder === "desc"
      ? query.lt(sortColumn, cursor)
      : query.gt(sortColumn, cursor);
  }

  const { data, error, count } = await query;
  if (error) throw error;

  const items = data as T[];
  const hasMore = items.length > limit;
  const page = hasMore ? items.slice(0, -1) : items;
  const nextCursor = hasMore
    ? (page[page.length - 1] as Record<string, unknown>)?.[sortColumn] as string
    : null;

  return {
    data: page,
    pagination: { nextCursor, hasMore, limit, total: count ?? undefined },
  };
}
```

### Client-Side Infinite Scroll

```typescript
"use client";

import { useInfiniteQuery } from "@tanstack/react-query";
import { useInView } from "react-intersection-observer";
import { useEffect } from "react";

function ProjectList() {
  const { ref, inView } = useInView({ threshold: 0 });

  const {
    data,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage,
    isLoading,
  } = useInfiniteQuery({
    queryKey: ["projects"],
    queryFn: async ({ pageParam }) => {
      const params = new URLSearchParams({ limit: "20" });
      if (pageParam) params.set("cursor", pageParam);

      const res = await fetch(`/api/projects?${params}`);
      if (!res.ok) throw new Error("Failed to fetch");
      return res.json() as Promise<PaginatedResponse<Project>>;
    },
    initialPageParam: undefined as string | undefined,
    getNextPageParam: (lastPage) => lastPage.pagination.nextCursor ?? undefined,
  });

  // Auto-fetch next page when sentinel is visible
  useEffect(() => {
    if (inView && hasNextPage && !isFetchingNextPage) {
      fetchNextPage();
    }
  }, [inView, hasNextPage, isFetchingNextPage, fetchNextPage]);

  const items = data?.pages.flatMap((page) => page.data) ?? [];

  return (
    <div>
      {items.map((project) => (
        <ProjectCard key={project.id} project={project} />
      ))}

      {/* Sentinel element */}
      <div ref={ref} className="h-10">
        {isFetchingNextPage && <Spinner />}
      </div>
    </div>
  );
}
```

---

## Filtering & Sorting

### Query Parameter Design

```typescript
// Convention: filter[field]=value, sort=field:order
// GET /api/projects?filter[status]=active&filter[team_id]=abc&sort=created_at:desc&limit=20

const filterSchema = z.object({
  "filter[status]": z.enum(["active", "archived", "draft"]).optional(),
  "filter[team_id]": z.string().uuid().optional(),
  "filter[created_after]": z.coerce.date().optional(),
  "filter[created_before]": z.coerce.date().optional(),
  "filter[search]": z.string().min(1).max(100).optional(),
  sort: z
    .string()
    .regex(/^(created_at|updated_at|name):(asc|desc)$/)
    .default("created_at:desc"),
  limit: z.coerce.number().min(1).max(100).default(20),
  cursor: z.string().optional(),
});
```

### Supabase Filter Builder

```typescript
type FilterConfig = {
  status?: string;
  teamId?: string;
  createdAfter?: Date;
  createdBefore?: Date;
  search?: string;
};

function applyFilters(
  query: ReturnType<typeof supabase.from>,
  filters: FilterConfig
) {
  if (filters.status) {
    query = query.eq("status", filters.status);
  }
  if (filters.teamId) {
    query = query.eq("team_id", filters.teamId);
  }
  if (filters.createdAfter) {
    query = query.gte("created_at", filters.createdAfter.toISOString());
  }
  if (filters.createdBefore) {
    query = query.lte("created_at", filters.createdBefore.toISOString());
  }
  if (filters.search) {
    // Full-text search with Supabase
    query = query.textSearch("name", filters.search, {
      type: "websearch",
      config: "english",
    });
  }
  return query;
}
```

---

## Rate Limiting

### Upstash Ratelimit Setup

```bash
bun add @upstash/ratelimit @upstash/redis
```

```typescript
// lib/ratelimit.ts
import { Ratelimit } from "@upstash/ratelimit";
import { Redis } from "@upstash/redis";

const redis = new Redis({
  url: process.env.UPSTASH_REDIS_URL!,
  token: process.env.UPSTASH_REDIS_TOKEN!,
});

// Per-user: 100 requests per 60s sliding window
export const rateLimitByUser = new Ratelimit({
  redis,
  limiter: Ratelimit.slidingWindow(100, "60 s"),
  analytics: true,
  prefix: "ratelimit:user",
});

// Per-IP: 30 requests per 60s (stricter for unauthenticated)
export const rateLimitByIp = new Ratelimit({
  redis,
  limiter: Ratelimit.slidingWindow(30, "60 s"),
  analytics: true,
  prefix: "ratelimit:ip",
});

// Expensive operations: 10 per hour
export const rateLimitExpensive = new Ratelimit({
  redis,
  limiter: Ratelimit.slidingWindow(10, "1 h"),
  analytics: true,
  prefix: "ratelimit:expensive",
});
```

### Rate Limit Middleware

```typescript
// lib/api-middleware.ts
import { NextRequest, NextResponse } from "next/server";
import { rateLimitByUser, rateLimitByIp } from "@/lib/ratelimit";
import { getSession } from "@/lib/auth";

export async function withRateLimit(
  request: NextRequest,
  handler: (request: NextRequest) => Promise<NextResponse>,
  options?: { limiter?: "user" | "ip" | "expensive" }
) {
  const session = await getSession(request);
  const identifier = session?.user.id ?? request.headers.get("x-forwarded-for") ?? "anonymous";

  const limiter = options?.limiter === "ip" ? rateLimitByIp : rateLimitByUser;
  const { success, limit, remaining, reset } = await limiter.limit(identifier);

  if (!success) {
    return NextResponse.json(
      {
        type: "https://api.example.com/errors/rate-limit",
        title: "Too Many Requests",
        status: 429,
        detail: `Rate limit exceeded. Try again in ${Math.ceil((reset - Date.now()) / 1000)}s.`,
      },
      {
        status: 429,
        headers: {
          "X-RateLimit-Limit": limit.toString(),
          "X-RateLimit-Remaining": "0",
          "X-RateLimit-Reset": reset.toString(),
          "Retry-After": Math.ceil((reset - Date.now()) / 1000).toString(),
        },
      }
    );
  }

  const response = await handler(request);

  // Add rate limit headers to successful responses
  response.headers.set("X-RateLimit-Limit", limit.toString());
  response.headers.set("X-RateLimit-Remaining", remaining.toString());
  response.headers.set("X-RateLimit-Reset", reset.toString());

  return response;
}
```

### Usage in Route Handler

```typescript
// app/api/projects/route.ts
export async function GET(request: NextRequest) {
  return withRateLimit(request, async (req) => {
    // ... handler logic
    return NextResponse.json({ data: projects });
  });
}
```

---

## OpenAPI Generation

### Using fumadocs (Recommended for 2026)

```bash
bun add fumadocs-openapi fumadocs-ui
```

```typescript
// lib/openapi.ts
import { createOpenAPI } from "fumadocs-openapi/server";

export const openapi = createOpenAPI({
  title: "My API",
  version: "1.0.0",
  description: "Project management API",
});
```

### Using next-swagger-doc

```bash
bun add next-swagger-doc swagger-ui-react
```

```typescript
// lib/swagger.ts
import { createSwaggerSpec } from "next-swagger-doc";

export function getApiDocs() {
  return createSwaggerSpec({
    apiFolder: "app/api",
    definition: {
      openapi: "3.1.0",
      info: {
        title: "My API",
        version: "1.0.0",
      },
      components: {
        securitySchemes: {
          bearerAuth: {
            type: "http",
            scheme: "bearer",
          },
        },
      },
    },
  });
}

// app/api-docs/page.tsx
import { getApiDocs } from "@/lib/swagger";
import ReactSwagger from "./ReactSwagger";

export default function ApiDocsPage() {
  const spec = getApiDocs();
  return <ReactSwagger spec={spec} />;
}
```

### JSDoc Annotations for Route Handlers

```typescript
/**
 * @openapi
 * /api/projects:
 *   get:
 *     summary: List projects
 *     tags: [Projects]
 *     parameters:
 *       - in: query
 *         name: cursor
 *         schema:
 *           type: string
 *       - in: query
 *         name: limit
 *         schema:
 *           type: integer
 *           default: 20
 *     responses:
 *       200:
 *         description: Paginated list of projects
 */
export async function GET(request: NextRequest) {
  // ...
}
```

---

## Error Responses

### RFC 7807 Problem Details Format

```typescript
// lib/api-errors.ts

// Error codes enum for consistent identification
export const ErrorCode = {
  VALIDATION_ERROR: "VALIDATION_ERROR",
  NOT_FOUND: "NOT_FOUND",
  UNAUTHORIZED: "UNAUTHORIZED",
  FORBIDDEN: "FORBIDDEN",
  RATE_LIMITED: "RATE_LIMITED",
  CONFLICT: "CONFLICT",
  INTERNAL_ERROR: "INTERNAL_ERROR",
  SERVICE_UNAVAILABLE: "SERVICE_UNAVAILABLE",
} as const;

type ErrorCode = (typeof ErrorCode)[keyof typeof ErrorCode];

// RFC 7807 Problem Details
type ProblemDetails = {
  type: string;       // URI reference identifying the error type
  title: string;      // Short human-readable summary
  status: number;     // HTTP status code
  detail?: string;    // Human-readable explanation specific to this occurrence
  instance?: string;  // URI reference for the specific occurrence
  code?: ErrorCode;   // Machine-readable error code
  errors?: Record<string, string[]>; // Field-level validation errors
};

export function apiError(
  code: ErrorCode,
  detail?: string,
  extra?: Partial<ProblemDetails>
): NextResponse<ProblemDetails> {
  const baseUrl = "https://api.example.com/errors";

  const defaults: Record<ErrorCode, { status: number; title: string }> = {
    VALIDATION_ERROR: { status: 422, title: "Validation Error" },
    NOT_FOUND: { status: 404, title: "Not Found" },
    UNAUTHORIZED: { status: 401, title: "Unauthorized" },
    FORBIDDEN: { status: 403, title: "Forbidden" },
    RATE_LIMITED: { status: 429, title: "Too Many Requests" },
    CONFLICT: { status: 409, title: "Conflict" },
    INTERNAL_ERROR: { status: 500, title: "Internal Server Error" },
    SERVICE_UNAVAILABLE: { status: 503, title: "Service Unavailable" },
  };

  const { status, title } = defaults[code];

  const body: ProblemDetails = {
    type: `${baseUrl}/${code.toLowerCase().replace(/_/g, "-")}`,
    title,
    status,
    code,
    detail,
    ...extra,
  };

  return NextResponse.json(body, {
    status,
    headers: { "Content-Type": "application/problem+json" },
  });
}
```

### Usage

```typescript
// Validation error with field details
return apiError("VALIDATION_ERROR", "Request body validation failed", {
  errors: parsed.error.flatten().fieldErrors,
});

// Not found
return apiError("NOT_FOUND", `Project with ID ${id} not found`);

// Rate limit
return apiError("RATE_LIMITED", "Try again in 30 seconds");

// Conflict (e.g., duplicate)
return apiError("CONFLICT", "A project with this name already exists");
```

---

## API Versioning

### URL Versioning (Recommended)

```
/api/v1/projects
/api/v2/projects
```

```typescript
// app/api/v1/projects/route.ts
export async function GET(request: NextRequest) {
  // v1 implementation
}

// app/api/v2/projects/route.ts
export async function GET(request: NextRequest) {
  // v2 implementation — different response shape
}
```

### Header Versioning (Alternative)

```typescript
// middleware.ts
export function middleware(request: NextRequest) {
  const version = request.headers.get("API-Version") ?? "2026-01-01";

  // Rewrite to versioned route internally
  if (request.nextUrl.pathname.startsWith("/api/")) {
    const url = request.nextUrl.clone();
    url.pathname = `/api/${version}${url.pathname.replace("/api", "")}`;
    return NextResponse.rewrite(url);
  }
}
```

### Breaking Change Strategy

1. **Additive changes** (non-breaking): add fields, add endpoints -> no version bump
2. **Breaking changes**: remove fields, rename fields, change types -> new version
3. **Deprecation timeline**: announce 3 months before, support old version 6 months
4. **Sunset header**: `Sunset: Sat, 01 Jun 2026 00:00:00 GMT`

```typescript
// Add deprecation headers to old versions
function withDeprecation(response: NextResponse, sunsetDate: string): NextResponse {
  response.headers.set("Deprecation", "true");
  response.headers.set("Sunset", sunsetDate);
  response.headers.set("Link", '</api/v2/docs>; rel="successor-version"');
  return response;
}
```

---

## Middleware Patterns

### Auth Middleware

```typescript
// lib/api-middleware.ts
import { auth } from "@/lib/auth";
import { headers } from "next/headers";

type AuthenticatedHandler = (
  request: NextRequest,
  context: { session: Session; params?: Promise<Record<string, string>> }
) => Promise<NextResponse>;

export function withAuth(handler: AuthenticatedHandler) {
  return async (request: NextRequest, routeContext?: { params: Promise<Record<string, string>> }) => {
    const session = await auth.api.getSession({ headers: await headers() });

    if (!session) {
      return apiError("UNAUTHORIZED", "Authentication required");
    }

    return handler(request, { session, params: routeContext?.params });
  };
}

// Usage
export const GET = withAuth(async (request, { session }) => {
  const projects = await getProjectsForUser(session.user.id);
  return NextResponse.json({ data: projects });
});
```

### Composable Middleware

```typescript
type Middleware = (
  request: NextRequest,
  next: () => Promise<NextResponse>
) => Promise<NextResponse>;

function compose(...middlewares: Middleware[]) {
  return (handler: (request: NextRequest) => Promise<NextResponse>) => {
    return async (request: NextRequest) => {
      let index = 0;

      const next = async (): Promise<NextResponse> => {
        if (index < middlewares.length) {
          const middleware = middlewares[index++];
          return middleware(request, next);
        }
        return handler(request);
      };

      return next();
    };
  };
}

// Logging middleware
const logging: Middleware = async (request, next) => {
  const start = Date.now();
  const response = await next();
  const duration = Date.now() - start;
  console.log(`${request.method} ${request.nextUrl.pathname} ${response.status} ${duration}ms`);
  return response;
};

// CORS middleware
const cors: Middleware = async (request, next) => {
  // Handle preflight
  if (request.method === "OPTIONS") {
    return new NextResponse(null, {
      status: 204,
      headers: {
        "Access-Control-Allow-Origin": process.env.ALLOWED_ORIGIN!,
        "Access-Control-Allow-Methods": "GET, POST, PUT, PATCH, DELETE, OPTIONS",
        "Access-Control-Allow-Headers": "Content-Type, Authorization",
        "Access-Control-Max-Age": "86400",
      },
    });
  }

  const response = await next();
  response.headers.set("Access-Control-Allow-Origin", process.env.ALLOWED_ORIGIN!);
  return response;
};

// Compose and use
const withMiddleware = compose(logging, cors);

export const GET = withMiddleware(async (request) => {
  return NextResponse.json({ data: "hello" });
});
```

### CORS in next.config

```typescript
// next.config.ts — simpler CORS for all API routes
const nextConfig: NextConfig = {
  async headers() {
    return [
      {
        source: "/api/:path*",
        headers: [
          { key: "Access-Control-Allow-Origin", value: process.env.ALLOWED_ORIGIN! },
          { key: "Access-Control-Allow-Methods", value: "GET,POST,PUT,PATCH,DELETE,OPTIONS" },
          { key: "Access-Control-Allow-Headers", value: "Content-Type, Authorization" },
        ],
      },
    ];
  },
};
```

---

## Server Actions vs API Routes

### Decision Matrix

| Use Case | Server Actions | API Routes |
|---|---|---|
| Form submissions | Preferred | Overkill |
| Data mutations from UI | Preferred | OK |
| Client-side fetch/SWR | Not possible | Required |
| Webhook receivers | Not possible | Required |
| File uploads | Possible (FormData) | Better control |
| Third-party consumption | Not possible | Required |
| Progressive enhancement | Built-in (no-JS works) | Requires JS |
| Caching/revalidation | `revalidatePath`/`revalidateTag` | Manual |
| Rate limiting | Server-side check | Middleware |
| OpenAPI docs | Not applicable | Yes |

### Server Action Pattern

```typescript
// app/actions/projects.ts
"use server";

import { z } from "zod";
import { auth } from "@/lib/auth";
import { headers } from "next/headers";
import { revalidatePath } from "next/cache";

const createProjectSchema = z.object({
  name: z.string().min(1).max(100),
  description: z.string().max(500).optional(),
});

type ActionResult<T = void> =
  | { success: true; data: T }
  | { success: false; error: string; fieldErrors?: Record<string, string[]> };

export async function createProject(formData: FormData): Promise<ActionResult<{ id: string }>> {
  const session = await auth.api.getSession({ headers: await headers() });
  if (!session) {
    return { success: false, error: "Unauthorized" };
  }

  const parsed = createProjectSchema.safeParse({
    name: formData.get("name"),
    description: formData.get("description"),
  });

  if (!parsed.success) {
    return {
      success: false,
      error: "Validation failed",
      fieldErrors: parsed.error.flatten().fieldErrors,
    };
  }

  const { data, error } = await supabase
    .from("projects")
    .insert({ ...parsed.data, user_id: session.user.id })
    .select("id")
    .single();

  if (error) {
    return { success: false, error: "Failed to create project" };
  }

  revalidatePath("/projects");
  return { success: true, data: { id: data.id } };
}
```

### Using Server Action in Component

```typescript
"use client";

import { useActionState } from "react";
import { createProject } from "@/app/actions/projects";

function CreateProjectForm() {
  const [state, action, isPending] = useActionState(createProject, null);

  return (
    <form action={action}>
      <input name="name" required />
      {state?.fieldErrors?.name && (
        <p className="text-sm text-destructive">{state.fieldErrors.name[0]}</p>
      )}

      <textarea name="description" />

      <button type="submit" disabled={isPending}>
        {isPending ? "Creating..." : "Create Project"}
      </button>

      {state?.success === false && !state.fieldErrors && (
        <p className="text-sm text-destructive">{state.error}</p>
      )}
    </form>
  );
}
```

### Security Considerations for Server Actions

```typescript
"use server";

// 1. Always validate input — never trust the client
const schema = z.object({ id: z.string().uuid() });

// 2. Always check authorization
export async function deleteProject(id: string) {
  const session = await auth.api.getSession({ headers: await headers() });
  if (!session) throw new Error("Unauthorized");

  // 3. Verify ownership/permission
  const { data: project } = await supabase
    .from("projects")
    .select("user_id")
    .eq("id", id)
    .single();

  if (project?.user_id !== session.user.id) {
    throw new Error("Forbidden");
  }

  // 4. Perform action
  await supabase.from("projects").delete().eq("id", id);
  revalidatePath("/projects");
}

// 5. Rate limit expensive actions
import { rateLimitExpensive } from "@/lib/ratelimit";

export async function generateReport(projectId: string) {
  const session = await auth.api.getSession({ headers: await headers() });
  if (!session) throw new Error("Unauthorized");

  const { success } = await rateLimitExpensive.limit(session.user.id);
  if (!success) throw new Error("Rate limit exceeded");

  // ... expensive operation
}
```
