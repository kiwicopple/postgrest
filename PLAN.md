# Reactive Queries via Database Introspection Extension

## Motivation

Supabase Realtime enables reactive queries by subscribing to database changes. However, when queries involve relationships (joins), the subscription logic becomes complex:

```javascript
const data = supabase
  .from('users')
  .eq('id', '10')
  .select(`
    id, name,
    organizations (
      id, name,
      projects (
        id, name
      )
    )
  `)
```

To make this query "reactive," we need subscriptions to:

1. **Direct rows**: The specific rows returned (`users:id=10`, `organizations:id=X`, etc.)
2. **Foreign key relationships**: Changes that could add/remove related rows
3. **Junction tables**: For M2M relationships, the intermediate table that links entities

### The M2M Problem

Consider `users` ↔ `organizations` linked via a `members` junction table:

```
users (id) ←─── members (user_id, org_id) ───→ organizations (id)
```

A naive implementation might assume a direct FK `organizations.user_id`, but this is incorrect. We must subscribe to the `members` table to detect when a user is added/removed from an organization.

**Required subscriptions for the example query:**

```javascript
subscriptions = [
  // Direct rows
  "public:users:id.eq.10",
  "public:organizations:id.eq.org1",
  "public:projects:id.eq.proj1",
  "public:projects:id.eq.proj2",

  // FK relationships (for detecting new related rows)
  "public:members:user_id.eq.10",           // Junction table!
  "public:members:org_id.eq.org1",          // Junction table!
  "public:projects:organization_id.eq.org1",
]
```

### Why Extract to PL/pgSQL?

PostgREST already performs sophisticated database introspection to understand relationships. This logic is currently in Haskell (`src/PostgREST/SchemaCache.hs`). By extracting it to a PL/pgSQL extension:

1. **Shared functionality**: Both PostgREST and Supabase Realtime can use the same relationship mapping
2. **Consistency**: Single source of truth for relationship metadata
3. **Accessibility**: Any PostgreSQL client can query relationship information
4. **Performance**: Introspection happens at the database level, cacheable via PostgreSQL

---

## Goals

### Primary Goals

1. **Create a PL/pgSQL extension** (`pg_schema_relationships` or similar) that exposes:
   - Table/view metadata
   - Foreign key relationships (M2O, O2M, O2O)
   - Many-to-many relationships with junction table detection
   - View key dependencies
   - Computed relationships (function-based)

2. **Design an API** for querying subscription metadata given a query structure

3. **Integrate with PostgREST** via Link headers to expose subscription metadata

4. **Enable supabase-js** to build reactive queries with `useReactive()`

### Secondary Goals

- Maintain backwards compatibility with existing PostgREST behavior
- Minimize performance overhead
- Support schema-qualified names for multi-tenant scenarios
- Handle complex scenarios (views, computed columns, partitioned tables)

---

## Current State: PostgREST Introspection

### Key Files

| File | Purpose |
|------|---------|
| `src/PostgREST/SchemaCache.hs` | Main introspection queries |
| `src/PostgREST/SchemaCache/Relationship.hs` | Relationship data types |
| `src/PostgREST/SchemaCache/Table.hs` | Table/column structures |

### Introspection Functions to Extract

#### 1. `allTables` (SchemaCache.hs:585)

Queries tables, views, materialized views with:
- Column names and types
- Primary keys
- Nullability, defaults
- Insertable/updatable status

**System catalogs used:**
- `pg_class`, `pg_namespace`, `pg_attribute`
- `pg_constraint`, `pg_type`, `pg_attrdef`

#### 2. `allM2OandO2ORels` (SchemaCache.hs:733)

Discovers foreign key relationships:

```sql
-- Simplified version of the query
SELECT
  c.conname,
  ns1.nspname AS constraint_schema,
  tab.relname AS constraint_table,
  ARRAY_AGG(att1.attname) AS columns,
  ns2.nspname AS foreign_schema,
  other.relname AS foreign_table,
  ARRAY_AGG(att2.attname) AS foreign_columns
FROM pg_constraint c
JOIN pg_class tab ON tab.oid = c.conrelid
JOIN pg_class other ON other.oid = c.confrelid
-- ... additional joins for column names
WHERE c.contype = 'f'
GROUP BY ...
```

**Distinguishing O2O from M2O:**
- If FK columns are a subset of the table's PK → One-to-One
- Otherwise → Many-to-One

#### 3. `addM2MRels` (SchemaCache.hs:554)

Detects junction tables for M2M relationships:

**Junction table criteria:**
1. Has 2+ foreign key constraints
2. FK columns are part of the primary key
3. Links two other tables

```
Table A ←── Junction ──→ Table B
   ↑           ↑            ↑
   PK         PK+FK        PK
```

#### 4. `allViewsKeyDependencies` (SchemaCache.hs:823)

Traces view dependencies to underlying tables using `pg_rewrite`. This allows views to inherit relationships from their base tables.

#### 5. `allComputedRels` (SchemaCache.hs:777)

Discovers function-based relationships:

```sql
SELECT * FROM users u
JOIN LATERAL get_user_preferences(u.id) prefs ON true
```

---

## Proposed Architecture

### Extension: `pg_schema_relationships`

```
pg_schema_relationships/
├── pg_schema_relationships.control
├── sql/
│   ├── pg_schema_relationships--1.0.sql
│   ├── types.sql              -- Custom types for relationships
│   ├── tables.sql             -- Table introspection
│   ├── relationships.sql      -- FK/M2M/computed relationships
│   ├── views.sql              -- View dependency tracking
│   └── subscriptions.sql      -- Subscription metadata generation
└── test/
    └── sql/
        └── test_relationships.sql
```

### Core Types

```sql
-- Relationship cardinality
CREATE TYPE rel_cardinality AS ENUM ('o2o', 'o2m', 'm2o', 'm2m');

-- Relationship definition
CREATE TYPE relationship AS (
  constraint_name    text,
  from_schema        text,
  from_table         text,
  from_columns       text[],
  to_schema          text,
  to_table           text,
  to_columns         text[],
  cardinality        rel_cardinality,
  junction_schema    text,    -- NULL unless M2M
  junction_table     text,    -- NULL unless M2M
  junction_from_cols text[],  -- NULL unless M2M
  junction_to_cols   text[]   -- NULL unless M2M
);

-- Subscription filter
CREATE TYPE subscription_filter AS (
  schema_name  text,
  table_name   text,
  column_name  text,
  operator     text,
  value        text
);
```

### Core Functions

#### 1. `get_relationships(schemas text[])`

Returns all relationships for the specified schemas.

```sql
CREATE FUNCTION get_relationships(schemas text[] DEFAULT ARRAY['public'])
RETURNS SETOF relationship
LANGUAGE sql STABLE
AS $$
  -- Combines M2O, O2O, O2M, and M2M relationships
  -- Ported from PostgREST's allM2OandO2ORels and addM2MRels
$$;
```

#### 2. `get_subscription_filters(query_info jsonb)`

Given a query structure, returns the subscription filters needed for reactivity.

```sql
CREATE FUNCTION get_subscription_filters(query_info jsonb)
RETURNS SETOF subscription_filter
LANGUAGE plpgsql STABLE
AS $$
  -- Analyzes the query structure
  -- Identifies all tables involved
  -- Determines FK paths and junction tables
  -- Returns subscription filters
$$;
```

**Input format:**

```json
{
  "schema": "public",
  "table": "users",
  "filters": [{"column": "id", "op": "eq", "value": "10"}],
  "select": [
    "id",
    "name",
    {
      "relation": "organizations",
      "select": [
        "id",
        "name",
        {
          "relation": "projects",
          "select": ["id", "name"]
        }
      ]
    }
  ]
}
```

**Output:**

```sql
('public', 'users', 'id', 'eq', '10')
('public', 'members', 'user_id', 'eq', '10')
('public', 'organizations', 'id', 'in', '(org1,org2)')
('public', 'members', 'org_id', 'in', '(org1,org2)')
('public', 'projects', 'organization_id', 'in', '(org1,org2)')
```

#### 3. `get_table_primary_key(schema_name text, table_name text)`

Returns the primary key columns for a table.

#### 4. `get_junction_tables(schemas text[])`

Returns all detected junction tables and what they connect.

---

## PostgREST Integration

### Link Header for Subscription Metadata

Following RFC 8288 (Web Linking), PostgREST will return a `Link` header pointing to subscription metadata:

```http
GET /users?id=eq.10&select=id,name,organizations(id,name,projects(id,name))

HTTP/1.1 200 OK
Content-Type: application/json
Link: </rpc/get_subscription_filters?query_hash=abc123>; rel="subscription"

[{"id": "10", "name": "Alice", "organizations": [...]}]
```

### Implementation Steps for PostgREST

1. **Add configuration option**: `server-subscription-metadata = true`

2. **Generate query hash**: Create a deterministic hash of the query structure

3. **Store query structure**: Temporarily cache the query structure by hash

4. **Return Link header**: Include the subscription endpoint URL

5. **Expose RPC endpoint**: Either via the extension or a built-in endpoint

### Alternative: Inline Metadata via Custom Header

For simpler integration, return metadata directly:

```http
GET /users?id=eq.10&select=...

HTTP/1.1 200 OK
Content-Type: application/json
X-Subscription-Filters: [{"schema":"public","table":"users","filter":"id.eq.10"},...]

[...]
```

---

## supabase-js Integration

### `useReactive()` Hook

```typescript
import { useReactive } from '@supabase/supabase-js'

function UserOrganizations({ userId }) {
  const { data, error, isLoading } = useReactive(
    supabase
      .from('users')
      .select(`
        id, name,
        organizations (
          id, name,
          projects (id, name)
        )
      `)
      .eq('id', userId)
  )

  // Component automatically re-renders when:
  // - The user row changes
  // - User is added/removed from an organization (via members table)
  // - Organization details change
  // - Projects are added/removed from organizations

  return <div>...</div>
}
```

### Implementation Flow

```
┌─────────────────┐
│  useReactive()  │
└────────┬────────┘
         │
         ▼
┌─────────────────────────────────────┐
│  1. Execute query via PostgREST     │
│  2. Parse Link header               │
│  3. Fetch subscription metadata     │
└────────┬────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────┐
│  4. Create Realtime subscriptions   │
│     for each filter                 │
└────────┬────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────┐
│  5. On change event:                │
│     - Re-fetch query                │
│     - Update subscriptions if       │
│       result set changed            │
└─────────────────────────────────────┘
```

### Subscription Management

```typescript
class ReactiveQuery {
  private subscriptions: RealtimeChannel[] = []
  private filters: SubscriptionFilter[] = []

  async subscribe(queryBuilder: PostgrestQueryBuilder) {
    // Execute query
    const { data, error } = await queryBuilder

    // Get subscription metadata from Link header
    const metadataUrl = this.extractLinkHeader(response)
    const filters = await fetch(metadataUrl).then(r => r.json())

    // Create subscriptions
    for (const filter of filters) {
      const channel = supabase
        .channel(`${filter.schema}:${filter.table}`)
        .on(
          'postgres_changes',
          {
            event: '*',
            schema: filter.schema,
            table: filter.table,
            filter: `${filter.column}=${filter.operator}.${filter.value}`
          },
          () => this.handleChange()
        )
        .subscribe()

      this.subscriptions.push(channel)
    }
  }

  private async handleChange() {
    // Re-fetch query
    // Diff subscriptions
    // Add/remove subscriptions as needed
  }
}
```

---

## Implementation Plan

### Phase 1: PL/pgSQL Extension Core

**Tasks:**

1. [ ] Set up extension scaffolding
2. [ ] Port `allTables` query to PL/pgSQL function
3. [ ] Port `allM2OandO2ORels` query
4. [ ] Implement junction table detection (`addM2MRels` logic)
5. [ ] Create `get_relationships()` function
6. [ ] Add comprehensive tests

**Deliverable:** `pg_schema_relationships` extension with relationship introspection

### Phase 2: Subscription Filter Generation

**Tasks:**

1. [ ] Design query structure JSON format
2. [ ] Implement `get_subscription_filters()` function
3. [ ] Handle nested relationships (recursive)
4. [ ] Handle M2M junction tables correctly
5. [ ] Add tests for complex query scenarios

**Deliverable:** Function to generate subscription filters from query structure

### Phase 3: PostgREST Integration

**Tasks:**

1. [ ] Add `server-subscription-metadata` config option
2. [ ] Generate query structure from parsed request
3. [ ] Implement Link header generation
4. [ ] Add endpoint for subscription metadata retrieval
5. [ ] Update documentation

**Deliverable:** PostgREST returns subscription metadata via Link header

### Phase 4: supabase-js Integration

**Tasks:**

1. [ ] Implement `useReactive()` hook
2. [ ] Parse Link header from PostgREST responses
3. [ ] Fetch and parse subscription metadata
4. [ ] Create/manage Realtime subscriptions
5. [ ] Handle subscription updates on data changes
6. [ ] Add TypeScript types

**Deliverable:** `useReactive()` hook in supabase-js

### Phase 5: Optimization & Edge Cases

**Tasks:**

1. [ ] Cache relationship metadata in extension
2. [ ] Handle view dependencies
3. [ ] Handle computed relationships
4. [ ] Handle schema changes (cache invalidation)
5. [ ] Performance testing and optimization

---

## Code Extraction Reference

### From PostgREST to PL/pgSQL

#### allM2OandO2ORels (SchemaCache.hs:733-775)

The Haskell code constructs a SQL query. The core query logic:

```sql
-- Extracted and simplified from PostgREST
WITH fk_constraints AS (
  SELECT
    c.conname AS constraint_name,
    ns1.nspname AS from_schema,
    tab.relname AS from_table,
    ns2.nspname AS to_schema,
    other.relname AS to_table,
    ARRAY(
      SELECT attname FROM pg_attribute
      WHERE attrelid = c.conrelid AND attnum = ANY(c.conkey)
      ORDER BY array_position(c.conkey, attnum)
    ) AS from_columns,
    ARRAY(
      SELECT attname FROM pg_attribute
      WHERE attrelid = c.confrelid AND attnum = ANY(c.confkey)
      ORDER BY array_position(c.confkey, attnum)
    ) AS to_columns
  FROM pg_constraint c
  JOIN pg_class tab ON tab.oid = c.conrelid
  JOIN pg_namespace ns1 ON ns1.oid = tab.relnamespace
  JOIN pg_class other ON other.oid = c.confrelid
  JOIN pg_namespace ns2 ON ns2.oid = other.relnamespace
  WHERE c.contype = 'f'
),
pk_columns AS (
  SELECT
    ns.nspname AS schema_name,
    tab.relname AS table_name,
    ARRAY(
      SELECT attname FROM pg_attribute
      WHERE attrelid = c.conrelid AND attnum = ANY(c.conkey)
    ) AS pk_cols
  FROM pg_constraint c
  JOIN pg_class tab ON tab.oid = c.conrelid
  JOIN pg_namespace ns ON ns.oid = tab.relnamespace
  WHERE c.contype = 'p'
)
SELECT
  fk.*,
  CASE
    WHEN fk.from_columns <@ pk.pk_cols THEN 'o2o'
    ELSE 'm2o'
  END AS cardinality
FROM fk_constraints fk
LEFT JOIN pk_columns pk
  ON pk.schema_name = fk.from_schema
  AND pk.table_name = fk.from_table;
```

#### Junction Table Detection (SchemaCache.hs:554-582)

Logic to identify M2M relationships:

```sql
-- A table is a junction table if:
-- 1. It has 2+ foreign keys
-- 2. All FK columns are part of the primary key

WITH table_fks AS (
  SELECT
    from_schema,
    from_table,
    COUNT(*) AS fk_count,
    ARRAY_AGG(to_schema || '.' || to_table) AS related_tables,
    ARRAY_AGG(from_columns) AS all_fk_columns
  FROM fk_constraints
  GROUP BY from_schema, from_table
  HAVING COUNT(*) >= 2
),
junction_candidates AS (
  SELECT
    tf.*,
    pk.pk_cols
  FROM table_fks tf
  JOIN pk_columns pk
    ON pk.schema_name = tf.from_schema
    AND pk.table_name = tf.from_table
  WHERE (
    -- All FK columns are part of the PK
    SELECT bool_and(col = ANY(pk.pk_cols))
    FROM unnest(tf.all_fk_columns) AS cols, unnest(cols) AS col
  )
)
SELECT * FROM junction_candidates;
```

---

## Testing Strategy

### Unit Tests (Extension)

```sql
-- Test M2O detection
CREATE TABLE test_users (id serial PRIMARY KEY);
CREATE TABLE test_posts (
  id serial PRIMARY KEY,
  user_id int REFERENCES test_users(id)
);

SELECT * FROM get_relationships(ARRAY['public'])
WHERE from_table = 'test_posts' AND to_table = 'test_users';
-- Should return M2O relationship

-- Test M2M detection
CREATE TABLE test_tags (id serial PRIMARY KEY);
CREATE TABLE test_post_tags (
  post_id int REFERENCES test_posts(id),
  tag_id int REFERENCES test_tags(id),
  PRIMARY KEY (post_id, tag_id)
);

SELECT * FROM get_relationships(ARRAY['public'])
WHERE junction_table = 'test_post_tags';
-- Should return M2M relationship between posts and tags
```

### Integration Tests (PostgREST)

```bash
# Test Link header presence
curl -i 'http://localhost:3000/users?select=*,posts(*)'
# Should include: Link: <...>; rel="subscription"

# Test subscription metadata endpoint
curl 'http://localhost:3000/rpc/get_subscription_filters?...'
# Should return filter array
```

### E2E Tests (supabase-js)

```typescript
test('useReactive subscribes to correct tables', async () => {
  const { result } = renderHook(() =>
    useReactive(
      supabase.from('users').select('*, organizations(*)').eq('id', '1')
    )
  )

  // Verify subscriptions created
  expect(mockRealtimeSubscriptions).toContainEqual(
    expect.objectContaining({ table: 'users', filter: 'id=eq.1' })
  )
  expect(mockRealtimeSubscriptions).toContainEqual(
    expect.objectContaining({ table: 'members', filter: 'user_id=eq.1' })
  )
})
```

---

## Open Questions

1. **Caching strategy**: Should the extension cache relationship metadata? How to invalidate on DDL changes?

2. **View support**: How deep should view dependency tracing go? PostgREST's implementation is complex.

3. **Computed relationships**: Should we support function-based relationships in v1?

4. **Performance**: For queries with many nested relations, subscription count could explode. Should we have limits?

5. **Security**: Should subscription metadata respect RLS policies?

6. **Versioning**: How to handle extension upgrades with schema changes?

---

## References

- [PostgREST SchemaCache.hs](src/PostgREST/SchemaCache.hs) - Current introspection implementation
- [RFC 8288 - Web Linking](https://tools.ietf.org/html/rfc8288) - Link header specification
- [PostgreSQL System Catalogs](https://www.postgresql.org/docs/current/catalogs.html)
- [Supabase Realtime](https://supabase.com/docs/guides/realtime)
