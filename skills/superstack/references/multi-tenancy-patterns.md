# Multi-Tenancy Patterns

## 1. Shared DB + RLS

The most common pattern for SaaS: one database, one schema, tenant isolation via Row-Level Security.

### Schema Design

```sql
-- Every tenant-scoped table has a tenant_id column
CREATE TABLE tenants (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL,
  slug TEXT UNIQUE NOT NULL,
  plan TEXT NOT NULL DEFAULT 'free',
  settings JSONB DEFAULT '{}',
  created_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE projects (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  name TEXT NOT NULL,
  created_at TIMESTAMPTZ DEFAULT now()
);

-- Index on tenant_id is CRITICAL for performance
CREATE INDEX idx_projects_tenant_id ON projects(tenant_id);
```

### Supabase RLS Implementation

```sql
-- Enable RLS
ALTER TABLE projects ENABLE ROW LEVEL SECURITY;

-- Policy: users can only see their tenant's data
CREATE POLICY "tenant_isolation" ON projects
  FOR ALL
  USING (
    tenant_id = (
      SELECT tenant_id FROM user_profiles
      WHERE user_id = auth.uid()
    )
  )
  WITH CHECK (
    tenant_id = (
      SELECT tenant_id FROM user_profiles
      WHERE user_id = auth.uid()
    )
  );

-- Helper function to get current tenant
CREATE OR REPLACE FUNCTION get_current_tenant_id()
RETURNS UUID
LANGUAGE sql
STABLE
SECURITY DEFINER
AS $$
  SELECT tenant_id FROM user_profiles WHERE user_id = auth.uid()
$$;

-- Simpler policy using the helper
CREATE POLICY "tenant_isolation_v2" ON projects
  FOR ALL
  USING (tenant_id = get_current_tenant_id())
  WITH CHECK (tenant_id = get_current_tenant_id());
```

### Testing Tenant Isolation

```typescript
// tests/tenant-isolation.test.ts
import { createClient } from '@supabase/supabase-js';

describe('Tenant Isolation', () => {
  const tenantAUser = createClient(SUPABASE_URL, SUPABASE_ANON_KEY, {
    global: { headers: { Authorization: `Bearer ${TENANT_A_JWT}` } },
  });
  const tenantBUser = createClient(SUPABASE_URL, SUPABASE_ANON_KEY, {
    global: { headers: { Authorization: `Bearer ${TENANT_B_JWT}` } },
  });

  it('tenant A cannot see tenant B data', async () => {
    // Create a project as tenant B
    const { data: project } = await tenantBUser
      .from('projects')
      .insert({ name: 'Secret Project', tenant_id: TENANT_B_ID })
      .select()
      .single();

    // Try to read it as tenant A
    const { data, error } = await tenantAUser
      .from('projects')
      .select()
      .eq('id', project!.id);

    expect(data).toHaveLength(0); // RLS blocks access
  });

  it('tenant A cannot update tenant B data', async () => {
    const { error } = await tenantAUser
      .from('projects')
      .update({ name: 'Hacked' })
      .eq('id', TENANT_B_PROJECT_ID);

    // Update succeeds but affects 0 rows (RLS filters it out)
    expect(error).toBeNull();
  });
});
```

---

## 2. Schema-per-Tenant

Each tenant gets their own database schema. Use when strict data isolation is required (compliance, enterprise customers).

### When to Use

| Factor | Shared DB + RLS | Schema-per-Tenant |
|---|---|---|
| Tenant count | 100s-1000s | 10s-100s |
| Data isolation | Logical (RLS) | Physical (schema) |
| Compliance | Standard | SOC2, HIPAA, enterprise |
| Schema customization | No | Possible per tenant |
| Migration complexity | Low | High |
| Resource usage | Efficient | Higher overhead |

### Prisma Multi-Schema

```typescript
// lib/prisma-tenant.ts
import { PrismaClient } from '@prisma/client';

const prismaClients = new Map<string, PrismaClient>();

export function getPrismaForTenant(tenantSchema: string): PrismaClient {
  if (prismaClients.has(tenantSchema)) {
    return prismaClients.get(tenantSchema)!;
  }

  const client = new PrismaClient({
    datasourceUrl: `${process.env.DATABASE_URL}?schema=${tenantSchema}`,
  });

  prismaClients.set(tenantSchema, client);
  return client;
}

// Cleanup on shutdown
export async function disconnectAll() {
  for (const [, client] of prismaClients) {
    await client.$disconnect();
  }
  prismaClients.clear();
}
```

### Migration Strategy

```typescript
// scripts/migrate-tenant.ts
import { execFileSync } from 'child_process';

async function migrateTenant(tenantSchema: string) {
  // Create schema if not exists
  await adminDb.$executeRawUnsafe(
    `CREATE SCHEMA IF NOT EXISTS "${tenantSchema}"`
  );

  // Run Prisma migrations against tenant schema
  execFileSync('npx', ['prisma', 'migrate', 'deploy'], {
    stdio: 'inherit',
    env: {
      ...process.env,
      DATABASE_URL: `${process.env.DATABASE_URL}?schema=${tenantSchema}`,
    },
  });
}

// Migrate all tenants
async function migrateAll() {
  const tenants = await adminDb.tenant.findMany();
  for (const tenant of tenants) {
    console.log(`Migrating tenant: ${tenant.slug}`);
    await migrateTenant(`tenant_${tenant.slug}`);
  }
}
```

**Pros:** Strong isolation, per-tenant backups, per-tenant schema customization.
**Cons:** Migration complexity (N schemas), connection pool overhead, harder cross-tenant queries.

---

## 3. Tenant Context Middleware

Extract the tenant from subdomain, header, or JWT and inject into request context.

```typescript
// lib/tenant-context.ts
import { NextRequest } from 'next/server';
import { createClient } from '@supabase/supabase-js';

interface TenantContext {
  tenantId: string;
  tenantSlug: string;
  plan: string;
}

export async function resolveTenant(request: NextRequest): Promise<TenantContext> {
  // Strategy 1: Subdomain
  const hostname = request.headers.get('host') ?? '';
  const subdomain = hostname.split('.')[0];

  if (subdomain && subdomain !== 'www' && subdomain !== 'app') {
    const tenant = await getTenantBySlug(subdomain);
    if (tenant) return tenant;
  }

  // Strategy 2: Custom header (for API clients)
  const tenantHeader = request.headers.get('x-tenant-id');
  if (tenantHeader) {
    const tenant = await getTenantById(tenantHeader);
    if (tenant) return tenant;
  }

  // Strategy 3: JWT claim (from Supabase auth)
  const authHeader = request.headers.get('authorization');
  if (authHeader) {
    const token = authHeader.replace('Bearer ', '');
    const { data: { user } } = await supabase.auth.getUser(token);
    if (user) {
      const tenantId = user.app_metadata?.tenant_id;
      if (tenantId) {
        const tenant = await getTenantById(tenantId);
        if (tenant) return tenant;
      }
    }
  }

  throw new Error('Unable to resolve tenant');
}
```

### Next.js Middleware Integration

```typescript
// middleware.ts
import { NextRequest, NextResponse } from 'next/server';
import { resolveTenant } from '@/lib/tenant-context';

export async function middleware(request: NextRequest) {
  try {
    const tenant = await resolveTenant(request);

    // Inject tenant info into headers for downstream use
    const requestHeaders = new Headers(request.headers);
    requestHeaders.set('x-tenant-id', tenant.tenantId);
    requestHeaders.set('x-tenant-slug', tenant.tenantSlug);
    requestHeaders.set('x-tenant-plan', tenant.plan);

    return NextResponse.next({
      request: { headers: requestHeaders },
    });
  } catch {
    return NextResponse.redirect(new URL('/login', request.url));
  }
}

export const config = {
  matcher: ['/dashboard/:path*', '/api/:path*'],
};
```

### Server Component Access

```typescript
// lib/get-tenant.ts
import { headers } from 'next/headers';

export async function getTenant(): Promise<TenantContext> {
  const headerStore = await headers();
  const tenantId = headerStore.get('x-tenant-id');
  const tenantSlug = headerStore.get('x-tenant-slug');
  const plan = headerStore.get('x-tenant-plan');

  if (!tenantId || !tenantSlug || !plan) {
    throw new Error('Tenant context not available');
  }

  return { tenantId, tenantSlug, plan };
}
```

---

## 4. Tenant-Aware Caching

### Cache Key Prefixing

```typescript
// lib/cache.ts
import { Redis } from 'ioredis';

const redis = new Redis(process.env.REDIS_URL!);

export class TenantCache {
  private tenantId: string;
  private defaultTTL: number;

  constructor(tenantId: string, defaultTTL = 3600) {
    this.tenantId = tenantId;
    this.defaultTTL = defaultTTL;
  }

  private key(key: string): string {
    return `tenant:${this.tenantId}:${key}`;
  }

  async get<T>(key: string): Promise<T | null> {
    const value = await redis.get(this.key(key));
    return value ? JSON.parse(value) : null;
  }

  async set<T>(key: string, value: T, ttl?: number): Promise<void> {
    await redis.setex(this.key(key), ttl ?? this.defaultTTL, JSON.stringify(value));
  }

  async invalidate(key: string): Promise<void> {
    await redis.del(this.key(key));
  }

  // Invalidate all cache for this tenant
  async invalidateAll(): Promise<void> {
    const keys = await redis.keys(`tenant:${this.tenantId}:*`);
    if (keys.length > 0) {
      await redis.del(...keys);
    }
  }
}

// Shared cache — data that's the same across tenants (plans, features, etc.)
export class SharedCache {
  private static prefix = 'shared';

  static async get<T>(key: string): Promise<T | null> {
    const value = await redis.get(`${this.prefix}:${key}`);
    return value ? JSON.parse(value) : null;
  }

  static async set<T>(key: string, value: T, ttl = 3600): Promise<void> {
    await redis.setex(`${this.prefix}:${key}`, ttl, JSON.stringify(value));
  }
}
```

### Usage

```typescript
// In an API route
const tenant = await getTenant();
const cache = new TenantCache(tenant.tenantId);

// Check cache first
const cached = await cache.get<Project[]>('projects:list');
if (cached) return NextResponse.json({ data: cached });

// Fetch from DB
const projects = await db.project.findMany({
  where: { tenantId: tenant.tenantId },
});

// Cache the result
await cache.set('projects:list', projects, 300);

return NextResponse.json({ data: projects });
```

---

## 5. Data Isolation Patterns

### Foreign Key to Tenant

```sql
-- Every table references the tenant
CREATE TABLE documents (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  project_id UUID NOT NULL REFERENCES projects(id),
  title TEXT NOT NULL,
  content TEXT,
  created_at TIMESTAMPTZ DEFAULT now()
);

-- Composite index for common queries
CREATE INDEX idx_documents_tenant_project ON documents(tenant_id, project_id);
```

### Cascading Tenant Filter

```typescript
// lib/db/tenant-queries.ts
// Always include tenant_id in every query — defense in depth on top of RLS

export function tenantQuery(supabase: SupabaseClient, tenantId: string) {
  return {
    projects: {
      list: () =>
        supabase
          .from('projects')
          .select('*')
          .eq('tenant_id', tenantId)
          .order('created_at', { ascending: false }),

      get: (projectId: string) =>
        supabase
          .from('projects')
          .select('*, documents(*)')
          .eq('tenant_id', tenantId)  // Always filter by tenant
          .eq('id', projectId)
          .single(),

      create: (data: ProjectInsert) =>
        supabase
          .from('projects')
          .insert({ ...data, tenant_id: tenantId })  // Always set tenant_id
          .select()
          .single(),
    },
  };
}
```

### Cross-Tenant Queries (Admin Only)

```sql
-- Service role bypasses RLS
-- Use ONLY from server-side admin endpoints with proper auth checks

-- Admin: get stats across all tenants
CREATE OR REPLACE FUNCTION admin_tenant_stats()
RETURNS TABLE (
  tenant_id UUID,
  tenant_name TEXT,
  project_count BIGINT,
  user_count BIGINT
)
LANGUAGE sql
SECURITY DEFINER  -- Runs as function owner, bypasses RLS
AS $$
  SELECT
    t.id AS tenant_id,
    t.name AS tenant_name,
    COUNT(DISTINCT p.id) AS project_count,
    COUNT(DISTINCT up.user_id) AS user_count
  FROM tenants t
  LEFT JOIN projects p ON p.tenant_id = t.id
  LEFT JOIN user_profiles up ON up.tenant_id = t.id
  GROUP BY t.id, t.name
$$;
```

---

## 6. Tenant Provisioning

### Signup Flow

```typescript
// lib/tenant-provisioning.ts
import { createClient } from '@supabase/supabase-js';

const supabaseAdmin = createClient(
  process.env.SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!
);

interface ProvisionTenantParams {
  tenantName: string;
  tenantSlug: string;
  ownerEmail: string;
  ownerName: string;
  plan?: string;
}

export async function provisionTenant({
  tenantName,
  tenantSlug,
  ownerEmail,
  ownerName,
  plan = 'free',
}: ProvisionTenantParams) {
  // 1. Create tenant record
  const { data: tenant, error: tenantError } = await supabaseAdmin
    .from('tenants')
    .insert({
      name: tenantName,
      slug: tenantSlug,
      plan,
      settings: getDefaultSettings(plan),
    })
    .select()
    .single();

  if (tenantError) throw tenantError;

  // 2. Create or invite the owner user
  const { data: authUser, error: authError } = await supabaseAdmin.auth.admin.createUser({
    email: ownerEmail,
    email_confirm: true,
    app_metadata: {
      tenant_id: tenant.id,
      role: 'owner',
    },
  });

  if (authError) throw authError;

  // 3. Create user profile with tenant link
  await supabaseAdmin.from('user_profiles').insert({
    user_id: authUser.user.id,
    tenant_id: tenant.id,
    name: ownerName,
    role: 'owner',
  });

  // 4. Seed initial data
  await seedTenantData(tenant.id);

  // 5. Send welcome email
  await sendWelcomeEmail(ownerEmail, tenantName);

  return { tenant, user: authUser.user };
}

async function seedTenantData(tenantId: string) {
  // Create default project
  await supabaseAdmin.from('projects').insert({
    tenant_id: tenantId,
    name: 'My First Project',
  });

  // Create default settings, templates, etc.
  await supabaseAdmin.from('tenant_settings').insert({
    tenant_id: tenantId,
    notification_preferences: { email: true, slack: false },
    branding: { primaryColor: '#000000' },
  });
}

function getDefaultSettings(plan: string): Record<string, unknown> {
  const planLimits: Record<string, unknown> = {
    free: { maxUsers: 3, maxProjects: 5, maxStorage: 1_000_000_000 },
    pro: { maxUsers: 20, maxProjects: 50, maxStorage: 50_000_000_000 },
    enterprise: { maxUsers: -1, maxProjects: -1, maxStorage: -1 },
  };
  return { limits: planLimits[plan] ?? planLimits.free };
}
```

### Custom Domain Mapping

```typescript
// Custom domain setup for tenants
// 1. Tenant adds their domain in settings
// 2. They configure a CNAME: app.theirdomain.com -> your-app.vercel.app
// 3. You add the domain to Vercel via API

import { VercelClient } from '@vercel/sdk';

async function addCustomDomain(tenantId: string, domain: string) {
  const vercel = new VercelClient({ token: process.env.VERCEL_TOKEN });

  // Add domain to Vercel project
  await vercel.projects.addProjectDomain({
    idOrName: process.env.VERCEL_PROJECT_ID!,
    requestBody: { name: domain },
  });

  // Store mapping in database
  await supabaseAdmin
    .from('tenant_domains')
    .insert({ tenant_id: tenantId, domain, verified: false });
}
```

---

## 7. Billing per Tenant

### Stripe Customer per Tenant

```typescript
// lib/billing.ts
import Stripe from 'stripe';

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!);

export async function createTenantBillingCustomer(
  tenantId: string,
  email: string,
  tenantName: string
) {
  const customer = await stripe.customers.create({
    email,
    name: tenantName,
    metadata: { tenantId },
  });

  await supabaseAdmin
    .from('tenants')
    .update({ stripe_customer_id: customer.id })
    .eq('id', tenantId);

  return customer;
}

export async function createSubscription(
  tenantId: string,
  priceId: string
) {
  const tenant = await supabaseAdmin
    .from('tenants')
    .select('stripe_customer_id')
    .eq('id', tenantId)
    .single();

  const subscription = await stripe.subscriptions.create({
    customer: tenant.data!.stripe_customer_id,
    items: [{ price: priceId }],
    metadata: { tenantId },
  });

  await supabaseAdmin
    .from('tenants')
    .update({
      plan: getPlanFromPrice(priceId),
      stripe_subscription_id: subscription.id,
    })
    .eq('id', tenantId);

  return subscription;
}
```

### Usage-Based Billing

```typescript
// Track usage and report to Stripe
export async function reportUsage(tenantId: string, metric: string, quantity: number) {
  const tenant = await supabaseAdmin
    .from('tenants')
    .select('stripe_subscription_id')
    .eq('id', tenantId)
    .single();

  const subscription = await stripe.subscriptions.retrieve(
    tenant.data!.stripe_subscription_id
  );

  // Find the metered price item
  const meteredItem = subscription.items.data.find(
    (item) => item.price.recurring?.usage_type === 'metered'
  );

  if (meteredItem) {
    await stripe.subscriptionItems.createUsageRecord(meteredItem.id, {
      quantity,
      timestamp: Math.floor(Date.now() / 1000),
      action: 'increment',
    });
  }
}
```

### Plan Limits Enforcement

```typescript
// lib/plan-limits.ts
interface PlanLimits {
  maxUsers: number;
  maxProjects: number;
  maxStorage: number; // bytes
  features: string[];
}

const PLAN_LIMITS: Record<string, PlanLimits> = {
  free: {
    maxUsers: 3,
    maxProjects: 5,
    maxStorage: 1_000_000_000,
    features: ['basic-analytics'],
  },
  pro: {
    maxUsers: 20,
    maxProjects: 50,
    maxStorage: 50_000_000_000,
    features: ['basic-analytics', 'advanced-analytics', 'api-access', 'custom-branding'],
  },
  enterprise: {
    maxUsers: -1, // unlimited
    maxProjects: -1,
    maxStorage: -1,
    features: ['basic-analytics', 'advanced-analytics', 'api-access', 'custom-branding', 'sso', 'audit-log'],
  },
};

export async function checkLimit(
  tenantId: string,
  resource: 'users' | 'projects' | 'storage'
): Promise<{ allowed: boolean; current: number; limit: number }> {
  const tenant = await supabaseAdmin
    .from('tenants')
    .select('plan')
    .eq('id', tenantId)
    .single();

  const limits = PLAN_LIMITS[tenant.data!.plan];
  const limitValue = limits[`max${resource.charAt(0).toUpperCase() + resource.slice(1)}` as keyof PlanLimits] as number;

  // -1 means unlimited
  if (limitValue === -1) {
    return { allowed: true, current: 0, limit: -1 };
  }

  const current = await getCurrentUsage(tenantId, resource);

  return {
    allowed: current < limitValue,
    current,
    limit: limitValue,
  };
}

export function hasFeature(plan: string, feature: string): boolean {
  return PLAN_LIMITS[plan]?.features.includes(feature) ?? false;
}
```

---

## 8. Admin Panel

### Super-Admin Cross-Tenant View

```typescript
// app/api/admin/tenants/route.ts
// This endpoint uses the service role key — bypasses RLS

import { withAdminAuth } from '@/lib/admin-auth';

export const GET = withAdminAuth(async (req) => {
  const { searchParams } = new URL(req.url);
  const page = parseInt(searchParams.get('page') ?? '1');
  const limit = parseInt(searchParams.get('limit') ?? '20');

  const { data: tenants, count } = await supabaseAdmin
    .from('tenants')
    .select(`
      *,
      user_profiles(count),
      projects(count)
    `, { count: 'exact' })
    .range((page - 1) * limit, page * limit - 1)
    .order('created_at', { ascending: false });

  return NextResponse.json({
    data: tenants,
    meta: { page, totalCount: count, totalPages: Math.ceil((count ?? 0) / limit) },
  });
});
```

### Impersonation

```typescript
// lib/admin-impersonation.ts
// Allow admins to act as a tenant for debugging

export async function impersonateTenant(
  adminUserId: string,
  targetTenantId: string
): Promise<{ token: string; expiresAt: Date }> {
  // 1. Verify admin role
  const admin = await supabaseAdmin
    .from('user_profiles')
    .select('role')
    .eq('user_id', adminUserId)
    .single();

  if (admin.data?.role !== 'super_admin') {
    throw new ForbiddenError('Only super admins can impersonate');
  }

  // 2. Log the impersonation (audit trail)
  await supabaseAdmin.from('audit_log').insert({
    actor_id: adminUserId,
    action: 'impersonate_tenant',
    target_tenant_id: targetTenantId,
    timestamp: new Date().toISOString(),
  });

  // 3. Generate a short-lived token with tenant context
  const expiresAt = new Date(Date.now() + 30 * 60 * 1000); // 30 minutes

  const { data } = await supabaseAdmin.auth.admin.generateLink({
    type: 'magiclink',
    email: `admin+impersonate-${targetTenantId}@yourdomain.com`,
    options: {
      data: {
        impersonating: true,
        original_admin_id: adminUserId,
        tenant_id: targetTenantId,
      },
    },
  });

  return { token: data.properties?.hashed_token ?? '', expiresAt };
}
```

### Tenant Health Dashboard

```typescript
// Metrics to track per tenant
interface TenantHealth {
  tenantId: string;
  tenantName: string;
  metrics: {
    activeUsers7d: number;
    errorRate24h: number;
    storageUsedBytes: number;
    storageQuotaBytes: number;
    mrr: number;           // Monthly recurring revenue
    plan: string;
    lastActivityAt: string;
    healthScore: number;   // 0-100 computed score
  };
}

function computeHealthScore(metrics: TenantHealth['metrics']): number {
  let score = 100;

  // Penalize high error rate
  if (metrics.errorRate24h > 0.05) score -= 20;
  else if (metrics.errorRate24h > 0.01) score -= 10;

  // Penalize low activity
  const daysSinceActivity = daysBetween(new Date(metrics.lastActivityAt), new Date());
  if (daysSinceActivity > 30) score -= 30;
  else if (daysSinceActivity > 7) score -= 10;

  // Penalize storage near quota
  const storagePercent = metrics.storageUsedBytes / metrics.storageQuotaBytes;
  if (storagePercent > 0.9) score -= 15;
  else if (storagePercent > 0.75) score -= 5;

  return Math.max(0, score);
}
```

---

## 9. Performance

### Indexing Strategy

```sql
-- Primary index: every tenant-scoped table
CREATE INDEX idx_<table>_tenant_id ON <table>(tenant_id);

-- Composite indexes for common queries
CREATE INDEX idx_projects_tenant_status ON projects(tenant_id, status);
CREATE INDEX idx_documents_tenant_created ON documents(tenant_id, created_at DESC);

-- Partial index for active tenants
CREATE INDEX idx_projects_active ON projects(tenant_id, id)
  WHERE status = 'active';
```

### Connection Pooling

```typescript
// For shared DB: use a connection pooler (Supabase uses PgBouncer)
// Configure in Supabase Dashboard > Settings > Database > Connection Pooling

// For schema-per-tenant: limit the client pool
const MAX_CLIENTS_PER_TENANT = 5;
const MAX_TOTAL_CLIENTS = 100;

// Use Prisma connection pool settings
const prisma = new PrismaClient({
  datasourceUrl: connectionString,
  // Prisma manages its own pool
  // Configure via connection string: ?connection_limit=5&pool_timeout=10
});
```

### Query Optimization

```sql
-- BAD: Full table scan, then filter
SELECT * FROM documents WHERE title LIKE '%report%' AND tenant_id = 'abc';

-- GOOD: Tenant filter first (uses index), then scan within tenant
SELECT * FROM documents WHERE tenant_id = 'abc' AND title LIKE '%report%';

-- GOOD: Use EXPLAIN ANALYZE to verify index usage
EXPLAIN ANALYZE
SELECT * FROM projects WHERE tenant_id = 'abc-123' AND status = 'active';
-- Should show: Index Scan using idx_projects_tenant_status

-- For large tenants: consider partitioning
CREATE TABLE events (
  id UUID DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL,
  event_type TEXT NOT NULL,
  payload JSONB,
  created_at TIMESTAMPTZ DEFAULT now()
) PARTITION BY HASH (tenant_id);

-- Create partitions
CREATE TABLE events_p0 PARTITION OF events FOR VALUES WITH (MODULUS 8, REMAINDER 0);
CREATE TABLE events_p1 PARTITION OF events FOR VALUES WITH (MODULUS 8, REMAINDER 1);
-- ... up to p7
```

### Preventing Noisy Neighbors

```typescript
// Rate limiting per tenant
import { Ratelimit } from '@upstash/ratelimit';
import { Redis } from '@upstash/redis';

const ratelimit = new Ratelimit({
  redis: Redis.fromEnv(),
  limiter: Ratelimit.slidingWindow(100, '1 m'), // 100 requests per minute per tenant
  prefix: 'ratelimit:tenant',
});

export async function checkTenantRateLimit(tenantId: string) {
  const { success, limit, remaining, reset } = await ratelimit.limit(tenantId);

  if (!success) {
    throw new RateLimitError(Math.ceil((reset - Date.now()) / 1000));
  }

  return { limit, remaining };
}
```

---

## 10. Migration -- Adding Tenancy to an Existing App

### Step-by-Step Strategy

```
1. Create tenants table
2. Create a default tenant (for existing data)
3. Add tenant_id column (nullable) to all tables
4. Backfill tenant_id with default tenant
5. Make tenant_id NOT NULL
6. Add indexes
7. Enable RLS
8. Update all queries to include tenant_id
9. Test isolation thoroughly
10. Deploy
```

### Migration Script

```sql
-- Step 1: Create tenants table
CREATE TABLE tenants (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL,
  slug TEXT UNIQUE NOT NULL,
  plan TEXT NOT NULL DEFAULT 'free',
  created_at TIMESTAMPTZ DEFAULT now()
);

-- Step 2: Create default tenant
INSERT INTO tenants (id, name, slug)
VALUES ('00000000-0000-0000-0000-000000000001', 'Default', 'default');

-- Step 3: Add nullable tenant_id
ALTER TABLE projects ADD COLUMN tenant_id UUID REFERENCES tenants(id);
ALTER TABLE documents ADD COLUMN tenant_id UUID REFERENCES tenants(id);
ALTER TABLE user_profiles ADD COLUMN tenant_id UUID REFERENCES tenants(id);

-- Step 4: Backfill
UPDATE projects SET tenant_id = '00000000-0000-0000-0000-000000000001' WHERE tenant_id IS NULL;
UPDATE documents SET tenant_id = '00000000-0000-0000-0000-000000000001' WHERE tenant_id IS NULL;
UPDATE user_profiles SET tenant_id = '00000000-0000-0000-0000-000000000001' WHERE tenant_id IS NULL;

-- Step 5: Make NOT NULL
ALTER TABLE projects ALTER COLUMN tenant_id SET NOT NULL;
ALTER TABLE documents ALTER COLUMN tenant_id SET NOT NULL;
ALTER TABLE user_profiles ALTER COLUMN tenant_id SET NOT NULL;

-- Step 6: Add indexes
CREATE INDEX idx_projects_tenant_id ON projects(tenant_id);
CREATE INDEX idx_documents_tenant_id ON documents(tenant_id);
CREATE INDEX idx_user_profiles_tenant_id ON user_profiles(tenant_id);

-- Step 7: Enable RLS
ALTER TABLE projects ENABLE ROW LEVEL SECURITY;
ALTER TABLE documents ENABLE ROW LEVEL SECURITY;

CREATE POLICY "tenant_isolation" ON projects FOR ALL
  USING (tenant_id = get_current_tenant_id())
  WITH CHECK (tenant_id = get_current_tenant_id());

CREATE POLICY "tenant_isolation" ON documents FOR ALL
  USING (tenant_id = get_current_tenant_id())
  WITH CHECK (tenant_id = get_current_tenant_id());
```

### Code Migration Checklist

```typescript
// Before: No tenant awareness
const projects = await supabase.from('projects').select('*');

// After: Always filter by tenant
const tenant = await getTenant();
const projects = await supabase
  .from('projects')
  .select('*')
  .eq('tenant_id', tenant.tenantId);

// Checklist:
// [ ] All SELECT queries include tenant_id filter
// [ ] All INSERT operations set tenant_id
// [ ] All UPDATE/DELETE operations filter by tenant_id
// [ ] API routes extract tenant from context
// [ ] Cron jobs/background workers have tenant context
// [ ] File uploads are namespaced by tenant
// [ ] Search indexes are tenant-aware
// [ ] Webhooks include tenant context
// [ ] Email templates reference correct tenant branding
```

### Testing the Migration

```typescript
// tests/migration.test.ts
describe('Tenancy Migration', () => {
  it('all existing data belongs to default tenant', async () => {
    const { count } = await supabaseAdmin
      .from('projects')
      .select('*', { count: 'exact', head: true })
      .is('tenant_id', null);

    expect(count).toBe(0); // No orphaned records
  });

  it('RLS is enabled on all tenant-scoped tables', async () => {
    const { data } = await supabaseAdmin.rpc('check_rls_enabled', {
      table_names: ['projects', 'documents', 'user_profiles'],
    });

    for (const table of data) {
      expect(table.rls_enabled).toBe(true);
    }
  });

  it('new records require tenant_id', async () => {
    const { error } = await supabaseAdmin
      .from('projects')
      .insert({ name: 'Test' }); // Missing tenant_id

    expect(error).toBeTruthy();
    expect(error!.message).toContain('tenant_id');
  });
});
```
