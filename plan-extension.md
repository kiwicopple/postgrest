# pg_schema_relationships: Trusted Language Extension Plan

## Overview

This document details the implementation plan for `pg_schema_relationships`, a PostgreSQL Trusted Language Extension (TLE) that extracts PostgREST's database introspection logic into reusable PL/pgSQL functions.

### What is a Trusted Language Extension?

TLEs allow creating PostgreSQL extensions using trusted procedural languages (PL/pgSQL, PL/v8, etc.) without requiring superuser privileges or file system access. This is critical for:

- **Managed database services** (Supabase, RDS, Cloud SQL) where users can't install native extensions
- **Security** — No C code execution, runs in PostgreSQL's sandbox
- **Portability** — Pure SQL/PL/pgSQL works across all PostgreSQL versions 12+

### Why TLE for This Use Case?

1. **Supabase deployment** — Can be installed on any Supabase project without platform changes
2. **User-installable** — Database owners can install without superuser
3. **Shared between services** — PostgREST, Realtime, and other services query the same functions
4. **Version independence** — Updates don't require PostgreSQL restart

---

## Extension Structure

```
pg_schema_relationships/
├── pg_schema_relationships.control
├── Makefile
├── sql/
│   ├── pg_schema_relationships--0.1.0.sql    # Main extension file
│   └── migrations/
│       └── pg_schema_relationships--0.1.0--0.2.0.sql
├── src/
│   ├── types.sql                              # Custom types
│   ├── util.sql                               # Helper functions
│   ├── tables.sql                             # Table introspection
│   ├── relationships.sql                      # FK/M2M relationships
│   ├── views.sql                              # View dependencies
│   └── subscriptions.sql                      # Subscription filter generation
└── test/
    ├── fixtures.sql                           # Test schema setup
    └── test_relationships.sql                 # pgTAP tests
```

### Control File

```ini
# pg_schema_relationships.control
comment = 'Database schema introspection for relationships and subscription metadata'
default_version = '0.1.0'
relocatable = true
schema = pgsrx
trusted = true
```

---

## Type Definitions

```sql
-- src/types.sql

-- Relationship cardinality types
CREATE TYPE pgsrx.cardinality AS ENUM (
  'm2o',    -- Many-to-One (FK source → FK target)
  'o2m',    -- One-to-Many (inverse of M2O)
  'o2o',    -- One-to-One (FK columns are unique/PK)
  'm2m'     -- Many-to-Many (via junction table)
);

-- Represents a relationship between two tables
CREATE TYPE pgsrx.relationship AS (
  -- Source table
  from_schema       name,
  from_table        name,
  from_columns      name[],

  -- Target table
  to_schema         name,
  to_table          name,
  to_columns        name[],

  -- Relationship metadata
  cardinality       pgsrx.cardinality,
  constraint_name   name,
  is_self_relation  boolean,

  -- Junction table info (only for M2M)
  junction_schema   name,
  junction_table    name,
  junction_from_fk  name,      -- FK constraint from junction → from_table
  junction_to_fk    name       -- FK constraint from junction → to_table
);

-- Represents a column in a table
CREATE TYPE pgsrx.column_info AS (
  column_name               name,
  description               text,
  is_nullable               boolean,
  data_type                 text,
  nominal_data_type         text,
  character_maximum_length  integer,
  column_default            text,
  enum_values               text[]
);

-- Represents a table/view
CREATE TYPE pgsrx.table_info AS (
  table_schema      name,
  table_name        name,
  description       text,
  is_view           boolean,
  is_insertable     boolean,
  is_updatable      boolean,
  is_deletable      boolean,
  primary_key_cols  name[],
  columns           pgsrx.column_info[]
);

-- Subscription filter for Realtime
CREATE TYPE pgsrx.subscription_filter AS (
  schema_name   name,
  table_name    name,
  column_name   name,
  operator      text,
  value         text
);

-- Query node for parsing nested selects
CREATE TYPE pgsrx.query_node AS (
  relation_name   text,
  alias           text,
  filters         jsonb,      -- [{"column": "id", "op": "eq", "value": "1"}]
  selected_cols   text[],
  nested          jsonb       -- Array of nested query_nodes
);
```

---

## Core Functions

### 1. Table Introspection

Extracted from `allTables` (SchemaCache.hs:585-730):

```sql
-- src/tables.sql

CREATE OR REPLACE FUNCTION pgsrx.get_tables(
  schemas name[] DEFAULT ARRAY['public']::name[]
)
RETURNS SETOF pgsrx.table_info
LANGUAGE sql
STABLE
PARALLEL SAFE
AS $$
WITH
-- Base types CTE for resolving domain types
base_types AS (
  SELECT
    t.oid,
    t.typnamespace AS base_namespace,
    CASE
      WHEN t.typtype = 'd' THEN (
        WITH RECURSIVE recurse AS (
          SELECT t.typbasetype, 0 AS depth
          UNION ALL
          SELECT bt.typbasetype, recurse.depth + 1
          FROM recurse
          JOIN pg_type bt ON bt.oid = recurse.typbasetype
          WHERE bt.typtype = 'd'
        )
        SELECT typbasetype FROM recurse ORDER BY depth DESC LIMIT 1
      )
      ELSE t.oid
    END AS base_type
  FROM pg_type t
),
-- Column information for each table
columns AS (
  SELECT
    c.oid AS relid,
    a.attname::name AS column_name,
    d.description,
    NOT (a.attnotnull OR t.typtype = 'd' AND t.typnotnull) AS is_nullable,
    CASE
      WHEN t.typtype = 'd' THEN
        CASE
          WHEN bt.base_namespace = 'pg_catalog'::regnamespace
            THEN format_type(bt.base_type, NULL::integer)
          ELSE format_type(a.atttypid, a.atttypmod)
        END
      ELSE
        CASE
          WHEN t.typnamespace = 'pg_catalog'::regnamespace
            THEN format_type(a.atttypid, NULL::integer)
          ELSE format_type(a.atttypid, a.atttypmod)
        END
    END::text AS data_type,
    format_type(a.atttypid, a.atttypmod)::text AS nominal_data_type,
    information_schema._pg_char_max_length(
      information_schema._pg_truetypid(a.*, t.*),
      information_schema._pg_truetypmod(a.*, t.*)
    )::integer AS character_maximum_length,
    CASE
      WHEN (t.typbasetype != 0) AND (ad.adbin IS NULL)
        THEN pg_get_expr(t.typdefaultbin, 0)
      WHEN a.attidentity = 'd'
        THEN format('nextval(%L)', seq.objid::regclass)
      WHEN a.attgenerated = 's'
        THEN NULL
      ELSE pg_get_expr(ad.adbin, ad.adrelid)::text
    END AS column_default,
    bt.base_type,
    a.attnum::integer AS position
  FROM pg_attribute a
  LEFT JOIN pg_description d
    ON d.objoid = a.attrelid
    AND d.objsubid = a.attnum
    AND d.classoid = 'pg_class'::regclass
  LEFT JOIN pg_attrdef ad
    ON a.attrelid = ad.adrelid AND a.attnum = ad.adnum
  JOIN pg_class c ON a.attrelid = c.oid
  JOIN pg_type t ON a.atttypid = t.oid
  LEFT JOIN base_types bt ON t.oid = bt.oid
  LEFT JOIN pg_depend seq
    ON seq.refobjid = a.attrelid
    AND seq.refobjsubid = a.attnum
    AND seq.deptype = 'i'
  WHERE
    NOT pg_is_other_temp_schema(c.relnamespace)
    AND a.attnum > 0
    AND NOT a.attisdropped
    AND c.relkind IN ('r', 'v', 'f', 'm', 'p')
    AND c.relnamespace = ANY(schemas::regnamespace[])
),
-- Aggregate columns into arrays
columns_agg AS (
  SELECT
    relid,
    array_agg(
      ROW(
        column_name,
        description,
        is_nullable,
        data_type,
        nominal_data_type,
        character_maximum_length,
        column_default,
        COALESCE(
          (SELECT array_agg(enumlabel ORDER BY enumsortorder)
           FROM pg_enum WHERE enumtypid = base_type),
          '{}'::text[]
        )
      )::pgsrx.column_info
      ORDER BY position
    ) AS columns
  FROM columns
  GROUP BY relid
),
-- Primary key columns
tbl_pk_cols AS (
  SELECT
    r.oid AS relid,
    array_agg(a.attname ORDER BY a.attname) AS pk_cols
  FROM pg_class r
  JOIN pg_constraint c ON r.oid = c.conrelid
  JOIN pg_attribute a ON a.attrelid = r.oid AND a.attnum = ANY(c.conkey)
  WHERE
    c.contype = 'p'
    AND r.relkind IN ('r', 'p')
    AND r.relnamespace NOT IN ('pg_catalog'::regnamespace, 'information_schema'::regnamespace)
    AND NOT pg_is_other_temp_schema(r.relnamespace)
    AND NOT a.attisdropped
  GROUP BY r.oid
)
SELECT
  n.nspname::name AS table_schema,
  c.relname::name AS table_name,
  d.description::text,
  c.relkind IN ('v', 'm') AS is_view,
  (
    c.relkind IN ('r', 'p')
    OR (
      c.relkind IN ('v', 'f')
      AND (pg_relation_is_updatable(c.oid::regclass, TRUE) & 8) = 8
    )
  ) AS is_insertable,
  (
    c.relkind IN ('r', 'p')
    OR (
      c.relkind IN ('v', 'f')
      AND (pg_relation_is_updatable(c.oid::regclass, TRUE) & 4) = 4
    )
  ) AS is_updatable,
  (
    c.relkind IN ('r', 'p')
    OR (
      c.relkind IN ('v', 'f')
      AND (pg_relation_is_updatable(c.oid::regclass, TRUE) & 16) = 16
    )
  ) AS is_deletable,
  COALESCE(tpks.pk_cols, '{}'::name[]) AS primary_key_cols,
  COALESCE(cols_agg.columns, '{}'::pgsrx.column_info[]) AS columns
FROM pg_class c
JOIN pg_namespace n ON n.oid = c.relnamespace
LEFT JOIN pg_description d
  ON d.objoid = c.oid
  AND d.objsubid = 0
  AND d.classoid = 'pg_class'::regclass
LEFT JOIN tbl_pk_cols tpks ON c.oid = tpks.relid
LEFT JOIN columns_agg cols_agg ON c.oid = cols_agg.relid
WHERE
  c.relkind IN ('v', 'r', 'm', 'f', 'p')
  AND c.relnamespace = ANY(schemas::regnamespace[])
  AND c.relnamespace NOT IN ('pg_catalog'::regnamespace, 'information_schema'::regnamespace)
  AND NOT c.relispartition
ORDER BY table_schema, table_name
$$;

COMMENT ON FUNCTION pgsrx.get_tables IS
  'Returns all tables, views, materialized views, and foreign tables with their columns and metadata';
```

### 2. Foreign Key Relationships (M2O, O2O)

Extracted from `allM2OandO2ORels` (SchemaCache.hs:733-775):

```sql
-- src/relationships.sql

CREATE OR REPLACE FUNCTION pgsrx.get_fk_relationships(
  schemas name[] DEFAULT ARRAY['public']::name[]
)
RETURNS SETOF pgsrx.relationship
LANGUAGE sql
STABLE
PARALLEL SAFE
AS $$
WITH
-- Get PK and unique constraint columns for O2O detection
pks_uniques_cols AS (
  SELECT
    conrelid,
    array_agg(key ORDER BY key) AS cols
  FROM pg_constraint,
  LATERAL unnest(conkey) AS _(key)
  WHERE
    contype IN ('p', 'u')
    AND connamespace NOT IN ('pg_catalog'::regnamespace)
  GROUP BY oid, conrelid
),
-- Get all foreign key constraints
fk_rels AS (
  SELECT
    ns1.nspname AS from_schema,
    tab.relname AS from_table,
    ns2.nspname AS to_schema,
    other.relname AS to_table,
    traint.conrelid = traint.confrelid AS is_self,
    traint.conname AS constraint_name,
    column_info.from_cols,
    column_info.to_cols,
    -- Determine if O2O: FK columns are a subset of PK/unique columns
    (column_info.col_nums IN (
      SELECT cols FROM pks_uniques_cols WHERE conrelid = traint.conrelid
    )) AS is_one_to_one
  FROM pg_constraint traint
  JOIN LATERAL (
    SELECT
      array_agg(cols.attname ORDER BY ord) AS from_cols,
      array_agg(refs.attname ORDER BY ord) AS to_cols,
      array_agg(cols.attnum ORDER BY cols.attnum) AS col_nums
    FROM unnest(traint.conkey, traint.confkey) WITH ORDINALITY AS _(col, ref, ord)
    JOIN pg_attribute cols ON cols.attrelid = traint.conrelid AND cols.attnum = col
    JOIN pg_attribute refs ON refs.attrelid = traint.confrelid AND refs.attnum = ref
  ) AS column_info ON TRUE
  JOIN pg_namespace ns1 ON ns1.oid = traint.connamespace
  JOIN pg_class tab ON tab.oid = traint.conrelid
  JOIN pg_class other ON other.oid = traint.confrelid
  JOIN pg_namespace ns2 ON ns2.oid = other.relnamespace
  WHERE
    traint.contype = 'f'
    AND traint.conparentid = 0
    AND (ns1.nspname = ANY(schemas) OR ns2.nspname = ANY(schemas))
)
SELECT
  from_schema::name,
  from_table::name,
  from_cols::name[],
  to_schema::name,
  to_table::name,
  to_cols::name[],
  CASE WHEN is_one_to_one THEN 'o2o' ELSE 'm2o' END::pgsrx.cardinality,
  constraint_name::name,
  is_self,
  NULL::name,   -- junction_schema
  NULL::name,   -- junction_table
  NULL::name,   -- junction_from_fk
  NULL::name    -- junction_to_fk
FROM fk_rels
ORDER BY from_schema, from_table, constraint_name
$$;

COMMENT ON FUNCTION pgsrx.get_fk_relationships IS
  'Returns all foreign key relationships with cardinality (M2O or O2O)';
```

### 3. Many-to-Many Relationships

Extracted from `addM2MRels` (SchemaCache.hs:554-566):

```sql
-- src/relationships.sql (continued)

CREATE OR REPLACE FUNCTION pgsrx.get_m2m_relationships(
  schemas name[] DEFAULT ARRAY['public']::name[]
)
RETURNS SETOF pgsrx.relationship
LANGUAGE sql
STABLE
PARALLEL SAFE
AS $$
WITH
-- Get all M2O relationships first
m2o_rels AS (
  SELECT * FROM pgsrx.get_fk_relationships(schemas)
  WHERE cardinality = 'm2o'
),
-- Get primary key columns for each table
pk_cols AS (
  SELECT
    n.nspname AS schema_name,
    c.relname AS table_name,
    array_agg(a.attname ORDER BY a.attnum) AS cols
  FROM pg_constraint con
  JOIN pg_class c ON c.oid = con.conrelid
  JOIN pg_namespace n ON n.oid = c.relnamespace
  JOIN pg_attribute a ON a.attrelid = c.oid AND a.attnum = ANY(con.conkey)
  WHERE con.contype = 'p'
  GROUP BY n.nspname, c.relname
),
-- Find junction tables: tables with 2+ FKs where all FK columns are part of the PK
junction_candidates AS (
  SELECT
    r1.from_schema AS junction_schema,
    r1.from_table AS junction_table,
    r1.constraint_name AS fk1_name,
    r1.to_schema AS table1_schema,
    r1.to_table AS table1_name,
    r1.from_columns AS fk1_from_cols,
    r1.to_columns AS fk1_to_cols,
    r2.constraint_name AS fk2_name,
    r2.to_schema AS table2_schema,
    r2.to_table AS table2_name,
    r2.from_columns AS fk2_from_cols,
    r2.to_columns AS fk2_to_cols,
    pk.cols AS pk_cols
  FROM m2o_rels r1
  JOIN m2o_rels r2
    ON r1.from_schema = r2.from_schema
    AND r1.from_table = r2.from_table
    AND r1.constraint_name < r2.constraint_name  -- Avoid duplicates
  JOIN pk_cols pk
    ON pk.schema_name = r1.from_schema
    AND pk.table_name = r1.from_table
  WHERE
    -- All FK columns from both FKs must be part of the PK
    r1.from_columns <@ pk.cols
    AND r2.from_columns <@ pk.cols
)
-- Generate M2M relationships in both directions
SELECT
  table1_schema::name AS from_schema,
  table1_name::name AS from_table,
  fk1_to_cols::name[] AS from_columns,
  table2_schema::name AS to_schema,
  table2_name::name AS to_table,
  fk2_to_cols::name[] AS to_columns,
  'm2m'::pgsrx.cardinality,
  (fk1_name || '_' || fk2_name)::name AS constraint_name,
  (table1_schema = table2_schema AND table1_name = table2_name) AS is_self_relation,
  junction_schema::name,
  junction_table::name,
  fk1_name::name AS junction_from_fk,
  fk2_name::name AS junction_to_fk
FROM junction_candidates
$$;

COMMENT ON FUNCTION pgsrx.get_m2m_relationships IS
  'Detects many-to-many relationships via junction tables';
```

### 4. Inverse Relationships (O2M)

Extracted from `addInverseRels` (SchemaCache.hs:547-551):

```sql
-- src/relationships.sql (continued)

CREATE OR REPLACE FUNCTION pgsrx.get_inverse_relationships(
  schemas name[] DEFAULT ARRAY['public']::name[]
)
RETURNS SETOF pgsrx.relationship
LANGUAGE sql
STABLE
PARALLEL SAFE
AS $$
-- Generate O2M (inverse of M2O)
SELECT
  to_schema AS from_schema,
  to_table AS from_table,
  to_columns AS from_columns,
  from_schema AS to_schema,
  from_table AS to_table,
  from_columns AS to_columns,
  'o2m'::pgsrx.cardinality,
  constraint_name,
  is_self_relation,
  NULL::name,
  NULL::name,
  NULL::name,
  NULL::name
FROM pgsrx.get_fk_relationships(schemas)
WHERE cardinality = 'm2o'
$$;

COMMENT ON FUNCTION pgsrx.get_inverse_relationships IS
  'Returns inverse (O2M) relationships derived from M2O foreign keys';
```

### 5. All Relationships (Combined)

```sql
-- src/relationships.sql (continued)

CREATE OR REPLACE FUNCTION pgsrx.get_relationships(
  schemas name[] DEFAULT ARRAY['public']::name[]
)
RETURNS SETOF pgsrx.relationship
LANGUAGE sql
STABLE
PARALLEL SAFE
AS $$
  -- M2O and O2O from foreign keys
  SELECT * FROM pgsrx.get_fk_relationships(schemas)
  UNION ALL
  -- O2M (inverse of M2O)
  SELECT * FROM pgsrx.get_inverse_relationships(schemas)
  UNION ALL
  -- M2M via junction tables
  SELECT * FROM pgsrx.get_m2m_relationships(schemas)
$$;

COMMENT ON FUNCTION pgsrx.get_relationships IS
  'Returns all relationships: M2O, O2M, O2O, and M2M';
```

### 6. Subscription Filter Generation

```sql
-- src/subscriptions.sql

CREATE OR REPLACE FUNCTION pgsrx.get_subscription_filters(
  query_info jsonb
)
RETURNS SETOF pgsrx.subscription_filter
LANGUAGE plpgsql
STABLE
AS $$
DECLARE
  root_schema name;
  root_table name;
  root_filters jsonb;
  nested_relations jsonb;
  rel record;
  nested jsonb;
  filter jsonb;
  pk_cols name[];
  fk_rel record;
BEGIN
  -- Parse root query info
  root_schema := COALESCE(query_info->>'schema', 'public')::name;
  root_table := (query_info->>'table')::name;
  root_filters := COALESCE(query_info->'filters', '[]'::jsonb);
  nested_relations := COALESCE(query_info->'nested', '[]'::jsonb);

  -- 1. Return filters for the root table
  FOR filter IN SELECT * FROM jsonb_array_elements(root_filters)
  LOOP
    RETURN NEXT ROW(
      root_schema,
      root_table,
      (filter->>'column')::name,
      COALESCE(filter->>'op', 'eq'),
      filter->>'value'
    )::pgsrx.subscription_filter;
  END LOOP;

  -- 2. For each nested relation, find the relationship and generate filters
  FOR nested IN SELECT * FROM jsonb_array_elements(nested_relations)
  LOOP
    DECLARE
      nested_relation text := nested->>'relation';
      nested_schema name;
      nested_table name;
    BEGIN
      -- Find the relationship from root table to nested relation
      SELECT r.* INTO fk_rel
      FROM pgsrx.get_relationships(ARRAY[root_schema]) r
      WHERE r.from_schema = root_schema
        AND r.from_table = root_table
        AND r.to_table = nested_relation
      LIMIT 1;

      IF fk_rel IS NULL THEN
        -- Try reverse direction
        SELECT r.* INTO fk_rel
        FROM pgsrx.get_relationships(ARRAY[root_schema]) r
        WHERE r.to_schema = root_schema
          AND r.to_table = root_table
          AND r.from_table = nested_relation
        LIMIT 1;
      END IF;

      IF fk_rel IS NOT NULL THEN
        nested_schema := COALESCE(fk_rel.to_schema, fk_rel.from_schema);
        nested_table := nested_relation::name;

        -- Handle M2M: subscribe to junction table
        IF fk_rel.cardinality = 'm2m' THEN
          -- Subscribe to junction table changes
          FOR filter IN SELECT * FROM jsonb_array_elements(root_filters)
          LOOP
            -- Junction table subscription using the FK column that points to root
            RETURN NEXT ROW(
              fk_rel.junction_schema,
              fk_rel.junction_table,
              fk_rel.from_columns[1],  -- FK column pointing to root table
              COALESCE(filter->>'op', 'eq'),
              filter->>'value'
            )::pgsrx.subscription_filter;
          END LOOP;
        ELSE
          -- For M2O/O2M/O2O, subscribe to FK column changes
          FOR filter IN SELECT * FROM jsonb_array_elements(root_filters)
          LOOP
            RETURN NEXT ROW(
              nested_schema,
              nested_table,
              fk_rel.to_columns[1],
              COALESCE(filter->>'op', 'eq'),
              filter->>'value'
            )::pgsrx.subscription_filter;
          END LOOP;
        END IF;

        -- Recursively process nested relations
        IF nested->'nested' IS NOT NULL AND jsonb_array_length(nested->'nested') > 0 THEN
          RETURN QUERY SELECT * FROM pgsrx.get_subscription_filters(
            jsonb_build_object(
              'schema', nested_schema,
              'table', nested_table,
              'filters', nested->'filters',
              'nested', nested->'nested'
            )
          );
        END IF;
      END IF;
    END;
  END LOOP;
END;
$$;

COMMENT ON FUNCTION pgsrx.get_subscription_filters IS
  'Generates Realtime subscription filters for a nested query structure';
```

### 7. Helper: Parse PostgREST Select String

```sql
-- src/util.sql

CREATE OR REPLACE FUNCTION pgsrx.parse_select_string(
  table_name text,
  select_string text,
  schema_name text DEFAULT 'public'
)
RETURNS jsonb
LANGUAGE plpgsql
IMMUTABLE
AS $$
DECLARE
  result jsonb;
  parts text[];
  part text;
  nested_match text[];
  columns text[] := '{}';
  nested jsonb := '[]'::jsonb;
BEGIN
  -- Simple parser for PostgREST select syntax
  -- e.g., "id,name,organizations(id,name,projects(id,name))"

  -- This is a simplified implementation
  -- A full implementation would handle:
  -- - Nested parentheses properly
  -- - Aliases (field:alias)
  -- - Spread operator (...)
  -- - JSON operators (->>, ->)

  result := jsonb_build_object(
    'schema', schema_name,
    'table', table_name,
    'filters', '[]'::jsonb,
    'nested', '[]'::jsonb
  );

  -- For MVP, return basic structure
  -- Full parsing should be done client-side
  RETURN result;
END;
$$;

COMMENT ON FUNCTION pgsrx.parse_select_string IS
  'Parses a PostgREST select string into query structure (simplified)';
```

---

## Caching Layer

For performance, add optional caching of relationship metadata:

```sql
-- src/cache.sql

-- Cache table for relationships
CREATE TABLE IF NOT EXISTS pgsrx.relationship_cache (
  id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  schemas name[] NOT NULL,
  relationships jsonb NOT NULL,
  cached_at timestamptz NOT NULL DEFAULT now(),
  UNIQUE(schemas)
);

-- Cache invalidation function
CREATE OR REPLACE FUNCTION pgsrx.invalidate_cache()
RETURNS event_trigger
LANGUAGE plpgsql
AS $$
BEGIN
  TRUNCATE pgsrx.relationship_cache;
END;
$$;

-- Event trigger for DDL changes
CREATE EVENT TRIGGER pgsrx_cache_invalidation
  ON ddl_command_end
  WHEN TAG IN (
    'CREATE TABLE', 'ALTER TABLE', 'DROP TABLE',
    'CREATE VIEW', 'ALTER VIEW', 'DROP VIEW',
    'CREATE FOREIGN TABLE', 'ALTER FOREIGN TABLE', 'DROP FOREIGN TABLE'
  )
  EXECUTE FUNCTION pgsrx.invalidate_cache();

-- Cached relationship lookup
CREATE OR REPLACE FUNCTION pgsrx.get_relationships_cached(
  schemas name[] DEFAULT ARRAY['public']::name[],
  max_age interval DEFAULT '5 minutes'
)
RETURNS SETOF pgsrx.relationship
LANGUAGE plpgsql
STABLE
AS $$
DECLARE
  cached_data jsonb;
BEGIN
  -- Try to get from cache
  SELECT relationships INTO cached_data
  FROM pgsrx.relationship_cache
  WHERE relationship_cache.schemas = get_relationships_cached.schemas
    AND cached_at > now() - max_age;

  IF cached_data IS NOT NULL THEN
    RETURN QUERY SELECT * FROM jsonb_populate_recordset(
      NULL::pgsrx.relationship,
      cached_data
    );
    RETURN;
  END IF;

  -- Cache miss: compute and store
  cached_data := (
    SELECT jsonb_agg(row_to_json(r))
    FROM pgsrx.get_relationships(schemas) r
  );

  INSERT INTO pgsrx.relationship_cache (schemas, relationships)
  VALUES (schemas, cached_data)
  ON CONFLICT (schemas) DO UPDATE SET
    relationships = EXCLUDED.relationships,
    cached_at = now();

  RETURN QUERY SELECT * FROM pgsrx.get_relationships(schemas);
END;
$$;
```

---

## Installation

### As a TLE (Trusted Language Extension)

```sql
-- Install pg_tle first (if not already available)
CREATE EXTENSION IF NOT EXISTS pg_tle;

-- Install the extension
SELECT pgtle.install_extension(
  'pg_schema_relationships',
  '0.1.0',
  'Database schema introspection for relationships',
  $_pg_tle_$
    -- Paste contents of pg_schema_relationships--0.1.0.sql here
  $_pg_tle_$
);

-- Create the extension in your database
CREATE EXTENSION pg_schema_relationships;
```

### As a Standard Extension

```sql
-- If installed on the server
CREATE EXTENSION pg_schema_relationships;

-- With specific schema
CREATE EXTENSION pg_schema_relationships WITH SCHEMA pgsrx;
```

---

## Usage Examples

### Get All Relationships

```sql
-- All relationships in public schema
SELECT * FROM pgsrx.get_relationships();

-- Multiple schemas
SELECT * FROM pgsrx.get_relationships(ARRAY['public', 'app']);

-- Filter by cardinality
SELECT * FROM pgsrx.get_relationships()
WHERE cardinality = 'm2m';
```

### Get Subscription Filters

```sql
-- For a nested query
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

-- Result:
--  schema_name | table_name    | column_name     | operator | value
-- -------------+---------------+-----------------+----------+-------
--  public      | users         | id              | eq       | 10
--  public      | members       | user_id         | eq       | 10
--  public      | organizations | id              | eq       | ...
--  public      | projects      | organization_id | eq       | ...
```

### Get Junction Tables

```sql
-- Find all M2M junction tables
SELECT DISTINCT
  junction_schema,
  junction_table,
  from_table AS connects_table_1,
  to_table AS connects_table_2
FROM pgsrx.get_relationships()
WHERE cardinality = 'm2m';
```

---

## Testing

### Test Fixtures

```sql
-- test/fixtures.sql

CREATE SCHEMA IF NOT EXISTS test_pgsrx;
SET search_path TO test_pgsrx;

-- Basic tables
CREATE TABLE users (
  id serial PRIMARY KEY,
  name text NOT NULL
);

CREATE TABLE organizations (
  id serial PRIMARY KEY,
  name text NOT NULL
);

CREATE TABLE projects (
  id serial PRIMARY KEY,
  name text NOT NULL,
  organization_id int REFERENCES organizations(id)
);

-- Junction table for M2M
CREATE TABLE members (
  user_id int REFERENCES users(id),
  organization_id int REFERENCES organizations(id),
  role text DEFAULT 'member',
  PRIMARY KEY (user_id, organization_id)
);

-- Self-referential
CREATE TABLE categories (
  id serial PRIMARY KEY,
  name text NOT NULL,
  parent_id int REFERENCES categories(id)
);

-- One-to-one
CREATE TABLE user_profiles (
  user_id int PRIMARY KEY REFERENCES users(id),
  bio text,
  avatar_url text
);
```

### pgTAP Tests

```sql
-- test/test_relationships.sql

BEGIN;
SELECT plan(10);

-- Test M2O detection
SELECT ok(
  EXISTS(
    SELECT 1 FROM pgsrx.get_fk_relationships(ARRAY['test_pgsrx'])
    WHERE from_table = 'projects'
      AND to_table = 'organizations'
      AND cardinality = 'm2o'
  ),
  'Detects M2O relationship: projects → organizations'
);

-- Test O2O detection
SELECT ok(
  EXISTS(
    SELECT 1 FROM pgsrx.get_fk_relationships(ARRAY['test_pgsrx'])
    WHERE from_table = 'user_profiles'
      AND to_table = 'users'
      AND cardinality = 'o2o'
  ),
  'Detects O2O relationship: user_profiles → users'
);

-- Test M2M detection
SELECT ok(
  EXISTS(
    SELECT 1 FROM pgsrx.get_m2m_relationships(ARRAY['test_pgsrx'])
    WHERE from_table = 'users'
      AND to_table = 'organizations'
      AND junction_table = 'members'
  ),
  'Detects M2M relationship: users ↔ organizations via members'
);

-- Test O2M (inverse)
SELECT ok(
  EXISTS(
    SELECT 1 FROM pgsrx.get_inverse_relationships(ARRAY['test_pgsrx'])
    WHERE from_table = 'organizations'
      AND to_table = 'projects'
      AND cardinality = 'o2m'
  ),
  'Generates O2M inverse: organizations → projects'
);

-- Test self-referential
SELECT ok(
  EXISTS(
    SELECT 1 FROM pgsrx.get_fk_relationships(ARRAY['test_pgsrx'])
    WHERE from_table = 'categories'
      AND to_table = 'categories'
      AND is_self_relation = true
  ),
  'Detects self-referential relationship: categories → categories'
);

-- Test subscription filters for M2M
SELECT ok(
  EXISTS(
    SELECT 1 FROM pgsrx.get_subscription_filters('{
      "schema": "test_pgsrx",
      "table": "users",
      "filters": [{"column": "id", "op": "eq", "value": "1"}],
      "nested": [{"relation": "organizations"}]
    }'::jsonb)
    WHERE table_name = 'members'
  ),
  'Generates subscription filter for junction table'
);

SELECT * FROM finish();
ROLLBACK;
```

---

## Migration Path for PostgREST

### Phase 1: Parallel Operation

1. Install extension on PostgreSQL
2. PostgREST continues using Haskell introspection
3. Validate extension results match Haskell results

### Phase 2: Gradual Migration

```haskell
-- In PostgREST, add option to use extension
querySchemaCache :: AppConfig -> SQL.Session SchemaCache
querySchemaCache config = do
  if configUseExtension config
    then queryFromExtension config
    else queryFromHaskell config

queryFromExtension :: AppConfig -> SQL.Session SchemaCache
queryFromExtension config = do
  tables <- SQL.statement () [sql|
    SELECT * FROM pgsrx.get_tables($1)
  |] (configDbSchemas config)

  rels <- SQL.statement () [sql|
    SELECT * FROM pgsrx.get_relationships($1)
  |] (configDbSchemas config)

  pure $ buildSchemaCache tables rels
```

### Phase 3: Full Migration

1. Remove Haskell introspection code
2. Extension becomes single source of truth
3. Add Link header support for subscription metadata

---

## Security Considerations

1. **No superuser required** — All functions use STABLE/IMMUTABLE and query system catalogs
2. **Schema isolation** — Extension lives in `pgsrx` schema, doesn't pollute public
3. **Read-only** — Only SELECT on system catalogs, no data modification
4. **RLS compatible** — Functions respect row-level security policies
5. **Privilege escalation prevention** — SECURITY INVOKER (default), runs as calling user

---

## Performance Considerations

1. **Caching** — Optional relationship cache with DDL invalidation
2. **Parallel safe** — All functions marked PARALLEL SAFE for query parallelization
3. **Selective introspection** — Filter by schema to reduce scope
4. **Lazy loading** — Only compute relationships when needed

---

## Future Enhancements

1. **Computed relationships** — Port `allComputedRels` for function-based relationships
2. **View dependencies** — Port `allViewsKeyDependencies` for view relationship inference
3. **Partitioned table support** — Handle partition hierarchies
4. **GraphQL integration** — Generate GraphQL schema from relationships
5. **Change data capture** — Integration with pg_logical for change streaming

---

## References

- [PostgREST SchemaCache.hs](src/PostgREST/SchemaCache.hs) — Original Haskell implementation
- [pg_tle Documentation](https://github.com/aws/pg_tle) — Trusted Language Extensions
- [PostgreSQL System Catalogs](https://www.postgresql.org/docs/current/catalogs.html)
- [PLAN.md](PLAN.md) — High-level reactive queries plan
