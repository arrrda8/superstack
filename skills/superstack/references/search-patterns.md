# Search Patterns Reference

Comprehensive guide for implementing search in Superstack projects — from PostgreSQL full-text search to dedicated search engines, UI patterns, and analytics.

---

## 1. Decision Matrix

| Criteria | PostgreSQL FTS | Meilisearch | Typesense | Algolia |
|---|---|---|---|---|
| **Cost** | Free (included in DB) | Free (self-hosted), Cloud from $30/mo | Free (self-hosted), Cloud from $0.04/1k searches | From $1/1k requests, expensive at scale |
| **Latency** | 5-50ms (depends on data size) | <50ms | <50ms | <50ms (CDN-backed) |
| **Typo tolerance** | Manual (pg_trgm) | Built-in, excellent | Built-in, good | Built-in, excellent |
| **Faceted search** | Manual queries | Built-in | Built-in | Built-in |
| **Self-hosted** | Yes (your DB) | Yes (easy, single binary) | Yes (single binary) | No |
| **Geo search** | PostGIS required | Built-in | Built-in | Built-in |
| **Ranking** | ts_rank, custom | Customizable rules | Customizable rules | Highly customizable |
| **Supabase integration** | Native | Via Edge Functions | Via Edge Functions | Via Edge Functions |
| **Best for** | <100k docs, tight budget, simple search | 100k-10M docs, self-hosted, great DX | 100k-10M docs, self-hosted | Enterprise, global, unlimited budget |

### Recommendation

1. **Start with PostgreSQL FTS** — zero additional infrastructure, works for most apps.
2. **Graduate to Meilisearch** — when you need typo tolerance, facets, or >100k documents.
3. **Consider Algolia** — only for enterprise with global latency requirements and budget.

---

## 2. PostgreSQL Full-Text Search

### Setup with Supabase

```sql
-- Add a tsvector column with GIN index
ALTER TABLE products ADD COLUMN fts tsvector
  GENERATED ALWAYS AS (
    setweight(to_tsvector('german', coalesce(title, '')), 'A') ||
    setweight(to_tsvector('german', coalesce(description, '')), 'B') ||
    setweight(to_tsvector('german', coalesce(category, '')), 'C')
  ) STORED;

CREATE INDEX products_fts_idx ON products USING GIN (fts);
```

### Weighted Search Query

```sql
-- Search with ranking
SELECT
  id,
  title,
  description,
  ts_rank(fts, query) AS rank,
  ts_headline('german', title, query, 'StartSel=<mark>, StopSel=</mark>') AS highlighted_title
FROM products,
  to_tsquery('german', 'marketing & automation') AS query
WHERE fts @@ query
ORDER BY rank DESC
LIMIT 20;
```

### Supabase Client Usage

```typescript
// Basic text search
const { data } = await supabase
  .from("products")
  .select("id, title, description")
  .textSearch("fts", "marketing automation", {
    type: "websearch", // natural language parsing
    config: "german",
  })
  .limit(20);

// With additional filters
const { data } = await supabase
  .from("products")
  .select("id, title, description, category")
  .textSearch("fts", query, { type: "websearch", config: "german" })
  .eq("status", "active")
  .gte("price", minPrice)
  .order("created_at", { ascending: false })
  .limit(20);
```

### Helper: Parse User Input to tsquery

```typescript
function buildSearchQuery(input: string): string {
  // Split input, remove empty strings, join with & for AND logic
  const terms = input
    .trim()
    .split(/\s+/)
    .filter(Boolean)
    .map((term) => `'${term}':*`) // prefix matching
    .join(" & ");
  return terms;
}

// Usage with raw query for prefix matching
const { data } = await supabase.rpc("search_products", {
  search_query: buildSearchQuery(userInput),
});
```

```sql
-- RPC function for advanced search
CREATE OR REPLACE FUNCTION search_products(search_query text, result_limit int DEFAULT 20)
RETURNS TABLE(id uuid, title text, description text, rank real) AS $$
BEGIN
  RETURN QUERY
  SELECT p.id, p.title, p.description, ts_rank(p.fts, to_tsquery('german', search_query)) AS rank
  FROM products p
  WHERE p.fts @@ to_tsquery('german', search_query)
  ORDER BY rank DESC
  LIMIT result_limit;
END;
$$ LANGUAGE plpgsql STABLE;
```

---

## 3. Meilisearch

### Setup (Docker / Dokploy)

```yaml
# docker-compose.yml
services:
  meilisearch:
    image: getmeili/meilisearch:v1.6
    ports:
      - "7700:7700"
    environment:
      MEILI_MASTER_KEY: ${MEILI_MASTER_KEY}
      MEILI_ENV: production
    volumes:
      - meili_data:/meili_data
```

### Index Configuration

```typescript
import { MeiliSearch } from "meilisearch";

const client = new MeiliSearch({
  host: process.env.MEILISEARCH_HOST!,
  apiKey: process.env.MEILISEARCH_API_KEY!,
});

// Create and configure index
const index = client.index("products");

await index.updateSettings({
  searchableAttributes: ["title", "description", "category", "tags"],
  filterableAttributes: ["category", "status", "price", "created_at"],
  sortableAttributes: ["price", "created_at", "title"],
  rankingRules: [
    "words",
    "typo",
    "proximity",
    "attribute",
    "sort",
    "exactness",
  ],
  typoTolerance: {
    minWordSizeForTypos: { oneTypo: 4, twoTypos: 8 },
  },
  pagination: { maxTotalHits: 1000 },
});
```

### Indexing Documents

```typescript
// Bulk index
const products = await supabase.from("products").select("*").eq("status", "active");
await index.addDocuments(products.data!, { primaryKey: "id" });

// Partial update (only changed fields)
await index.updateDocuments([
  { id: "product-123", title: "Updated Title", price: 29.99 },
]);

// Delete
await index.deleteDocument("product-123");
```

### Search with Filters and Facets

```typescript
const results = await index.search("marketing tool", {
  filter: ['category = "SaaS"', "price < 100"],
  facets: ["category", "status"],
  sort: ["price:asc"],
  limit: 20,
  offset: 0,
  attributesToHighlight: ["title", "description"],
  highlightPreTag: "<mark>",
  highlightPostTag: "</mark>",
});

// results.hits — matched documents
// results.facetDistribution — { category: { SaaS: 12, Tool: 5 }, status: { active: 15, draft: 2 } }
// results.estimatedTotalHits — total matching count
```

### Next.js API Route

```typescript
// app/api/search/route.ts
import { MeiliSearch } from "meilisearch";
import { NextRequest, NextResponse } from "next/server";

const client = new MeiliSearch({
  host: process.env.MEILISEARCH_HOST!,
  apiKey: process.env.MEILISEARCH_API_KEY!,
});

export async function GET(request: NextRequest) {
  const { searchParams } = new URL(request.url);
  const q = searchParams.get("q") || "";
  const category = searchParams.get("category");
  const page = parseInt(searchParams.get("page") || "1");
  const limit = 20;

  const filters: string[] = [];
  if (category) filters.push(`category = "${category}"`);

  const results = await client.index("products").search(q, {
    filter: filters,
    facets: ["category", "status"],
    limit,
    offset: (page - 1) * limit,
    attributesToHighlight: ["title", "description"],
  });

  return NextResponse.json(results);
}
```

---

## 4. Autocomplete / Typeahead

### Debounced Search Hook

```typescript
import { useCallback, useEffect, useRef, useState } from "react";

interface UseAutocompleteOptions<T> {
  search: (query: string) => Promise<T[]>;
  debounceMs?: number;
  minChars?: number;
}

function useAutocomplete<T>({ search, debounceMs = 300, minChars = 2 }: UseAutocompleteOptions<T>) {
  const [query, setQuery] = useState("");
  const [results, setResults] = useState<T[]>([]);
  const [loading, setLoading] = useState(false);
  const [isOpen, setIsOpen] = useState(false);
  const abortRef = useRef<AbortController | null>(null);

  const debouncedSearch = useCallback(
    debounce(async (q: string) => {
      if (q.length < minChars) {
        setResults([]);
        setIsOpen(false);
        return;
      }

      // Cancel previous request
      abortRef.current?.abort();
      abortRef.current = new AbortController();

      setLoading(true);
      try {
        const data = await search(q);
        setResults(data);
        setIsOpen(data.length > 0);
      } catch (err) {
        if (err instanceof DOMException && err.name === "AbortError") return;
        setResults([]);
      } finally {
        setLoading(false);
      }
    }, debounceMs),
    [search, debounceMs, minChars]
  );

  useEffect(() => {
    debouncedSearch(query);
  }, [query, debouncedSearch]);

  return { query, setQuery, results, loading, isOpen, setIsOpen };
}
```

### Highlight Matches

```typescript
function highlightMatches(text: string, query: string): React.ReactNode {
  if (!query.trim()) return text;

  const regex = new RegExp(`(${escapeRegExp(query)})`, "gi");
  const parts = text.split(regex);

  return parts.map((part, i) =>
    regex.test(part) ? <mark key={i} className="bg-yellow-200 dark:bg-yellow-800">{part}</mark> : part
  );
}

function escapeRegExp(string: string): string {
  return string.replace(/[.*+?^${}()|[\]\\]/g, "\\$&");
}
```

### Keyboard Navigation

```typescript
function useKeyboardNavigation(items: any[], onSelect: (item: any) => void) {
  const [activeIndex, setActiveIndex] = useState(-1);

  const handleKeyDown = useCallback(
    (e: React.KeyboardEvent) => {
      switch (e.key) {
        case "ArrowDown":
          e.preventDefault();
          setActiveIndex((prev) => Math.min(prev + 1, items.length - 1));
          break;
        case "ArrowUp":
          e.preventDefault();
          setActiveIndex((prev) => Math.max(prev - 1, 0));
          break;
        case "Enter":
          e.preventDefault();
          if (activeIndex >= 0 && items[activeIndex]) {
            onSelect(items[activeIndex]);
          }
          break;
        case "Escape":
          setActiveIndex(-1);
          break;
      }
    },
    [items, activeIndex, onSelect]
  );

  // Reset when items change
  useEffect(() => setActiveIndex(-1), [items]);

  return { activeIndex, handleKeyDown };
}
```

### Recent Searches (localStorage)

```typescript
const RECENT_KEY = "recent-searches";
const MAX_RECENT = 5;

function getRecentSearches(): string[] {
  if (typeof window === "undefined") return [];
  const stored = localStorage.getItem(RECENT_KEY);
  return stored ? JSON.parse(stored) : [];
}

function addRecentSearch(query: string): void {
  const recent = getRecentSearches().filter((q) => q !== query);
  recent.unshift(query);
  localStorage.setItem(RECENT_KEY, JSON.stringify(recent.slice(0, MAX_RECENT)));
}

function clearRecentSearches(): void {
  localStorage.removeItem(RECENT_KEY);
}
```

---

## 5. Fuzzy Search

### PostgreSQL pg_trgm Extension

```sql
-- Enable extension (Supabase: enabled by default)
CREATE EXTENSION IF NOT EXISTS pg_trgm;

-- Create trigram index
CREATE INDEX products_title_trgm_idx ON products USING GIN (title gin_trgm_ops);
CREATE INDEX products_description_trgm_idx ON products USING GIN (description gin_trgm_ops);

-- Similarity search (finds typos, partial matches)
SELECT id, title, similarity(title, 'marketng') AS sim
FROM products
WHERE similarity(title, 'marketng') > 0.3
ORDER BY sim DESC
LIMIT 10;

-- Trigram search with LIKE pattern (uses GIN index)
SELECT * FROM products
WHERE title % 'marketng'  -- similarity operator
ORDER BY title <-> 'marketng'  -- distance operator
LIMIT 10;
```

### Levenshtein Distance

```sql
-- Enable fuzzystrmatch extension
CREATE EXTENSION IF NOT EXISTS fuzzystrmatch;

-- Levenshtein distance (edit distance)
SELECT id, title, levenshtein(lower(title), lower('marketng')) AS distance
FROM products
WHERE levenshtein(lower(title), lower('marketng')) <= 2
ORDER BY distance ASC
LIMIT 10;
```

### When to Use Which

| Method | Best For | Index Support | Performance |
|---|---|---|---|
| **pg_trgm similarity** | General fuzzy matching, typo tolerance | GIN index, fast | Good for <1M rows |
| **Levenshtein** | Exact edit distance calculation, short strings | No index, slow scan | Small datasets only |
| **Full-text search** | Natural language queries, stemming, ranking | GIN index, fast | Best for large text bodies |

### Supabase RPC for Combined Search

```sql
-- Combines FTS with trigram fallback
CREATE OR REPLACE FUNCTION smart_search(search_input text, result_limit int DEFAULT 20)
RETURNS TABLE(id uuid, title text, description text, score real, match_type text) AS $$
BEGIN
  -- First: try full-text search
  RETURN QUERY
  SELECT p.id, p.title, p.description,
    ts_rank(p.fts, websearch_to_tsquery('german', search_input)) AS score,
    'fts'::text AS match_type
  FROM products p
  WHERE p.fts @@ websearch_to_tsquery('german', search_input)
  ORDER BY score DESC
  LIMIT result_limit;

  -- If no FTS results, fall back to trigram similarity
  IF NOT FOUND THEN
    RETURN QUERY
    SELECT p.id, p.title, p.description,
      similarity(p.title, search_input) AS score,
      'fuzzy'::text AS match_type
    FROM products p
    WHERE similarity(p.title, search_input) > 0.2
    ORDER BY score DESC
    LIMIT result_limit;
  END IF;
END;
$$ LANGUAGE plpgsql STABLE;
```

---

## 6. Faceted Search

### Filter Sidebar Component

```typescript
interface FacetGroup {
  key: string;
  label: string;
  type: "checkbox" | "range" | "radio";
  options: FacetOption[];
}

interface FacetOption {
  value: string;
  label: string;
  count: number;
}

interface FacetedSearchState {
  query: string;
  filters: Record<string, string[]>; // { category: ["SaaS", "Tool"], status: ["active"] }
  sort: string;
  page: number;
}
```

### URL State Sync with nuqs

```typescript
import { parseAsArrayOf, parseAsInteger, parseAsString, useQueryStates } from "nuqs";

function useSearchParams() {
  const [params, setParams] = useQueryStates({
    q: parseAsString.withDefault(""),
    category: parseAsArrayOf(parseAsString).withDefault([]),
    status: parseAsArrayOf(parseAsString).withDefault([]),
    sort: parseAsString.withDefault("relevance"),
    page: parseAsInteger.withDefault(1),
    minPrice: parseAsInteger,
    maxPrice: parseAsInteger,
  });

  const toggleFilter = (key: string, value: string) => {
    const current = (params as any)[key] as string[];
    const next = current.includes(value)
      ? current.filter((v: string) => v !== value)
      : [...current, value];
    setParams({ [key]: next, page: 1 }); // reset page on filter change
  };

  return { params, setParams, toggleFilter };
}
```

### PostgreSQL Facet Counts

```sql
-- Get facet counts for active filters
SELECT
  category,
  COUNT(*) AS count
FROM products
WHERE status = 'active'
  AND (title_fts @@ websearch_to_tsquery('german', $1) OR $1 IS NULL)
GROUP BY category
ORDER BY count DESC;

-- Multiple facets in one query
SELECT
  'category' AS facet,
  category AS value,
  COUNT(*) AS count
FROM products WHERE status = 'active'
GROUP BY category
UNION ALL
SELECT
  'status' AS facet,
  status AS value,
  COUNT(*) AS count
FROM products
GROUP BY status
ORDER BY facet, count DESC;
```

### Meilisearch Facets (Built-in)

```typescript
const results = await index.search(query, {
  facets: ["category", "status", "price_range"],
  filter: activeFilters, // ["category = 'SaaS'"]
});

// results.facetDistribution = {
//   category: { SaaS: 42, Tool: 18, Service: 7 },
//   status: { active: 55, draft: 12 },
//   price_range: { "0-50": 30, "50-100": 20, "100+": 17 }
// }

// Use facetDistribution to render filter sidebar with counts
```

---

## 7. Search UI

### Cmd+K Search Dialog

```typescript
"use client";

import { useEffect, useState } from "react";

export function SearchDialog() {
  const [open, setOpen] = useState(false);

  // Global keyboard shortcut
  useEffect(() => {
    const handleKeyDown = (e: KeyboardEvent) => {
      if ((e.metaKey || e.ctrlKey) && e.key === "k") {
        e.preventDefault();
        setOpen((prev) => !prev);
      }
    };
    document.addEventListener("keydown", handleKeyDown);
    return () => document.removeEventListener("keydown", handleKeyDown);
  }, []);

  return (
    <>
      {/* Trigger button in navbar */}
      <button
        onClick={() => setOpen(true)}
        className="flex items-center gap-2 rounded-md border px-3 py-1.5 text-sm text-muted-foreground hover:bg-accent"
      >
        <SearchIcon className="h-4 w-4" />
        <span>Search...</span>
        <kbd className="ml-auto rounded bg-muted px-1.5 py-0.5 text-xs">
          {navigator.platform.includes("Mac") ? "Cmd" : "Ctrl"}+K
        </kbd>
      </button>

      {/* Command dialog */}
      <CommandDialog open={open} onOpenChange={setOpen}>
        {/* ... search input, results, sections */}
      </CommandDialog>
    </>
  );
}
```

### Results Page Layout

```typescript
interface SearchPageProps {
  query: string;
  results: SearchResult[];
  facets: FacetGroup[];
  totalHits: number;
  page: number;
  totalPages: number;
}

// Layout structure:
// ┌──────────────────────────────────────┐
// │ Search bar (sticky top)              │
// ├──────────┬───────────────────────────┤
// │ Filters  │ Results header (count,    │
// │ sidebar  │ sort dropdown)            │
// │          │───────────────────────────│
// │ Category │ Result card               │
// │ □ SaaS   │ Result card               │
// │ □ Tool   │ Result card               │
// │          │ ...                        │
// │ Status   │───────────────────────────│
// │ □ Active │ Pagination                │
// │ □ Draft  │                           │
// └──────────┴───────────────────────────┘
```

### No Results State

```typescript
function NoResults({ query, suggestions }: { query: string; suggestions?: string[] }) {
  return (
    <div className="flex flex-col items-center justify-center py-16 text-center">
      <SearchXIcon className="h-12 w-12 text-muted-foreground/50" />
      <h3 className="mt-4 text-lg font-semibold">No results for "{query}"</h3>
      <p className="mt-1 text-sm text-muted-foreground">
        Try adjusting your search or filters.
      </p>
      {suggestions && suggestions.length > 0 && (
        <div className="mt-4">
          <p className="text-sm text-muted-foreground">Did you mean:</p>
          <div className="mt-2 flex gap-2">
            {suggestions.map((s) => (
              <button key={s} className="rounded-full border px-3 py-1 text-sm hover:bg-accent">
                {s}
              </button>
            ))}
          </div>
        </div>
      )}
    </div>
  );
}
```

---

## 8. Indexing Strategy

### What to Index

```typescript
// Define which fields to index and their purpose
interface IndexableDocument {
  // Primary key
  id: string;

  // Searchable (full-text)
  title: string;
  description: string;
  content: string;
  tags: string[];

  // Filterable (exact match)
  category: string;
  status: "active" | "draft" | "archived";
  author_id: string;

  // Sortable
  created_at: string; // ISO date
  updated_at: string;
  price: number;
  popularity: number;

  // Display only (not searchable)
  thumbnail_url: string;
  slug: string;
}
```

### When to Re-index

| Trigger | Strategy | Implementation |
|---|---|---|
| Record created | Index single document | Supabase webhook -> n8n -> Meilisearch API |
| Record updated | Partial update | Supabase webhook -> n8n -> Meilisearch `updateDocuments` |
| Record deleted | Delete from index | Supabase webhook -> n8n -> Meilisearch `deleteDocument` |
| Schema change | Full re-index | Manual or scheduled job |
| Bulk import | Batch index | Chunk into batches of 1000, use `addDocuments` |

### Webhook-Triggered Indexing (Supabase + n8n)

```sql
-- Supabase: create a webhook trigger
CREATE OR REPLACE FUNCTION notify_search_index()
RETURNS trigger AS $$
BEGIN
  PERFORM pg_notify('search_index', json_build_object(
    'table', TG_TABLE_NAME,
    'operation', TG_OP,
    'id', COALESCE(NEW.id, OLD.id),
    'record', CASE WHEN TG_OP = 'DELETE' THEN row_to_json(OLD) ELSE row_to_json(NEW) END
  )::text);
  RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER products_search_trigger
  AFTER INSERT OR UPDATE OR DELETE ON products
  FOR EACH ROW EXECUTE FUNCTION notify_search_index();
```

```typescript
// n8n: Webhook node receives Supabase event, then:
// 1. Parse operation type
// 2. Transform record to indexable document
// 3. Call Meilisearch API accordingly

// Alternative: Supabase Edge Function
Deno.serve(async (req) => {
  const { type, record, old_record } = await req.json();
  const client = new MeiliSearch({ host: MEILI_HOST, apiKey: MEILI_KEY });
  const index = client.index("products");

  switch (type) {
    case "INSERT":
    case "UPDATE":
      await index.addDocuments([transformToSearchDoc(record)]);
      break;
    case "DELETE":
      await index.deleteDocument(old_record.id);
      break;
  }

  return new Response("ok");
});
```

### Scheduled Full Re-index

```typescript
// Cron job (n8n Schedule Trigger or Supabase pg_cron)
async function fullReindex() {
  const client = new MeiliSearch({ host: MEILI_HOST, apiKey: MEILI_KEY });
  const index = client.index("products");

  // Fetch all active records in batches
  let offset = 0;
  const batchSize = 1000;

  while (true) {
    const { data, error } = await supabase
      .from("products")
      .select("*")
      .eq("status", "active")
      .range(offset, offset + batchSize - 1);

    if (!data || data.length === 0) break;

    await index.addDocuments(data.map(transformToSearchDoc));
    offset += batchSize;
  }

  console.log(`Re-indexed ${offset} documents`);
}
```

---

## 9. Search Analytics

### Schema

```sql
CREATE TABLE search_events (
  id uuid DEFAULT gen_random_uuid() PRIMARY KEY,
  query text NOT NULL,
  results_count int NOT NULL,
  filters jsonb DEFAULT '{}',
  user_id uuid REFERENCES auth.users(id),
  session_id text,
  clicked_result_id uuid, -- NULL if no click
  clicked_position int, -- position in results (1-based)
  created_at timestamptz DEFAULT now()
);

CREATE INDEX search_events_query_idx ON search_events (query, created_at);
CREATE INDEX search_events_created_idx ON search_events (created_at);

-- Enable RLS
ALTER TABLE search_events ENABLE ROW LEVEL SECURITY;

-- Policy: users can insert their own events
CREATE POLICY "Users can insert own search events"
  ON search_events FOR INSERT
  WITH CHECK (auth.uid() = user_id);
```

### Track Search Events

```typescript
async function trackSearch(params: {
  query: string;
  resultsCount: number;
  filters?: Record<string, string[]>;
}) {
  await supabase.from("search_events").insert({
    query: params.query.toLowerCase().trim(),
    results_count: params.resultsCount,
    filters: params.filters || {},
    user_id: (await supabase.auth.getUser()).data.user?.id,
    session_id: getSessionId(),
  });
}

async function trackClick(params: {
  query: string;
  resultId: string;
  position: number;
}) {
  await supabase.from("search_events").insert({
    query: params.query.toLowerCase().trim(),
    results_count: -1, // indicates click event
    clicked_result_id: params.resultId,
    clicked_position: params.position,
    user_id: (await supabase.auth.getUser()).data.user?.id,
    session_id: getSessionId(),
  });
}
```

### Analytics Queries

```sql
-- Top searches (last 30 days)
SELECT query, COUNT(*) AS search_count, AVG(results_count) AS avg_results
FROM search_events
WHERE created_at > now() - interval '30 days'
  AND results_count >= 0
GROUP BY query
ORDER BY search_count DESC
LIMIT 20;

-- Zero-result queries (opportunity to improve content or synonyms)
SELECT query, COUNT(*) AS occurrences
FROM search_events
WHERE results_count = 0
  AND created_at > now() - interval '30 days'
GROUP BY query
ORDER BY occurrences DESC
LIMIT 20;

-- Click-through rate per query
WITH searches AS (
  SELECT query, COUNT(*) AS total_searches
  FROM search_events
  WHERE results_count >= 0 AND created_at > now() - interval '30 days'
  GROUP BY query
),
clicks AS (
  SELECT query, COUNT(*) AS total_clicks
  FROM search_events
  WHERE clicked_result_id IS NOT NULL AND created_at > now() - interval '30 days'
  GROUP BY query
)
SELECT
  s.query,
  s.total_searches,
  COALESCE(c.total_clicks, 0) AS total_clicks,
  ROUND(COALESCE(c.total_clicks, 0)::numeric / s.total_searches * 100, 1) AS ctr_pct
FROM searches s
LEFT JOIN clicks c ON s.query = c.query
ORDER BY s.total_searches DESC
LIMIT 20;

-- Average click position (lower = better relevance)
SELECT
  query,
  COUNT(*) AS clicks,
  ROUND(AVG(clicked_position), 1) AS avg_position
FROM search_events
WHERE clicked_result_id IS NOT NULL
  AND created_at > now() - interval '30 days'
GROUP BY query
HAVING COUNT(*) >= 5
ORDER BY avg_position ASC
LIMIT 20;
```

### Improving Relevance from Analytics

```typescript
// Use zero-result queries to build a synonym map
const synonyms: Record<string, string[]> = {
  "crm": ["customer relationship", "sales tool", "contact management"],
  "ai": ["artificial intelligence", "machine learning", "ki"],
  "dashboard": ["analytics", "reporting", "overview"],
};

// Meilisearch: configure synonyms
await index.updateSettings({
  synonyms: {
    crm: ["customer relationship management", "sales tool"],
    ai: ["artificial intelligence", "ki"],
  },
});

// PostgreSQL: expand query with synonyms
function expandQuery(query: string, synonymMap: Record<string, string[]>): string {
  let expanded = query;
  for (const [term, synonyms] of Object.entries(synonymMap)) {
    if (query.toLowerCase().includes(term)) {
      expanded += " | " + synonyms.join(" | ");
    }
  }
  return expanded;
}
```
