# Multi-Tenancy Patterns: Next.js + Supabase

## Table of Contents
- [1. Shared DB + RLS (Row-Level Security)](#1-shared-db--rls-row-level-security)
  - [Schema: tenant_id column](#schema-tenant_id-column)
  - [RLS policies](#rls-policies)
  - [Custom JWT claims (via Supabase hook)](#custom-jwt-claims-via-supabase-hook)
  - [Testing isolation](#testing-isolation)
- [2. Schema-per-Tenant](#2-schema-per-tenant)
  - [When to use](#when-to-use)
  - [Implementation](#implementation)
  - [Migration complexity](#migration-complexity)
  - [Pros/Cons](#proscons)
- [3. Tenant Context Middleware](#3-tenant-context-middleware)
  - [Subdomain extraction](#subdomain-extraction)
  - [Tenant context provider](#tenant-context-provider)
  - [JWT-based tenant extraction](#jwt-based-tenant-extraction)
  - [API route with tenant context](#api-route-with-tenant-context)
- [4. Tenant-Aware Caching](#4-tenant-aware-caching)
  - [Cache key prefixing](#cache-key-prefixing)
  - [Tenant-specific invalidation](#tenant-specific-invalidation)
  - [Redis cache with tenant namespacing](#redis-cache-with-tenant-namespacing)
- [5. Data Isolation](#5-data-isolation)
  - [Foreign key to tenant on every table](#foreign-key-to-tenant-on-every-table)
  - [Cascading filter with database function](#cascading-filter-with-database-function)
  - [Cross-tenant admin queries (service role only)](#cross-tenant-admin-queries-service-role-only)
- [6. Tenant Provisioning](#6-tenant-provisioning)
  - [Signup flow](#signup-flow)
  - [Initial seeding](#initial-seeding)
  - [Custom domain setup](#custom-domain-setup)
  - [Domain resolution in middleware](#domain-resolution-in-middleware)
- [7. Billing per Tenant](#7-billing-per-tenant)
  - [Stripe customer per tenant](#stripe-customer-per-tenant)
  - [Usage-based billing](#usage-based-billing)
  - [Plan limits enforcement](#plan-limits-enforcement)
- [8. Admin Panel](#8-admin-panel)
  - [Super-admin middleware](#super-admin-middleware)
  - [Tenant health dashboard](#tenant-health-dashboard)
  - [Impersonation](#impersonation)
  - [Tenant health scoring](#tenant-health-scoring)
- [9. Performance](#9-performance)
  - [Indexing strategy](#indexing-strategy)
  - [Connection pooling with Supabase](#connection-pooling-with-supabase)
  - [Query optimization](#query-optimization)
  - [Monitoring slow queries per tenant](#monitoring-slow-queries-per-tenant)
- [10. Migration: Adding Tenancy to an Existing App](#10-migration-adding-tenancy-to-an-existing-app)
  - [Step-by-step migration plan](#step-by-step-migration-plan)
  - [Add tenant_id column safely](#add-tenant_id-column-safely)
  - [Migration script for user assignment](#migration-script-for-user-assignment)
  - [Verification queries](#verification-queries)
  - [Application code migration checklist](#application-code-migration-checklist)
- [Quick Decision Matrix](#quick-decision-matrix)

Reference for implementing multi-tenant architectures with Next.js (App Router) and Supabase.

---

## 1. Shared DB + RLS (Row-Level Security)

The default and recommended approach for most SaaS apps. All tenants share one database; isolation is enforced at the row level.

### Schema: tenant_id column

```sql
-- Tenants table
CREATE TABLE tenants (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL,
  slug TEXT UNIQUE NOT NULL,
  plan TEXT NOT NULL DEFAULT 'free',
  created_at TIMESTAMPTZ DEFAULT now()
);

-- Example: projects table with tenant_id
CREATE TABLE projects (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  name TEXT NOT NULL,
  created_at TIMESTAMPTZ DEFAULT now()
);

-- Index for performance (critical)
CREATE INDEX idx_projects_tenant_id ON projects(tenant_id);
```

### RLS policies

```sql
-- Enable RLS
ALTER TABLE projects ENABLE ROW LEVEL SECURITY;

-- Helper: extract tenant_id from JWT
CREATE OR REPLACE FUNCTION auth.tenant_id()
RETURNS UUID AS $$
  SELECT (current_setting('request.jwt.claims', true)::json->>'tenant_id')::UUID;
$$ LANGUAGE sql STABLE;

-- SELECT: users can only read their tenant's rows
CREATE POLICY "tenant_isolation_select" ON projects
  FOR SELECT USING (tenant_id = auth.tenant_id());

-- INSERT: force tenant_id to match JWT
CREATE POLICY "tenant_isolation_insert" ON projects
  FOR INSERT WITH CHECK (tenant_id = auth.tenant_id());

-- UPDATE: scoped to tenant
CREATE POLICY "tenant_isolation_update" ON projects
  FOR UPDATE USING (tenant_id = auth.tenant_id())
  WITH CHECK (tenant_id = auth.tenant_id());

-- DELETE: scoped to tenant
CREATE POLICY "tenant_isolation_delete" ON projects
  FOR DELETE USING (tenant_id = auth.tenant_id());
```

### Custom JWT claims (via Supabase hook)

```sql
-- Database function to add tenant_id to JWT
CREATE OR REPLACE FUNCTION public.custom_access_token_hook(event jsonb)
RETURNS jsonb AS $$
DECLARE
  claims jsonb;
  user_tenant_id uuid;
BEGIN
  SELECT t.id INTO user_tenant_id
  FROM tenant_members tm
  JOIN tenants t ON t.id = tm.tenant_id
  WHERE tm.user_id = (event->>'user_id')::uuid
  LIMIT 1;

  claims := event->'claims';
  claims := jsonb_set(claims, '{tenant_id}', to_jsonb(user_tenant_id));
  event := jsonb_set(event, '{claims}', claims);
  RETURN event;
END;
$$ LANGUAGE plpgsql;
```

### Testing isolation

```typescript
// test/tenant-isolation.test.ts
import { createClient } from "@supabase/supabase-js";

describe("Tenant Isolation", () => {
  const supabaseAdmin = createClient(
    process.env.SUPABASE_URL!,
    process.env.SUPABASE_SERVICE_ROLE_KEY!
  );

  it("tenant A cannot read tenant B data", async () => {
    // Create as tenant B (service role)
    await supabaseAdmin.from("projects").insert({
      tenant_id: TENANT_B_ID,
      name: "Secret Project",
    });

    // Query as tenant A (with tenant A JWT)
    const tenantAClient = createClient(
      process.env.SUPABASE_URL!,
      process.env.SUPABASE_ANON_KEY!,
      { global: { headers: { Authorization: `Bearer ${TENANT_A_JWT}` } } }
    );

    const { data } = await tenantAClient
      .from("projects")
      .select("*")
      .eq("name", "Secret Project");

    expect(data).toHaveLength(0);
  });

  it("tenant cannot insert rows for another tenant", async () => {
    const tenantAClient = createClient(
      process.env.SUPABASE_URL!,
      process.env.SUPABASE_ANON_KEY!,
      { global: { headers: { Authorization: `Bearer ${TENANT_A_JWT}` } } }
    );

    const { error } = await tenantAClient.from("projects").insert({
      tenant_id: TENANT_B_ID, // trying to write to another tenant
      name: "Injected Project",
    });

    expect(error).not.toBeNull();
  });
});
```

---

## 2. Schema-per-Tenant

Each tenant gets a separate PostgreSQL schema. Use only when regulatory or contractual requirements demand hard data separation.

### When to use

- Healthcare, finance, or government with strict compliance requirements
- Clients contractually require physical data separation
- Wildly different data models per tenant (rare)
- Fewer than ~100 tenants (schema count becomes a management burden beyond that)

### Implementation

```sql
-- Create schema for tenant
CREATE SCHEMA tenant_acme;

-- Create tables within schema
CREATE TABLE tenant_acme.projects (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL,
  created_at TIMESTAMPTZ DEFAULT now()
);

-- Set search_path per request
SET search_path TO tenant_acme, public;
```

```typescript
// middleware to set schema
import { createClient } from "@supabase/supabase-js";

function createTenantClient(tenantSlug: string) {
  const client = createClient(
    process.env.SUPABASE_URL!,
    process.env.SUPABASE_SERVICE_ROLE_KEY!,
    {
      db: { schema: `tenant_${tenantSlug}` },
    }
  );
  return client;
}
```

### Migration complexity

```typescript
// migrate-all-tenants.ts
import { Pool } from "pg";

async function migrateAllTenants(migrationSQL: string) {
  const pool = new Pool({ connectionString: process.env.DATABASE_URL });

  const { rows: tenants } = await pool.query(
    "SELECT slug FROM public.tenants"
  );

  for (const tenant of tenants) {
    const schema = `tenant_${tenant.slug}`;
    await pool.query(`SET search_path TO ${schema}`);
    await pool.query(migrationSQL);
    console.log(`Migrated schema: ${schema}`);
  }

  await pool.end();
}
```

### Pros/Cons

| Aspect | Shared DB + RLS | Schema-per-Tenant |
|---|---|---|
| Data isolation | Logical (RLS) | Physical (schema) |
| Migration effort | Single migration | N migrations (1 per tenant) |
| Tenant count scalability | Thousands+ | Dozens to low hundreds |
| Query complexity | Simple (tenant_id filter) | Schema switching |
| Backup/restore per tenant | Complex | Easy (pg_dump per schema) |
| Cost | Low | Higher (more overhead) |

**Recommendation:** Use shared DB + RLS unless you have a hard compliance reason not to.

---

## 3. Tenant Context Middleware

Extract tenant identity early in the request lifecycle and make it available everywhere.

### Subdomain extraction

```typescript
// middleware.ts
import { NextRequest, NextResponse } from "next/server";

export function middleware(request: NextRequest) {
  const host = request.headers.get("host") || "";
  const subdomain = extractSubdomain(host);

  if (!subdomain) {
    return NextResponse.redirect(new URL("/select-workspace", request.url));
  }

  // Forward tenant context via headers
  const requestHeaders = new Headers(request.headers);
  requestHeaders.set("x-tenant-slug", subdomain);

  return NextResponse.next({ request: { headers: requestHeaders } });
}

function extractSubdomain(host: string): string | null {
  const parts = host.split(".");
  // app.example.com -> "app"
  // example.com -> null
  if (parts.length >= 3) {
    return parts[0];
  }
  return null;
}

export const config = {
  matcher: ["/((?!_next/static|_next/image|favicon.ico|api/webhooks).*)"],
};
```

### Tenant context provider

```typescript
// lib/tenant-context.ts
import { headers } from "next/headers";
import { cache } from "react";
import { createServerClient } from "@supabase/ssr";

export const getTenant = cache(async () => {
  const headerStore = await headers();
  const slug = headerStore.get("x-tenant-slug");

  if (!slug) throw new Error("No tenant context");

  const supabase = createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.SUPABASE_SERVICE_ROLE_KEY!,
    { cookies: { getAll: () => [] } }
  );

  const { data: tenant, error } = await supabase
    .from("tenants")
    .select("*")
    .eq("slug", slug)
    .single();

  if (error || !tenant) throw new Error(`Tenant not found: ${slug}`);

  return tenant;
});
```

### JWT-based tenant extraction

```typescript
// lib/tenant-from-jwt.ts
import { createServerClient } from "@supabase/ssr";
import { cookies } from "next/headers";

export async function getTenantFromSession() {
  const cookieStore = await cookies();
  const supabase = createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll: () => cookieStore.getAll(),
      },
    }
  );

  const {
    data: { session },
  } = await supabase.auth.getSession();

  if (!session) throw new Error("Not authenticated");

  // tenant_id is embedded in the JWT via custom claims hook
  const tenantId = session.user?.app_metadata?.tenant_id;
  if (!tenantId) throw new Error("No tenant in session");

  return tenantId;
}
```

### API route with tenant context

```typescript
// app/api/projects/route.ts
import { NextRequest, NextResponse } from "next/server";
import { getTenant } from "@/lib/tenant-context";

export async function GET(request: NextRequest) {
  const tenant = await getTenant();

  // Supabase client with RLS will automatically scope queries
  // because tenant_id is in the JWT
  const { data } = await supabase
    .from("projects")
    .select("*");

  return NextResponse.json(data);
}
```

---

## 4. Tenant-Aware Caching

### Cache key prefixing

```typescript
// lib/cache.ts
import { unstable_cache } from "next/cache";
import { getTenant } from "@/lib/tenant-context";

export function tenantCache<T>(
  fn: (tenantId: string) => Promise<T>,
  keyParts: string[],
  options?: { revalidate?: number; tags?: string[] }
) {
  return async () => {
    const tenant = await getTenant();
    const tenantTags = (options?.tags || []).map(
      (tag) => `tenant:${tenant.id}:${tag}`
    );

    return unstable_cache(
      () => fn(tenant.id),
      [`tenant:${tenant.id}`, ...keyParts],
      {
        revalidate: options?.revalidate ?? 60,
        tags: [`tenant:${tenant.id}`, ...tenantTags],
      }
    )();
  };
}

// Usage
export const getProjects = tenantCache(
  async (tenantId) => {
    const { data } = await supabase
      .from("projects")
      .select("*")
      .eq("tenant_id", tenantId);
    return data;
  },
  ["projects"],
  { revalidate: 300, tags: ["projects"] }
);
```

### Tenant-specific invalidation

```typescript
// lib/revalidate.ts
import { revalidateTag } from "next/cache";

export function revalidateTenantCache(
  tenantId: string,
  resource: string
) {
  // Invalidate specific resource for this tenant only
  revalidateTag(`tenant:${tenantId}:${resource}`);
}

export function revalidateAllTenantCache(tenantId: string) {
  // Invalidate everything for this tenant
  revalidateTag(`tenant:${tenantId}`);
}

// In a server action or API route
export async function updateProject(projectId: string, data: any) {
  const tenant = await getTenant();

  await supabase
    .from("projects")
    .update(data)
    .eq("id", projectId)
    .eq("tenant_id", tenant.id);

  revalidateTenantCache(tenant.id, "projects");
}
```

### Redis cache with tenant namespacing

```typescript
// lib/redis-cache.ts
import { Redis } from "ioredis";

const redis = new Redis(process.env.REDIS_URL!);

export async function getTenantCached<T>(
  tenantId: string,
  key: string,
  fetcher: () => Promise<T>,
  ttlSeconds = 300
): Promise<T> {
  const cacheKey = `t:${tenantId}:${key}`;
  const cached = await redis.get(cacheKey);

  if (cached) return JSON.parse(cached);

  const data = await fetcher();
  await redis.setex(cacheKey, ttlSeconds, JSON.stringify(data));
  return data;
}

export async function invalidateTenantKeys(
  tenantId: string,
  pattern = "*"
) {
  const keys = await redis.keys(`t:${tenantId}:${pattern}`);
  if (keys.length > 0) {
    await redis.del(...keys);
  }
}
```

---

## 5. Data Isolation

### Foreign key to tenant on every table

```sql
-- Every tenant-scoped table follows this pattern
CREATE TABLE invoices (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  amount DECIMAL(10,2) NOT NULL,
  created_at TIMESTAMPTZ DEFAULT now()
);

-- Composite index for common queries
CREATE INDEX idx_invoices_tenant_project
  ON invoices(tenant_id, project_id);

ALTER TABLE invoices ENABLE ROW LEVEL SECURITY;

CREATE POLICY "tenant_isolation" ON invoices
  FOR ALL USING (tenant_id = auth.tenant_id())
  WITH CHECK (tenant_id = auth.tenant_id());
```

### Cascading filter with database function

```sql
-- Ensure referential integrity across tenant boundaries
-- Prevent project_id from tenant B being linked in tenant A's invoice
CREATE OR REPLACE FUNCTION check_same_tenant()
RETURNS TRIGGER AS $$
BEGIN
  IF NOT EXISTS (
    SELECT 1 FROM projects
    WHERE id = NEW.project_id AND tenant_id = NEW.tenant_id
  ) THEN
    RAISE EXCEPTION 'Cross-tenant reference violation: project belongs to different tenant';
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER enforce_tenant_ref_invoices
  BEFORE INSERT OR UPDATE ON invoices
  FOR EACH ROW EXECUTE FUNCTION check_same_tenant();
```

### Cross-tenant admin queries (service role only)

```typescript
// lib/admin-queries.ts
import { createClient } from "@supabase/supabase-js";

// Service role client bypasses RLS
const adminClient = createClient(
  process.env.SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!
);

export async function getGlobalStats() {
  const { data } = await adminClient.rpc("get_global_stats");
  return data;
}

// Aggregate stats across tenants
export async function getTenantUsageSummary() {
  const { data } = await adminClient
    .from("projects")
    .select("tenant_id, count:id.count()")
    .order("count", { ascending: false });
  return data;
}
```

```sql
-- Database function for cross-tenant reporting
CREATE OR REPLACE FUNCTION get_global_stats()
RETURNS JSON AS $$
  SELECT json_build_object(
    'total_tenants', (SELECT count(*) FROM tenants),
    'total_users', (SELECT count(*) FROM auth.users),
    'total_projects', (SELECT count(*) FROM projects),
    'revenue_by_plan', (
      SELECT json_agg(row_to_json(t))
      FROM (
        SELECT plan, count(*), sum(mrr) as total_mrr
        FROM tenants
        GROUP BY plan
      ) t
    )
  );
$$ LANGUAGE sql SECURITY DEFINER;
```

---

## 6. Tenant Provisioning

### Signup flow

```typescript
// app/api/tenants/provision/route.ts
import { NextRequest, NextResponse } from "next/server";
import { createClient } from "@supabase/supabase-js";

const adminClient = createClient(
  process.env.SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!
);

export async function POST(request: NextRequest) {
  const { tenantName, slug, ownerEmail, ownerPassword } =
    await request.json();

  // Validate slug uniqueness
  const { data: existing } = await adminClient
    .from("tenants")
    .select("id")
    .eq("slug", slug)
    .single();

  if (existing) {
    return NextResponse.json(
      { error: "Workspace URL already taken" },
      { status: 409 }
    );
  }

  // 1. Create tenant
  const { data: tenant, error: tenantError } = await adminClient
    .from("tenants")
    .insert({
      name: tenantName,
      slug,
      plan: "free",
      settings: { timezone: "Europe/Berlin" },
    })
    .select()
    .single();

  if (tenantError) throw tenantError;

  // 2. Create owner user
  const { data: authUser, error: authError } =
    await adminClient.auth.admin.createUser({
      email: ownerEmail,
      password: ownerPassword,
      email_confirm: true,
      app_metadata: { tenant_id: tenant.id },
    });

  if (authError) {
    // Rollback tenant creation
    await adminClient.from("tenants").delete().eq("id", tenant.id);
    throw authError;
  }

  // 3. Create tenant membership
  await adminClient.from("tenant_members").insert({
    tenant_id: tenant.id,
    user_id: authUser.user.id,
    role: "owner",
  });

  // 4. Seed initial data
  await seedTenantData(tenant.id);

  // 5. Create Stripe customer (see section 7)
  await createStripeCustomer(tenant);

  return NextResponse.json({ tenant, redirect: `https://${slug}.app.com` });
}
```

### Initial seeding

```typescript
// lib/tenant-seed.ts
async function seedTenantData(tenantId: string) {
  const adminClient = createClient(
    process.env.SUPABASE_URL!,
    process.env.SUPABASE_SERVICE_ROLE_KEY!
  );

  // Default project
  await adminClient.from("projects").insert({
    tenant_id: tenantId,
    name: "Getting Started",
  });

  // Default notification settings
  await adminClient.from("tenant_settings").insert({
    tenant_id: tenantId,
    key: "notifications",
    value: {
      email_digest: "weekly",
      slack_enabled: false,
    },
  });

  // Default roles
  await adminClient.from("tenant_roles").insert([
    { tenant_id: tenantId, name: "admin", permissions: ["*"] },
    {
      tenant_id: tenantId,
      name: "member",
      permissions: ["read", "write"],
    },
    { tenant_id: tenantId, name: "viewer", permissions: ["read"] },
  ]);
}
```

### Custom domain setup

```typescript
// app/api/tenants/domains/route.ts
export async function POST(request: NextRequest) {
  const tenant = await getTenant();
  const { domain } = await request.json();

  // 1. Store domain mapping
  await adminClient.from("tenant_domains").insert({
    tenant_id: tenant.id,
    domain,
    verified: false,
  });

  // 2. Add domain to Vercel via API
  const vercelRes = await fetch(
    `https://api.vercel.com/v10/projects/${process.env.VERCEL_PROJECT_ID}/domains`,
    {
      method: "POST",
      headers: {
        Authorization: `Bearer ${process.env.VERCEL_TOKEN}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({ name: domain }),
    }
  );

  const vercelData = await vercelRes.json();

  // 3. Return DNS instructions to user
  return NextResponse.json({
    domain,
    dns_records: vercelData.verification || [],
    instructions:
      "Add the CNAME record to your DNS provider, then verify.",
  });
}
```

### Domain resolution in middleware

```typescript
// middleware.ts (extended)
async function resolveTenant(host: string): Promise<string | null> {
  // Check subdomain first
  const subdomain = extractSubdomain(host);
  if (subdomain) return subdomain;

  // Check custom domain mapping
  // Use edge-compatible fetch to Supabase
  const res = await fetch(
    `${process.env.NEXT_PUBLIC_SUPABASE_URL}/rest/v1/tenant_domains?domain=eq.${host}&verified=eq.true&select=tenants(slug)`,
    {
      headers: {
        apikey: process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
        Authorization: `Bearer ${process.env.SUPABASE_SERVICE_ROLE_KEY!}`,
      },
    }
  );

  const data = await res.json();
  return data?.[0]?.tenants?.slug || null;
}
```

---

## 7. Billing per Tenant

### Stripe customer per tenant

```typescript
// lib/billing.ts
import Stripe from "stripe";

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!);

export async function createStripeCustomer(tenant: {
  id: string;
  name: string;
  slug: string;
}) {
  const customer = await stripe.customers.create({
    name: tenant.name,
    metadata: {
      tenant_id: tenant.id,
      tenant_slug: tenant.slug,
    },
  });

  await adminClient
    .from("tenants")
    .update({ stripe_customer_id: customer.id })
    .eq("id", tenant.id);

  return customer;
}

export async function createCheckoutSession(
  tenantId: string,
  priceId: string
) {
  const { data: tenant } = await adminClient
    .from("tenants")
    .select("stripe_customer_id, slug")
    .eq("id", tenantId)
    .single();

  return stripe.checkout.sessions.create({
    customer: tenant!.stripe_customer_id,
    mode: "subscription",
    line_items: [{ price: priceId, quantity: 1 }],
    success_url: `https://${tenant!.slug}.app.com/settings/billing?success=true`,
    cancel_url: `https://${tenant!.slug}.app.com/settings/billing`,
    metadata: { tenant_id: tenantId },
  });
}
```

### Usage-based billing

```typescript
// lib/usage-tracking.ts
export async function trackUsage(
  tenantId: string,
  metric: string,
  quantity: number
) {
  // Store in DB for internal tracking
  await adminClient.from("usage_events").insert({
    tenant_id: tenantId,
    metric,
    quantity,
    timestamp: new Date().toISOString(),
  });

  // Report to Stripe for metered billing
  const { data: tenant } = await adminClient
    .from("tenants")
    .select("stripe_subscription_id")
    .eq("id", tenantId)
    .single();

  if (tenant?.stripe_subscription_id) {
    const subscription = await stripe.subscriptions.retrieve(
      tenant.stripe_subscription_id
    );

    const meteredItem = subscription.items.data.find(
      (item) => item.price.recurring?.usage_type === "metered"
    );

    if (meteredItem) {
      await stripe.subscriptionItems.createUsageRecord(meteredItem.id, {
        quantity,
        timestamp: Math.floor(Date.now() / 1000),
        action: "increment",
      });
    }
  }
}
```

### Plan limits enforcement

```typescript
// lib/plan-limits.ts
const PLAN_LIMITS: Record<string, Record<string, number>> = {
  free: { projects: 3, members: 2, storage_mb: 100 },
  pro: { projects: 50, members: 20, storage_mb: 10000 },
  enterprise: { projects: -1, members: -1, storage_mb: -1 }, // -1 = unlimited
};

export async function checkPlanLimit(
  tenantId: string,
  resource: string
): Promise<{ allowed: boolean; current: number; limit: number }> {
  const { data: tenant } = await adminClient
    .from("tenants")
    .select("plan")
    .eq("id", tenantId)
    .single();

  const limit = PLAN_LIMITS[tenant!.plan]?.[resource] ?? 0;
  if (limit === -1) return { allowed: true, current: 0, limit: -1 };

  const { count } = await adminClient
    .from(resource)
    .select("*", { count: "exact", head: true })
    .eq("tenant_id", tenantId);

  return {
    allowed: (count ?? 0) < limit,
    current: count ?? 0,
    limit,
  };
}

// Usage in API route
export async function POST(request: NextRequest) {
  const tenant = await getTenant();
  const { allowed, current, limit } = await checkPlanLimit(
    tenant.id,
    "projects"
  );

  if (!allowed) {
    return NextResponse.json(
      {
        error: "Plan limit reached",
        current,
        limit,
        upgrade_url: `/settings/billing`,
      },
      { status: 403 }
    );
  }

  // ... create project
}
```

---

## 8. Admin Panel

### Super-admin middleware

```typescript
// middleware/admin.ts
export async function requireSuperAdmin() {
  const supabase = await createServerClient();
  const {
    data: { user },
  } = await supabase.auth.getUser();

  if (!user) throw new Error("Not authenticated");

  const { data: admin } = await adminClient
    .from("super_admins")
    .select("id")
    .eq("user_id", user.id)
    .single();

  if (!admin) throw new Error("Not authorized");

  return user;
}
```

### Tenant health dashboard

```typescript
// app/admin/tenants/page.tsx
import { requireSuperAdmin } from "@/middleware/admin";

export default async function TenantsAdminPage() {
  await requireSuperAdmin();

  const { data: tenants } = await adminClient
    .from("tenants")
    .select(`
      *,
      tenant_members(count),
      projects(count),
      usage_events(
        metric,
        quantity.sum()
      )
    `)
    .order("created_at", { ascending: false });

  return (
    <div>
      <h1>Tenant Dashboard</h1>
      <table>
        <thead>
          <tr>
            <th>Tenant</th>
            <th>Plan</th>
            <th>Members</th>
            <th>Projects</th>
            <th>MRR</th>
            <th>Health</th>
            <th>Actions</th>
          </tr>
        </thead>
        <tbody>
          {tenants?.map((tenant) => (
            <tr key={tenant.id}>
              <td>{tenant.name}</td>
              <td>{tenant.plan}</td>
              <td>{tenant.tenant_members[0]?.count ?? 0}</td>
              <td>{tenant.projects[0]?.count ?? 0}</td>
              <td>{formatCurrency(tenant.mrr)}</td>
              <td>
                <HealthBadge tenant={tenant} />
              </td>
              <td>
                <ImpersonateButton tenantId={tenant.id} />
              </td>
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  );
}
```

### Impersonation

```typescript
// app/api/admin/impersonate/route.ts
export async function POST(request: NextRequest) {
  await requireSuperAdmin();
  const { tenantId } = await request.json();

  const { data: tenant } = await adminClient
    .from("tenants")
    .select("slug")
    .eq("id", tenantId)
    .single();

  // Get or create an impersonation user for this tenant
  const { data: owner } = await adminClient
    .from("tenant_members")
    .select("user_id")
    .eq("tenant_id", tenantId)
    .eq("role", "owner")
    .single();

  // Generate a magic link for the owner account
  const { data: link } = await adminClient.auth.admin.generateLink({
    type: "magiclink",
    email: owner!.user_id, // need to look up email
  });

  // Log the impersonation event
  await adminClient.from("admin_audit_log").insert({
    action: "impersonate",
    target_tenant_id: tenantId,
    admin_user_id: (await adminClient.auth.getUser()).data.user?.id,
    metadata: { timestamp: new Date().toISOString() },
  });

  return NextResponse.json({
    redirect: `https://${tenant!.slug}.app.com/auth/callback?token=${link.properties.hashed_token}`,
  });
}
```

### Tenant health scoring

```sql
-- Health score function
CREATE OR REPLACE FUNCTION calculate_tenant_health(p_tenant_id UUID)
RETURNS JSON AS $$
DECLARE
  result JSON;
  last_active TIMESTAMPTZ;
  member_count INT;
  active_members INT;
BEGIN
  SELECT max(created_at) INTO last_active
  FROM usage_events WHERE tenant_id = p_tenant_id;

  SELECT count(*) INTO member_count
  FROM tenant_members WHERE tenant_id = p_tenant_id;

  SELECT count(DISTINCT user_id) INTO active_members
  FROM usage_events
  WHERE tenant_id = p_tenant_id
    AND timestamp > now() - interval '7 days';

  SELECT json_build_object(
    'last_active', last_active,
    'days_since_active', EXTRACT(DAY FROM now() - last_active),
    'member_count', member_count,
    'active_members_7d', active_members,
    'activation_rate', ROUND(active_members::DECIMAL / GREATEST(member_count, 1) * 100, 1),
    'status', CASE
      WHEN last_active IS NULL THEN 'never_active'
      WHEN last_active < now() - interval '30 days' THEN 'churning'
      WHEN last_active < now() - interval '7 days' THEN 'at_risk'
      WHEN active_members::DECIMAL / GREATEST(member_count, 1) < 0.3 THEN 'low_engagement'
      ELSE 'healthy'
    END
  ) INTO result;

  RETURN result;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

---

## 9. Performance

### Indexing strategy

```sql
-- Primary tenant index on every table (mandatory)
CREATE INDEX idx_{table}_tenant_id ON {table}(tenant_id);

-- Composite indexes for common query patterns
CREATE INDEX idx_projects_tenant_status
  ON projects(tenant_id, status);

CREATE INDEX idx_invoices_tenant_created
  ON invoices(tenant_id, created_at DESC);

-- Partial index for active tenants (reduces index size)
CREATE INDEX idx_projects_active_tenants
  ON projects(tenant_id, created_at)
  WHERE archived = false;

-- Covering index to avoid heap lookups
CREATE INDEX idx_projects_tenant_covering
  ON projects(tenant_id)
  INCLUDE (name, status, created_at);
```

### Connection pooling with Supabase

```typescript
// Use Supabase connection pooler for high-traffic apps
// supabase.config.ts

// Transaction mode (default, recommended for serverless)
const supabasePooler = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
  {
    db: {
      // Use pooler connection string in production
      // Format: postgresql://user:pass@pooler.supabase.com:6543/postgres
    },
  }
);

// For long-running connections (realtime, subscriptions)
// Use direct connection, not pooler
```

### Query optimization

```typescript
// BAD: N+1 query per tenant
async function getProjectsWithTasks_BAD(tenantId: string) {
  const { data: projects } = await supabase
    .from("projects")
    .select("*")
    .eq("tenant_id", tenantId);

  for (const project of projects!) {
    const { data: tasks } = await supabase
      .from("tasks")
      .select("*")
      .eq("project_id", project.id); // N queries!
    project.tasks = tasks;
  }
  return projects;
}

// GOOD: Single query with join
async function getProjectsWithTasks_GOOD(tenantId: string) {
  const { data } = await supabase
    .from("projects")
    .select(`
      *,
      tasks(*)
    `)
    .eq("tenant_id", tenantId)
    .order("created_at", { ascending: false })
    .limit(50);

  return data;
}

// GOOD: Paginated with cursor
async function getProjectsPaginated(
  tenantId: string,
  cursor?: string,
  limit = 20
) {
  let query = supabase
    .from("projects")
    .select("*")
    .eq("tenant_id", tenantId)
    .order("created_at", { ascending: false })
    .limit(limit);

  if (cursor) {
    query = query.lt("created_at", cursor);
  }

  const { data } = await query;
  const nextCursor = data?.at(-1)?.created_at;

  return { data, nextCursor, hasMore: data?.length === limit };
}
```

### Monitoring slow queries per tenant

```sql
-- Find slow queries by tenant
SELECT
  tenant_id,
  count(*) as slow_query_count,
  avg(total_exec_time) as avg_time_ms
FROM pg_stat_statements pss
JOIN (
  -- Custom logging table populated by a trigger or middleware
  SELECT query_hash, tenant_id, exec_time
  FROM query_log
  WHERE exec_time > 1000 -- > 1 second
    AND created_at > now() - interval '24 hours'
) ql ON pss.queryid = ql.query_hash
GROUP BY tenant_id
ORDER BY slow_query_count DESC;
```

---

## 10. Migration: Adding Tenancy to an Existing App

### Step-by-step migration plan

1. **Add tenant infrastructure** (no breaking changes)
2. **Backfill existing data** to a default tenant
3. **Add RLS policies** (start with permissive, tighten gradually)
4. **Update application code** to be tenant-aware
5. **Test thoroughly** with multiple tenants
6. **Enable hard enforcement**

### Add tenant_id column safely

```sql
-- Step 1: Add column as nullable (no downtime)
ALTER TABLE projects ADD COLUMN tenant_id UUID REFERENCES tenants(id);

-- Step 2: Create default tenant for existing data
INSERT INTO tenants (id, name, slug, plan)
VALUES ('00000000-0000-0000-0000-000000000001', 'Default', 'default', 'pro')
ON CONFLICT DO NOTHING;

-- Step 3: Backfill all existing rows
UPDATE projects SET tenant_id = '00000000-0000-0000-0000-000000000001'
WHERE tenant_id IS NULL;

-- Step 4: Make column NOT NULL (after backfill is complete)
ALTER TABLE projects ALTER COLUMN tenant_id SET NOT NULL;

-- Step 5: Add index
CREATE INDEX CONCURRENTLY idx_projects_tenant_id ON projects(tenant_id);

-- Step 6: Enable RLS
ALTER TABLE projects ENABLE ROW LEVEL SECURITY;

-- Step 7: Permissive policy first (allows everything, logs violations)
CREATE POLICY "migration_permissive" ON projects
  FOR ALL USING (true);

-- Step 8: After testing, replace with restrictive policy
DROP POLICY "migration_permissive" ON projects;
CREATE POLICY "tenant_isolation" ON projects
  FOR ALL USING (tenant_id = auth.tenant_id())
  WITH CHECK (tenant_id = auth.tenant_id());
```

### Migration script for user assignment

```typescript
// scripts/migrate-users-to-tenants.ts
async function migrateUsersToTenants() {
  const adminClient = createClient(
    process.env.SUPABASE_URL!,
    process.env.SUPABASE_SERVICE_ROLE_KEY!
  );

  // Get all users not yet assigned to a tenant
  const {
    data: { users },
  } = await adminClient.auth.admin.listUsers();

  const defaultTenantId = "00000000-0000-0000-0000-000000000001";

  for (const user of users) {
    // Check if already a member
    const { data: existing } = await adminClient
      .from("tenant_members")
      .select("id")
      .eq("user_id", user.id)
      .single();

    if (existing) continue;

    // Assign to default tenant
    await adminClient.from("tenant_members").insert({
      tenant_id: defaultTenantId,
      user_id: user.id,
      role: "member",
    });

    // Update JWT claims
    await adminClient.auth.admin.updateUserById(user.id, {
      app_metadata: { tenant_id: defaultTenantId },
    });

    console.log(`Migrated user ${user.email} to default tenant`);
  }
}
```

### Verification queries

```sql
-- Check for rows missing tenant_id (should return 0)
SELECT 'projects' as table_name, count(*) as missing
FROM projects WHERE tenant_id IS NULL
UNION ALL
SELECT 'invoices', count(*) FROM invoices WHERE tenant_id IS NULL
UNION ALL
SELECT 'tasks', count(*) FROM tasks WHERE tenant_id IS NULL;

-- Verify RLS is enabled on all tenant tables
SELECT schemaname, tablename, rowsecurity
FROM pg_tables
WHERE schemaname = 'public'
  AND tablename IN ('projects', 'invoices', 'tasks', 'tenant_members')
ORDER BY tablename;

-- Verify all tenant tables have tenant_id index
SELECT t.tablename, i.indexname
FROM pg_tables t
LEFT JOIN pg_indexes i
  ON i.tablename = t.tablename
  AND i.indexdef LIKE '%tenant_id%'
WHERE t.schemaname = 'public'
  AND t.tablename IN ('projects', 'invoices', 'tasks')
ORDER BY t.tablename;
```

### Application code migration checklist

```typescript
// Before: single-tenant query
const { data } = await supabase.from("projects").select("*");

// After: tenant-scoped (RLS handles filtering automatically if JWT has tenant_id)
const { data } = await supabase.from("projects").select("*");
// No code change needed if RLS + JWT claims are properly configured!

// But for service role queries (admin), always filter explicitly:
const { data } = await adminClient
  .from("projects")
  .select("*")
  .eq("tenant_id", tenantId); // MANDATORY for service role
```

---

## Quick Decision Matrix

| Scenario | Approach |
|---|---|
| SaaS with < 10k tenants, standard isolation | Shared DB + RLS |
| Regulated industry, contractual data separation | Schema-per-tenant |
| B2B with custom domains | Shared DB + RLS + domain mapping |
| Marketplace with sellers & buyers | Shared DB + RLS + role-based policies |
| White-label product | Schema-per-tenant or shared + heavy config |
| Internal tool with department separation | Shared DB + RLS (simplest) |
