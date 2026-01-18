# pg_schema_relationships

A PostgreSQL Trusted Language Extension (TLE) for database schema introspection. Discovers table relationships, foreign keys, and generates subscription metadata for reactive queries.

## Features

- **Relationship Discovery** — Automatically detects M2O, O2M, O2O, and M2M relationships
- **Junction Table Detection** — Identifies many-to-many relationships through junction tables
- **Subscription Filters** — Generates Realtime subscription metadata for nested queries
- **TLE Compatible** — Installs without superuser privileges on managed databases
- **Zero Dependencies** — Pure PL/pgSQL, works on PostgreSQL 12+

## Installation

### As a Trusted Language Extension (Supabase, RDS, etc.)

```sql
-- Requires pg_tle to be available
CREATE EXTENSION IF NOT EXISTS pg_tle;

-- Install from the extension SQL
SELECT pgtle.install_extension_version_sql(
  'pg_schema_relationships',
  '0.1.0',
  $extcode$
    -- Extension code here (see sql/pg_schema_relationships--0.1.0.sql)
  $extcode$
);

-- Create the extension
CREATE EXTENSION pg_schema_relationships;
```

### As a Standard Extension

```sql
CREATE EXTENSION pg_schema_relationships;
```

## Quick Start

```sql
-- Get all relationships in the public schema
SELECT * FROM pgsrx.get_relationships();

-- Get relationships for specific schemas
SELECT * FROM pgsrx.get_relationships(ARRAY['public', 'app']);

-- Get subscription filters for a nested query
SELECT * FROM pgsrx.get_subscription_filters('{
  "schema": "public",
  "table": "users",
  "filters": [{"column": "id", "op": "eq", "value": "10"}],
  "nested": [{"relation": "organizations"}]
}'::jsonb);
```

## API Reference

### Types

#### `pgsrx.cardinality`

Relationship cardinality enum:

| Value | Description |
|-------|-------------|
| `m2o` | Many-to-One (foreign key source → target) |
| `o2m` | One-to-Many (inverse of M2O) |
| `o2o` | One-to-One (FK columns are unique/PK) |
| `m2m` | Many-to-Many (via junction table) |

#### `pgsrx.relationship`

```sql
(
  from_schema       name,      -- Source table schema
  from_table        name,      -- Source table name
  from_columns      name[],    -- Source columns
  to_schema         name,      -- Target table schema
  to_table          name,      -- Target table name
  to_columns        name[],    -- Target columns
  cardinality       pgsrx.cardinality,
  constraint_name   name,      -- FK constraint name
  is_self_relation  boolean,   -- Self-referential?
  junction_schema   name,      -- Junction table schema (M2M only)
  junction_table    name,      -- Junction table name (M2M only)
  junction_from_fk  name,      -- FK to source table (M2M only)
  junction_to_fk    name       -- FK to target table (M2M only)
)
```

#### `pgsrx.subscription_filter`

```sql
(
  schema_name   name,    -- Table schema
  table_name    name,    -- Table name
  column_name   name,    -- Column to filter
  operator      text,    -- Filter operator (eq, in, etc.)
  value         text     -- Filter value
)
```

### Functions

#### `pgsrx.get_tables(schemas name[])`

Returns all tables, views, and materialized views with their columns and metadata.

```sql
SELECT * FROM pgsrx.get_tables(ARRAY['public']);
```

Returns: `SETOF pgsrx.table_info`

#### `pgsrx.get_relationships(schemas name[])`

Returns all relationships (M2O, O2M, O2O, M2M) for the specified schemas.

```sql
SELECT * FROM pgsrx.get_relationships(ARRAY['public']);
```

Returns: `SETOF pgsrx.relationship`

#### `pgsrx.get_fk_relationships(schemas name[])`

Returns only foreign key relationships (M2O and O2O).

```sql
SELECT * FROM pgsrx.get_fk_relationships(ARRAY['public']);
```

Returns: `SETOF pgsrx.relationship`

#### `pgsrx.get_m2m_relationships(schemas name[])`

Returns only many-to-many relationships detected via junction tables.

```sql
SELECT * FROM pgsrx.get_m2m_relationships(ARRAY['public']);
```

Returns: `SETOF pgsrx.relationship`

#### `pgsrx.get_subscription_filters(query_info jsonb)`

Generates Realtime subscription filters for a nested query structure.

```sql
SELECT * FROM pgsrx.get_subscription_filters('{
  "schema": "public",
  "table": "users",
  "filters": [{"column": "id", "op": "eq", "value": "10"}],
  "nested": [
    {
      "relation": "organizations",
      "nested": [
        {"relation": "projects"}
      ]
    }
  ]
}'::jsonb);
```

Returns: `SETOF pgsrx.subscription_filter`

## Examples

### Example Schema

```sql
CREATE TABLE users (
  id serial PRIMARY KEY,
  name text NOT NULL
);

CREATE TABLE organizations (
  id serial PRIMARY KEY,
  name text NOT NULL
);

-- Junction table for M2M
CREATE TABLE members (
  user_id int REFERENCES users(id),
  org_id int REFERENCES organizations(id),
  PRIMARY KEY (user_id, org_id)
);

CREATE TABLE projects (
  id serial PRIMARY KEY,
  name text NOT NULL,
  org_id int REFERENCES organizations(id)
);
```

### Discover Relationships

```sql
SELECT
  from_table,
  to_table,
  cardinality,
  junction_table
FROM pgsrx.get_relationships();
```

Result:

| from_table | to_table | cardinality | junction_table |
|------------|----------|-------------|----------------|
| members | users | m2o | |
| members | organizations | m2o | |
| users | members | o2m | |
| organizations | members | o2m | |
| projects | organizations | m2o | |
| organizations | projects | o2m | |
| users | organizations | m2m | members |
| organizations | users | m2m | members |

### Generate Subscription Filters

For a query like:

```javascript
supabase
  .from('users')
  .select('id, name, organizations(id, name)')
  .eq('id', '10')
```

Generate the subscription filters:

```sql
SELECT * FROM pgsrx.get_subscription_filters('{
  "schema": "public",
  "table": "users",
  "filters": [{"column": "id", "op": "eq", "value": "10"}],
  "nested": [{"relation": "organizations"}]
}'::jsonb);
```

Result:

| schema_name | table_name | column_name | operator | value |
|-------------|------------|-------------|----------|-------|
| public | users | id | eq | 10 |
| public | members | user_id | eq | 10 |

The `members` junction table subscription ensures you're notified when the user is added/removed from an organization.

### Find Junction Tables

```sql
SELECT DISTINCT
  junction_schema,
  junction_table,
  from_table AS table_a,
  to_table AS table_b
FROM pgsrx.get_relationships()
WHERE cardinality = 'm2m'
  AND from_table < to_table;  -- Deduplicate
```

## Use Cases

### Supabase Realtime Reactive Queries

```typescript
import { useReactive } from '@supabase/supabase-js'

function UserOrgs({ userId }) {
  // Automatically subscribes to users, members (junction), and organizations
  const { data } = useReactive(
    supabase
      .from('users')
      .select('*, organizations(*)')
      .eq('id', userId)
  )

  return <div>{/* render data */}</div>
}
```

### Schema Documentation

```sql
-- Generate relationship documentation
SELECT
  format('%s.%s', from_schema, from_table) AS source,
  CASE cardinality
    WHEN 'm2o' THEN '→'
    WHEN 'o2m' THEN '←'
    WHEN 'o2o' THEN '↔'
    WHEN 'm2m' THEN '⇆'
  END AS relation,
  format('%s.%s', to_schema, to_table) AS target,
  CASE
    WHEN junction_table IS NOT NULL
    THEN format('(via %s)', junction_table)
    ELSE ''
  END AS via
FROM pgsrx.get_relationships()
ORDER BY from_table, to_table;
```

### GraphQL Schema Generation

```sql
-- Get type definitions for GraphQL
SELECT
  t.table_name,
  jsonb_agg(jsonb_build_object(
    'field', r.to_table,
    'type', CASE r.cardinality
      WHEN 'o2m' THEN format('[%s!]!', r.to_table)
      WHEN 'm2m' THEN format('[%s!]!', r.to_table)
      ELSE format('%s', r.to_table)
    END
  )) AS relationships
FROM pgsrx.get_tables() t
LEFT JOIN pgsrx.get_relationships() r
  ON r.from_schema = t.table_schema
  AND r.from_table = t.table_name
GROUP BY t.table_name;
```

## Performance

- All functions are marked `STABLE` and `PARALLEL SAFE`
- Queries system catalogs directly (no temp tables)
- Optional caching available via `pgsrx.get_relationships_cached()`

### Caching (Optional)

```sql
-- Use cached relationships (5-minute TTL)
SELECT * FROM pgsrx.get_relationships_cached(
  ARRAY['public'],
  interval '5 minutes'
);
```

Cache is automatically invalidated on DDL changes (CREATE/ALTER/DROP TABLE).

## Compatibility

| PostgreSQL Version | Status |
|-------------------|--------|
| 17 | ✅ Supported |
| 16 | ✅ Supported |
| 15 | ✅ Supported |
| 14 | ✅ Supported |
| 13 | ✅ Supported |
| 12 | ✅ Supported |
| 11 | ⚠️ Limited (no procedures) |

## License

Apache 2.0

## Related

- [PostgREST](https://postgrest.org) — REST API for PostgreSQL
- [Supabase Realtime](https://supabase.com/docs/guides/realtime) — Real-time subscriptions
- [pg_tle](https://github.com/aws/pg_tle) — Trusted Language Extensions
