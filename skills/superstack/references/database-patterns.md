# Database Patterns — Next.js + Supabase/PostgreSQL

Production-ready patterns for database design, querying, and operations.

---

## 1. ORM Comparison — Prisma vs Drizzle vs Raw SQL

### Decision Matrix

| Criteria | Prisma | Drizzle ORM | Supabase Client (raw SQL) |
|---|---|---|---|
| **DX** | Excellent — schema-first, auto-migrations | Good — SQL-like syntax, lightweight | Good — minimal abstraction |
| **Type Safety** | Generated types from schema | Inferred from schema definition | Manual or generated via `supabase gen types` |
| **Performance** | Overhead from query engine | Minimal — compiles to SQL | Direct — no abstraction layer |
| **Edge Compatibility** | Prisma Accelerate required | Native edge support | Native edge support |
| **Bundle Size** | Large (~2MB engine) | Small (~50KB) | Small (Supabase client) |
| **Migrations** | Built-in (`prisma migrate`) | Built-in (`drizzle-kit`) | Supabase CLI (`supabase db diff`) |
| **RLS Support** | No native support | No native support | First-class — policies enforced automatically |

### Recommendation

- **Supabase client** — best for projects already on Supabase with RLS. Zero abstraction overhead.
- **Drizzle** — best when you need an ORM with edge compatibility and want SQL-like control.
- **Prisma** — best for teams that prioritize DX and don't need edge runtime.

### Drizzle Setup (with Supabase)

```typescript
// drizzle/schema.ts
import { pgTable, uuid, text, timestamp, boolean } from "drizzle-orm/pg-core";

export const users = pgTable("users", {
  id: uuid("id").defaultRandom().primaryKey(),
  email: text("email").notNull().unique(),
  displayName: text("display_name"),
  isActive: boolean("is_active").default(true).notNull(),
  createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().notNull(),
});
```

```typescript
// drizzle/db.ts
import { drizzle } from "drizzle-orm/postgres-js";
import postgres from "postgres";
import * as schema from "./schema";

const connectionString = process.env.DATABASE_URL!;
const client = postgres(connectionString);
export const db = drizzle(client, { schema });
```

### Supabase Client (Recommended for Supabase Projects)

```typescript
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
        getAll: () => cookieStore.getAll(),
        setAll: (cookiesToSet) => {
          cookiesToSet.forEach(({ name, value, options }) =>
            cookieStore.set(name, value, options)
          );
        },
      },
    }
  );
}
```

```typescript
// Generate types from your Supabase schema
// npx supabase gen types typescript --project-id <id> > lib/supabase/database.types.ts

import { Database } from "@/lib/supabase/database.types";

type Tables = Database["public"]["Tables"];
type UserRow = Tables["users"]["Row"];
type UserInsert = Tables["users"]["Insert"];
type UserUpdate = Tables["users"]["Update"];
```

---

## 2. Schema Design

### Naming Conventions

```sql
-- Tables: snake_case, plural
CREATE TABLE user_profiles ( ... );
CREATE TABLE order_items ( ... );

-- Columns: snake_case
-- Foreign keys: <singular_table>_id
-- Booleans: is_*, has_*, can_*
-- Timestamps: *_at
-- Counts: *_count

CREATE TABLE orders (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id uuid REFERENCES users(id) NOT NULL,
  status order_status NOT NULL DEFAULT 'pending',
  total_amount numeric(12, 2) NOT NULL DEFAULT 0,
  item_count integer NOT NULL DEFAULT 0,
  is_paid boolean NOT NULL DEFAULT false,
  paid_at timestamptz,
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now(),
  deleted_at timestamptz  -- soft delete
);
```

### UUIDs vs CUID2

```sql
-- UUID v4 — standard, built into PostgreSQL
id uuid PRIMARY KEY DEFAULT gen_random_uuid()

-- UUID v7 — time-ordered, better for indexing (PostgreSQL 17+ or extension)
-- Recommended over v4 when available
id uuid PRIMARY KEY DEFAULT uuidv7()
```

```typescript
// CUID2 — URL-friendly, collision-resistant, generated in app layer
import { createId } from "@paralleldrive/cuid2";

// Use when you need:
// - Shorter IDs for URLs
// - IDs generated client-side before insert
// - No PostgreSQL extension dependency

const id = createId(); // e.g., "clh3am0g40000qjvc1234abcd"
```

**Recommendation:** Use `gen_random_uuid()` (UUID v4) as the default. Switch to CUID2 only for client-generated IDs or URL-friendly slugs.

### JSON Columns

```sql
-- Use jsonb (NOT json) — supports indexing and operators
CREATE TABLE events (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  type text NOT NULL,
  metadata jsonb NOT NULL DEFAULT '{}',
  created_at timestamptz NOT NULL DEFAULT now()
);

-- Index specific JSON paths
CREATE INDEX idx_events_metadata_source ON events ((metadata->>'source'));

-- GIN index for full JSON querying
CREATE INDEX idx_events_metadata ON events USING gin (metadata);
```

```typescript
// Querying JSON with Supabase
const { data } = await supabase
  .from("events")
  .select("*")
  .eq("metadata->>source", "stripe")
  .contains("metadata", { tags: ["important"] });
```

### Enums

```sql
-- PostgreSQL enums — strict at DB level
CREATE TYPE order_status AS ENUM ('pending', 'processing', 'shipped', 'delivered', 'cancelled');

-- Use in table
CREATE TABLE orders (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  status order_status NOT NULL DEFAULT 'pending'
);

-- Adding a value (non-destructive, no downtime)
ALTER TYPE order_status ADD VALUE 'refunded' AFTER 'cancelled';

-- CAUTION: Removing enum values requires recreating the type
-- For frequently changing values, use a CHECK constraint or reference table instead
```

```typescript
// Mirror enums in TypeScript
export const ORDER_STATUS = {
  PENDING: "pending",
  PROCESSING: "processing",
  SHIPPED: "shipped",
  DELIVERED: "delivered",
  CANCELLED: "cancelled",
} as const;

export type OrderStatus = (typeof ORDER_STATUS)[keyof typeof ORDER_STATUS];
```

---

## 3. Seeding

### Seed Script with Faker.js

```typescript
// scripts/seed.ts
import { faker } from "@faker-js/faker";
import { createClient } from "@supabase/supabase-js";

const supabase = createClient(
  process.env.SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY! // bypass RLS for seeding
);

// Deterministic seed for reproducible test data
faker.seed(42);

async function seed() {
  console.log("Seeding database...");

  // Clear existing data (reverse FK order)
  await supabase.from("order_items").delete().neq("id", "");
  await supabase.from("orders").delete().neq("id", "");
  await supabase.from("users").delete().neq("id", "");

  // Seed users
  const users = Array.from({ length: 50 }, () => ({
    email: faker.internet.email().toLowerCase(),
    display_name: faker.person.fullName(),
    avatar_url: faker.image.avatar(),
    is_active: faker.datatype.boolean({ probability: 0.9 }),
  }));

  const { data: insertedUsers, error } = await supabase
    .from("users")
    .insert(users)
    .select("id");

  if (error) throw error;

  // Seed orders referencing real user IDs
  const orders = insertedUsers!.flatMap((user) =>
    Array.from({ length: faker.number.int({ min: 0, max: 5 }) }, () => ({
      user_id: user.id,
      status: faker.helpers.arrayElement([
        "pending", "processing", "shipped", "delivered",
      ]),
      total_amount: parseFloat(faker.commerce.price({ min: 10, max: 500 })),
    }))
  );

  await supabase.from("orders").insert(orders);

  console.log(`Seeded ${users.length} users, ${orders.length} orders`);
}

seed().catch(console.error);
```

### Environment-Aware Seeding

```typescript
// scripts/seed.ts
const ENV = process.env.NODE_ENV ?? "development";

async function seed() {
  if (ENV === "production") {
    console.error("Cannot seed production database.");
    process.exit(1);
  }

  if (ENV === "test") {
    // Minimal deterministic data for tests
    faker.seed(12345);
    await seedMinimal();
  } else {
    // Rich data for development
    faker.seed(42);
    await seedFull();
  }
}
```

```json
// package.json
{
  "scripts": {
    "db:seed": "tsx scripts/seed.ts",
    "db:seed:test": "NODE_ENV=test tsx scripts/seed.ts",
    "db:reset": "supabase db reset && bun run db:seed"
  }
}
```

---

## 4. Connection Pooling

### Supabase Connection Strings

```bash
# Direct connection — for migrations, one-off scripts
# Max connections limited by your plan
DATABASE_URL="postgresql://postgres.[ref]:[password]@aws-0-eu-central-1.pooler.supabase.com:5432/postgres"

# Transaction mode (port 6543) — for serverless/edge functions
# Each query gets its own connection, returned immediately after
# NO: prepared statements, SET, LISTEN/NOTIFY, advisory locks
DATABASE_URL_POOLER="postgresql://postgres.[ref]:[password]@aws-0-eu-central-1.pooler.supabase.com:6543/postgres?pgbouncer=true"

# Session mode (port 5432 via pooler) — for long-lived connections
# Connection persists for the session duration
# Supports prepared statements
DATABASE_URL_SESSION="postgresql://postgres.[ref]:[password]@aws-0-eu-central-1.pooler.supabase.com:5432/postgres"
```

### Serverless Considerations

```typescript
// For Next.js API routes / Server Components — use transaction mode
import postgres from "postgres";

// Connection is reused across hot invocations in the same container
const sql = postgres(process.env.DATABASE_URL_POOLER!, {
  prepare: false, // Required for transaction mode (PgBouncer)
  idle_timeout: 20,
  max_lifetime: 60 * 30, // 30 minutes
});

export default sql;
```

```typescript
// Drizzle with transaction mode pooler
import { drizzle } from "drizzle-orm/postgres-js";
import postgres from "postgres";

const client = postgres(process.env.DATABASE_URL_POOLER!, {
  prepare: false,
});

export const db = drizzle(client);
```

### Connection Limits

```sql
-- Check current connections
SELECT count(*) FROM pg_stat_activity;

-- Check max connections
SHOW max_connections;

-- See connections by state
SELECT state, count(*)
FROM pg_stat_activity
GROUP BY state;
```

---

## 5. Row Level Security (RLS)

### Enable RLS on Every Table

```sql
-- ALWAYS enable RLS — even if you add no policies yet
-- With no policies, ALL access is denied (secure default)
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
```

### Pattern: Own Data

```sql
-- Users can only read/update their own row
CREATE POLICY "users_select_own"
  ON users FOR SELECT
  USING (auth.uid() = id);

CREATE POLICY "users_update_own"
  ON users FOR UPDATE
  USING (auth.uid() = id)
  WITH CHECK (auth.uid() = id);

-- Users can only access their own orders
CREATE POLICY "orders_select_own"
  ON orders FOR SELECT
  USING (auth.uid() = user_id);

CREATE POLICY "orders_insert_own"
  ON orders FOR INSERT
  WITH CHECK (auth.uid() = user_id);
```

### Pattern: Team/Organization Data

```sql
-- Assumes a team_members junction table
CREATE POLICY "team_data_select"
  ON projects FOR SELECT
  USING (
    team_id IN (
      SELECT team_id FROM team_members
      WHERE user_id = auth.uid()
    )
  );

-- Performance optimization: use a security definer function
CREATE OR REPLACE FUNCTION auth.user_team_ids()
RETURNS SETOF uuid
LANGUAGE sql
STABLE
SECURITY DEFINER
SET search_path = ''
AS $$
  SELECT team_id FROM public.team_members WHERE user_id = auth.uid();
$$;

CREATE POLICY "team_data_select_optimized"
  ON projects FOR SELECT
  USING (team_id IN (SELECT auth.user_team_ids()));
```

### Pattern: Admin Bypass

```sql
-- Check user role from JWT/metadata
CREATE OR REPLACE FUNCTION auth.is_admin()
RETURNS boolean
LANGUAGE sql
STABLE
AS $$
  SELECT coalesce(
    (auth.jwt() -> 'app_metadata' ->> 'role') = 'admin',
    false
  );
$$;

-- Admin can see everything, others only own data
CREATE POLICY "orders_select"
  ON orders FOR SELECT
  USING (
    auth.is_admin() OR auth.uid() = user_id
  );
```

### Service Role Bypass

```typescript
// Server-side: service role key bypasses ALL RLS
// Use ONLY in trusted server environments (API routes, background jobs)
import { createClient } from "@supabase/supabase-js";

const supabaseAdmin = createClient(
  process.env.SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY! // NEVER expose to client
);

// This query ignores RLS
const { data } = await supabaseAdmin.from("orders").select("*");
```

### Testing RLS

```sql
-- Test as a specific user
SET request.jwt.claims = '{"sub": "user-uuid-here", "role": "authenticated"}';
SET role = 'authenticated';

-- Should only return the user's own orders
SELECT * FROM orders;

-- Reset
RESET role;
RESET request.jwt.claims;
```

```typescript
// Integration test with Supabase
import { createClient } from "@supabase/supabase-js";

test("user can only see own orders", async () => {
  // Create client with user's JWT
  const supabase = createClient(SUPABASE_URL, SUPABASE_ANON_KEY, {
    global: { headers: { Authorization: `Bearer ${userJwt}` } },
  });

  const { data } = await supabase.from("orders").select("*");
  expect(data!.every((o) => o.user_id === userId)).toBe(true);
});
```

---

## 6. Soft Delete

### Implementation

```sql
-- Add deleted_at to table
ALTER TABLE orders ADD COLUMN deleted_at timestamptz;

-- Create an index for filtering (partial index on non-deleted)
CREATE INDEX idx_orders_active ON orders (created_at)
  WHERE deleted_at IS NULL;

-- View for convenience
CREATE VIEW active_orders AS
  SELECT * FROM orders WHERE deleted_at IS NULL;
```

### Soft Delete Function

```sql
CREATE OR REPLACE FUNCTION soft_delete()
RETURNS trigger
LANGUAGE plpgsql
AS $$
BEGIN
  NEW.deleted_at = now();
  RETURN NEW;
END;
$$;

-- Alternative: intercept DELETE and convert to UPDATE
CREATE OR REPLACE FUNCTION soft_delete_instead()
RETURNS trigger
LANGUAGE plpgsql
AS $$
BEGIN
  UPDATE orders SET deleted_at = now() WHERE id = OLD.id;
  RETURN NULL; -- prevent actual deletion
END;
$$;

CREATE TRIGGER orders_soft_delete
  BEFORE DELETE ON orders
  FOR EACH ROW
  EXECUTE FUNCTION soft_delete_instead();
```

### Querying with Soft Delete (Supabase)

```typescript
// Always filter out deleted records
const { data: activeOrders } = await supabase
  .from("orders")
  .select("*")
  .is("deleted_at", null);

// Soft delete
const { error } = await supabase
  .from("orders")
  .update({ deleted_at: new Date().toISOString() })
  .eq("id", orderId);

// Restore
const { error: restoreError } = await supabase
  .from("orders")
  .update({ deleted_at: null })
  .eq("id", orderId);
```

### RLS-Based Automatic Filtering

```sql
-- Automatically hide soft-deleted records via RLS
CREATE POLICY "hide_deleted_orders"
  ON orders FOR SELECT
  USING (deleted_at IS NULL AND auth.uid() = user_id);

-- Admin policy to see deleted records
CREATE POLICY "admin_see_all_orders"
  ON orders FOR SELECT
  USING (auth.is_admin());
```

### Cascading Soft Delete

```sql
CREATE OR REPLACE FUNCTION cascade_soft_delete_order()
RETURNS trigger
LANGUAGE plpgsql
AS $$
BEGIN
  IF NEW.deleted_at IS NOT NULL AND OLD.deleted_at IS NULL THEN
    UPDATE order_items SET deleted_at = NEW.deleted_at
    WHERE order_id = NEW.id AND deleted_at IS NULL;
  END IF;
  RETURN NEW;
END;
$$;

CREATE TRIGGER orders_cascade_soft_delete
  AFTER UPDATE OF deleted_at ON orders
  FOR EACH ROW
  EXECUTE FUNCTION cascade_soft_delete_order();
```

---

## 7. Pagination

### Offset Pagination (Simple, Suitable for Small Datasets)

```typescript
// Supabase offset pagination
const PAGE_SIZE = 20;

async function getOrders(page: number) {
  const from = page * PAGE_SIZE;
  const to = from + PAGE_SIZE - 1;

  const { data, count } = await supabase
    .from("orders")
    .select("*, user:users(display_name)", { count: "exact" })
    .is("deleted_at", null)
    .order("created_at", { ascending: false })
    .range(from, to);

  return {
    orders: data,
    totalCount: count,
    totalPages: Math.ceil((count ?? 0) / PAGE_SIZE),
    currentPage: page,
  };
}
```

**Drawback:** `OFFSET` scans and discards rows — slow for large offsets. Page 1000 scans 20,000 rows.

### Cursor/Keyset Pagination (Production-Grade)

```typescript
// Cursor-based pagination — consistent performance at any depth
async function getOrders(cursor?: string, limit = 20) {
  let query = supabase
    .from("orders")
    .select("*")
    .is("deleted_at", null)
    .order("created_at", { ascending: false })
    .order("id", { ascending: false }) // tiebreaker
    .limit(limit + 1); // fetch one extra to detect next page

  if (cursor) {
    // cursor = "2024-01-15T10:30:00Z|order-uuid"
    const [cursorDate, cursorId] = cursor.split("|");
    query = query.or(
      `created_at.lt.${cursorDate},and(created_at.eq.${cursorDate},id.lt.${cursorId})`
    );
  }

  const { data } = await query;
  const hasMore = (data?.length ?? 0) > limit;
  const items = data?.slice(0, limit) ?? [];
  const lastItem = items[items.length - 1];

  return {
    orders: items,
    nextCursor: hasMore && lastItem
      ? `${lastItem.created_at}|${lastItem.id}`
      : null,
  };
}
```

### Infinite Scroll with React Query

```typescript
import { useInfiniteQuery } from "@tanstack/react-query";

function useOrders() {
  return useInfiniteQuery({
    queryKey: ["orders"],
    queryFn: async ({ pageParam }) => {
      const res = await fetch(`/api/orders?cursor=${pageParam ?? ""}`);
      return res.json();
    },
    getNextPageParam: (lastPage) => lastPage.nextCursor,
    initialPageParam: undefined as string | undefined,
  });
}

// Component
function OrderList() {
  const { data, fetchNextPage, hasNextPage, isFetchingNextPage } = useOrders();

  return (
    <>
      {data?.pages.flatMap((page) =>
        page.orders.map((order) => <OrderRow key={order.id} order={order} />)
      )}
      {hasNextPage && (
        <button onClick={() => fetchNextPage()} disabled={isFetchingNextPage}>
          {isFetchingNextPage ? "Loading..." : "Load more"}
        </button>
      )}
    </>
  );
}
```

---

## 8. Full-Text Search

### Setup tsvector Column and GIN Index

```sql
-- Add a generated tsvector column
ALTER TABLE products ADD COLUMN fts tsvector
  GENERATED ALWAYS AS (
    setweight(to_tsvector('english', coalesce(name, '')), 'A') ||
    setweight(to_tsvector('english', coalesce(description, '')), 'B')
  ) STORED;

-- GIN index for fast search
CREATE INDEX idx_products_fts ON products USING gin (fts);
```

### Search with Ranking

```sql
-- Basic search with ranking
SELECT
  id, name, description,
  ts_rank(fts, query) AS rank
FROM products, to_tsquery('english', 'wireless & headphone') AS query
WHERE fts @@ query
ORDER BY rank DESC
LIMIT 20;

-- Prefix search (autocomplete)
SELECT * FROM products
WHERE fts @@ to_tsquery('english', 'wire:*');

-- Phrase search
SELECT * FROM products
WHERE fts @@ phraseto_tsquery('english', 'noise cancelling');
```

### Supabase textSearch()

```typescript
// Basic full-text search
const { data } = await supabase
  .from("products")
  .select("id, name, description")
  .textSearch("fts", "wireless headphone", {
    type: "websearch", // supports natural language queries like "wireless OR headphone"
    config: "english",
  })
  .limit(20);

// Combine with filters
const { data: filtered } = await supabase
  .from("products")
  .select("*")
  .textSearch("fts", "headphone", { type: "websearch" })
  .eq("category", "electronics")
  .gte("price", 50)
  .order("created_at", { ascending: false });
```

### Search with Highlights

```sql
-- Return highlighted snippets
SELECT
  id, name,
  ts_headline('english', description, to_tsquery('english', 'wireless & headphone'),
    'StartSel=<mark>, StopSel=</mark>, MaxWords=35, MinWords=15'
  ) AS highlighted_description
FROM products
WHERE fts @@ to_tsquery('english', 'wireless & headphone')
ORDER BY ts_rank(fts, to_tsquery('english', 'wireless & headphone')) DESC;
```

```typescript
// Call via Supabase RPC for highlights
const { data } = await supabase.rpc("search_products", {
  search_query: "wireless headphone",
  result_limit: 20,
});
```

```sql
-- Database function for search with highlights
CREATE OR REPLACE FUNCTION search_products(search_query text, result_limit int DEFAULT 20)
RETURNS TABLE (id uuid, name text, highlighted_description text, rank real)
LANGUAGE sql
STABLE
AS $$
  SELECT
    p.id, p.name,
    ts_headline('english', p.description, websearch_to_tsquery('english', search_query),
      'StartSel=<mark>, StopSel=</mark>, MaxWords=35, MinWords=15'
    ),
    ts_rank(p.fts, websearch_to_tsquery('english', search_query))
  FROM products p
  WHERE p.fts @@ websearch_to_tsquery('english', search_query)
    AND p.deleted_at IS NULL
  ORDER BY ts_rank(p.fts, websearch_to_tsquery('english', search_query)) DESC
  LIMIT result_limit;
$$;
```

---

## 9. Migrations

### Supabase Migration Workflow

```bash
# Generate migration from schema diff (local vs remote)
supabase db diff --schema public -f add_orders_table

# Creates: supabase/migrations/<timestamp>_add_orders_table.sql

# Apply locally
supabase db reset

# Push to remote (staging/production)
supabase db push

# Pull remote schema to local
supabase db pull
```

### Manual Migration File

```sql
-- supabase/migrations/20240115120000_add_order_status.sql

-- Up
ALTER TABLE orders ADD COLUMN status order_status NOT NULL DEFAULT 'pending';
CREATE INDEX idx_orders_status ON orders (status) WHERE deleted_at IS NULL;

-- Add a comment for the rollback (Supabase doesn't support down migrations natively)
-- ROLLBACK: ALTER TABLE orders DROP COLUMN status; DROP INDEX idx_orders_status;
```

### Rollback Strategy

```bash
# Option 1: Create a new migration that reverses the change
supabase migration new rollback_order_status

# Option 2: Repair migration history (development only)
supabase migration repair --status reverted <version>
supabase db reset
```

### Zero-Downtime Migration Patterns

```sql
-- SAFE: Adding a column with a default (PostgreSQL 11+)
-- Does NOT rewrite the table
ALTER TABLE orders ADD COLUMN priority int NOT NULL DEFAULT 0;

-- SAFE: Adding an index concurrently (doesn't lock table)
CREATE INDEX CONCURRENTLY idx_orders_priority ON orders (priority);

-- UNSAFE: Renaming a column — breaks running application
-- Instead: add new column, backfill, update app, drop old column

-- Step 1 (migration 1): Add new column
ALTER TABLE orders ADD COLUMN total_cents bigint;

-- Step 2 (app deploy): Write to BOTH columns, read from new
-- Step 3 (migration 2): Backfill
UPDATE orders SET total_cents = total_amount * 100 WHERE total_cents IS NULL;

-- Step 4 (migration 3): Add NOT NULL, drop old
ALTER TABLE orders ALTER COLUMN total_cents SET NOT NULL;
ALTER TABLE orders DROP COLUMN total_amount;
```

---

## 10. Database Functions & Triggers

### Auto-Update updated_at

```sql
-- Reusable trigger function
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS trigger
LANGUAGE plpgsql
AS $$
BEGIN
  NEW.updated_at = now();
  RETURN NEW;
END;
$$;

-- Apply to any table
CREATE TRIGGER set_updated_at
  BEFORE UPDATE ON users
  FOR EACH ROW
  EXECUTE FUNCTION update_updated_at();

CREATE TRIGGER set_updated_at
  BEFORE UPDATE ON orders
  FOR EACH ROW
  EXECUTE FUNCTION update_updated_at();
```

### Computed Columns (Generated)

```sql
-- Stored generated column — computed on write, indexed
ALTER TABLE users ADD COLUMN full_name text
  GENERATED ALWAYS AS (
    coalesce(first_name, '') || ' ' || coalesce(last_name, '')
  ) STORED;

-- Can be indexed
CREATE INDEX idx_users_full_name ON users (full_name);
```

### Audit Trigger

```sql
-- Audit log table
CREATE TABLE audit_log (
  id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  table_name text NOT NULL,
  record_id uuid NOT NULL,
  action text NOT NULL, -- INSERT, UPDATE, DELETE
  old_data jsonb,
  new_data jsonb,
  changed_by uuid,
  changed_at timestamptz NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_log_record ON audit_log (table_name, record_id);
CREATE INDEX idx_audit_log_changed_at ON audit_log (changed_at);

-- Generic audit function
CREATE OR REPLACE FUNCTION audit_trigger()
RETURNS trigger
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = ''
AS $$
BEGIN
  INSERT INTO public.audit_log (table_name, record_id, action, old_data, new_data, changed_by)
  VALUES (
    TG_TABLE_NAME,
    coalesce(NEW.id, OLD.id),
    TG_OP,
    CASE WHEN TG_OP IN ('UPDATE', 'DELETE') THEN to_jsonb(OLD) END,
    CASE WHEN TG_OP IN ('INSERT', 'UPDATE') THEN to_jsonb(NEW) END,
    auth.uid()
  );

  RETURN coalesce(NEW, OLD);
END;
$$;

-- Attach to tables
CREATE TRIGGER audit_orders
  AFTER INSERT OR UPDATE OR DELETE ON orders
  FOR EACH ROW
  EXECUTE FUNCTION audit_trigger();
```

### Auto-Increment Counter

```sql
-- Automatically update item_count on parent when children change
CREATE OR REPLACE FUNCTION update_order_item_count()
RETURNS trigger
LANGUAGE plpgsql
AS $$
BEGIN
  IF TG_OP = 'DELETE' THEN
    UPDATE orders SET item_count = (
      SELECT count(*) FROM order_items WHERE order_id = OLD.order_id AND deleted_at IS NULL
    ) WHERE id = OLD.order_id;
    RETURN OLD;
  ELSE
    UPDATE orders SET item_count = (
      SELECT count(*) FROM order_items WHERE order_id = NEW.order_id AND deleted_at IS NULL
    ) WHERE id = NEW.order_id;
    RETURN NEW;
  END IF;
END;
$$;

CREATE TRIGGER order_items_count
  AFTER INSERT OR UPDATE OR DELETE ON order_items
  FOR EACH ROW
  EXECUTE FUNCTION update_order_item_count();
```

---

## 11. Indexing

### When to Add Indexes

- **Always index:** foreign keys, columns in `WHERE`, `JOIN`, `ORDER BY` on large tables
- **Don't index:** small tables (<1000 rows), rarely queried columns, columns with very low cardinality (e.g., boolean)
- **Consider:** composite indexes when queries filter on multiple columns together

### Index Types

```sql
-- B-tree (default) — equality and range queries
CREATE INDEX idx_orders_user_id ON orders (user_id);

-- Composite — for queries that filter on multiple columns
-- Column order matters: most selective first, or match query patterns
CREATE INDEX idx_orders_user_status ON orders (user_id, status);
-- Covers: WHERE user_id = x AND status = y
-- Covers: WHERE user_id = x (prefix)
-- Does NOT cover: WHERE status = y (no leading column)

-- Partial — index only relevant rows (smaller, faster)
CREATE INDEX idx_orders_pending ON orders (created_at)
  WHERE status = 'pending' AND deleted_at IS NULL;

-- GIN — for arrays, JSONB, full-text search
CREATE INDEX idx_products_tags ON products USING gin (tags);
CREATE INDEX idx_events_metadata ON events USING gin (metadata);

-- BRIN — for naturally ordered data (timestamps on append-only tables)
-- Much smaller than B-tree, good for large time-series tables
CREATE INDEX idx_events_created_at ON events USING brin (created_at);
```

### Unique Indexes

```sql
-- Unique constraint (creates a unique index)
ALTER TABLE users ADD CONSTRAINT users_email_unique UNIQUE (email);

-- Partial unique — unique only among active records
CREATE UNIQUE INDEX idx_users_email_active
  ON users (email)
  WHERE deleted_at IS NULL;
```

### EXPLAIN ANALYZE

```sql
-- Always check query plans for slow queries
EXPLAIN ANALYZE
SELECT * FROM orders
WHERE user_id = '550e8400-e29b-41d4-a716-446655440000'
  AND status = 'pending'
  AND deleted_at IS NULL
ORDER BY created_at DESC
LIMIT 20;

-- Key things to look for:
-- Seq Scan → missing index (on large tables)
-- Index Scan / Index Only Scan → good
-- Bitmap Index Scan → acceptable for multiple conditions
-- Sort → consider adding column to index
-- Rows (estimated vs actual) → if very different, run ANALYZE

-- Force PostgreSQL to update statistics
ANALYZE orders;
```

### Monitoring Index Usage

```sql
-- Find unused indexes (wasting write performance and disk)
SELECT
  schemaname, tablename, indexname,
  idx_scan AS times_used,
  pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indexrelid NOT IN (
    SELECT conindid FROM pg_constraint WHERE contype IN ('p', 'u')
  )
ORDER BY pg_relation_size(indexrelid) DESC;

-- Find missing indexes (tables with many sequential scans)
SELECT
  relname AS table_name,
  seq_scan,
  idx_scan,
  seq_scan - idx_scan AS too_many_seq_scans,
  pg_size_pretty(pg_relation_size(relid)) AS table_size
FROM pg_stat_user_tables
WHERE seq_scan > idx_scan
  AND pg_relation_size(relid) > 100000 -- skip tiny tables
ORDER BY too_many_seq_scans DESC;
```
