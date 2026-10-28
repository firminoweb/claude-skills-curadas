# JSONB Operations

## JSONB vs JSON

```sql
-- Use JSONB (binary, indexed, faster)
CREATE TABLE documents (
  id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  data JSONB NOT NULL
);

-- NOT json (text storage, no indexing)
-- Only use json if you need to preserve exact formatting/whitespace
```

## JSONB Operators

### Retrieval Operators

```sql
-- -> returns JSONB
SELECT data -> 'user' FROM documents;                    -- {"id": 123, "name": "Alice"}
SELECT data -> 'user' -> 'name' FROM documents;          -- "Alice" (still JSONB)

-- ->> returns text
SELECT data ->> 'status' FROM documents;                 -- active (text)
SELECT data -> 'user' ->> 'name' FROM documents;         -- Alice (text)

-- #> for nested paths (JSONB)
SELECT data #> '{user,address,city}' FROM documents;    -- "NYC" (JSONB)

-- #>> for nested paths (text)
SELECT data #>> '{user,address,city}' FROM documents;   -- NYC (text)

-- Array access
SELECT data -> 'tags' -> 0 FROM documents;               -- First tag
SELECT jsonb_array_elements(data -> 'tags') FROM documents;  -- Expand array
```

### Containment Operators

```sql
-- @> contains (most useful for indexing)
SELECT * FROM documents WHERE data @> '{"status": "active"}';
SELECT * FROM documents WHERE data @> '{"tags": ["postgresql"]}';
SELECT * FROM documents WHERE data -> 'user' @> '{"role": "admin"}';

-- <@ is contained by
SELECT * FROM documents WHERE '{"status": "active"}' <@ data;

-- ? key exists
SELECT * FROM documents WHERE data ? 'email';
SELECT * FROM documents WHERE data -> 'user' ? 'email';

-- ?| any key exists
SELECT * FROM documents WHERE data ?| ARRAY['email', 'phone'];

-- ?& all keys exist
SELECT * FROM documents WHERE data ?& ARRAY['email', 'phone'];
```

### Modification Operators

```sql
-- || concatenate/merge (shallow)
UPDATE documents SET data = data || '{"updated_at": "2024-01-01"}'::jsonb;

-- - remove key
UPDATE documents SET data = data - 'temp_field';

-- #- remove nested path
UPDATE documents SET data = data #- '{user,temp_field}';

-- jsonb_set for deep updates
UPDATE documents
SET data = jsonb_set(data, '{user,email}', '"new@example.com"'::jsonb)
WHERE id = 123;

-- jsonb_insert
UPDATE documents
SET data = jsonb_insert(data, '{tags,0}', '"new-tag"'::jsonb)
WHERE id = 123;
```

## JSONB Indexing

### GIN Index (Default for containment)

```sql
-- Standard GIN index (for @>, ?, ?&, ?| operators)
CREATE INDEX idx_documents_data ON documents USING GIN(data);

-- Queries that benefit:
SELECT * FROM documents WHERE data @> '{"status": "active"}';
SELECT * FROM documents WHERE data ? 'email';
SELECT * FROM documents WHERE data ?& ARRAY['email', 'phone'];
```

### GIN Index on Specific Path

```sql
-- Index specific path for better performance
CREATE INDEX idx_documents_status ON documents USING GIN((data -> 'status'));
CREATE INDEX idx_documents_user ON documents USING GIN((data -> 'user'));

-- Smaller index, faster queries on specific paths
SELECT * FROM documents WHERE data -> 'status' @> '"active"';
```

### GIN Index with jsonb_path_ops

```sql
-- Smaller, faster index for @> queries only
CREATE INDEX idx_documents_path_ops ON documents USING GIN(data jsonb_path_ops);

-- Good for: WHERE data @> '{"key": "value"}'
-- Bad for: WHERE data ? 'key' (not supported)
-- ~20% smaller than default GIN, faster for containment
```

### B-tree Index on Extracted Values

```sql
-- Index extracted value (most selective)
CREATE INDEX idx_documents_status_btree ON documents((data ->> 'status'));
CREATE INDEX idx_documents_user_id ON documents((CAST(data -> 'user' ->> 'id' AS INTEGER)));

-- Enables efficient equality and range queries
SELECT * FROM documents WHERE data ->> 'status' = 'active';
SELECT * FROM documents WHERE CAST(data -> 'user' ->> 'id' AS INTEGER) > 1000;
```

### Expression Index for Nested Values

```sql
-- Index deep nested value
CREATE INDEX idx_documents_user_email ON documents((data #>> '{user,email}'));

-- Enables:
SELECT * FROM documents WHERE data #>> '{user,email}' = 'user@example.com';
```

## Query Patterns

### Filtering

```sql
-- Exact match
SELECT * FROM documents WHERE data @> '{"status": "active"}';

-- Multiple conditions
SELECT * FROM documents
WHERE data @> '{"status": "active", "verified": true}';

-- Nested conditions
SELECT * FROM documents
WHERE data -> 'user' @> '{"role": "admin"}';

-- Array containment
SELECT * FROM documents
WHERE data -> 'tags' @> '["postgresql"]';

-- Text search in JSONB value
SELECT * FROM documents
WHERE data ->> 'title' ILIKE '%postgres%';
```

### Aggregation

```sql
-- Extract and aggregate
SELECT
  data ->> 'status' as status,
  COUNT(*) as count,
  AVG(CAST(data ->> 'score' AS FLOAT)) as avg_score
FROM documents
GROUP BY data ->> 'status';

-- Array aggregation
SELECT
  jsonb_agg(data -> 'user') as users
FROM documents
WHERE data @> '{"status": "active"}';

-- Object aggregation
SELECT
  jsonb_object_agg(id, data -> 'user') as user_map
FROM documents
WHERE data ? 'user';
```

### Array Operations

```sql
-- Expand array to rows
SELECT
  id,
  jsonb_array_elements(data -> 'tags') as tag
FROM documents;

-- Expand array to text
SELECT
  id,
  jsonb_array_elements_text(data -> 'tags') as tag
FROM documents;

-- Array length
SELECT * FROM documents
WHERE jsonb_array_length(data -> 'tags') > 5;

-- Filter array elements
SELECT
  id,
  jsonb_path_query_array(data, '$.tags[*] ? (@ like_regex "^post.*" flag "i")') as postgres_tags
FROM documents;
```

## JSONB Functions

```sql
-- Build JSONB
SELECT jsonb_build_object('id', 123, 'name', 'Alice', 'active', true);
SELECT jsonb_build_array(1, 2, 'three', true);

-- Object keys
SELECT jsonb_object_keys(data) FROM documents;

-- Pretty print
SELECT jsonb_pretty(data) FROM documents;

-- Type checking
SELECT jsonb_typeof(data -> 'score') FROM documents;  -- number, string, array, object, boolean, null

-- Strip nulls
SELECT jsonb_strip_nulls(data) FROM documents;
```

## SQL/JSON (PG 17): JSON_TABLE, JSON_VALUE, JSON_EXISTS

```sql
-- Turn a JSON array into rows with typed columns
SELECT o.id, items.*
FROM orders o,
  JSON_TABLE(o.data, '$.items[*]' COLUMNS (
    sku   text    PATH '$.sku',
    qty   int     PATH '$.qty',
    price numeric PATH '$.price'
  )) AS items;

SELECT JSON_VALUE(data, '$.user.email') FROM documents;          -- scalar as text
SELECT * FROM documents WHERE JSON_EXISTS(data, '$.tags[*] ? (@ == "postgresql")');
```

## JSONB Path Queries (Postgres 12+)

```sql
-- jsonb_path_query for flexible queries
SELECT jsonb_path_query(data, '$.user.address.city') FROM documents;

-- With filters
SELECT jsonb_path_query(data, '$.items[*] ? (@.price > 100)') FROM documents;

-- Exists check
SELECT * FROM documents
WHERE jsonb_path_exists(data, '$.tags[*] ? (@ == "postgresql")');

-- Array result
SELECT jsonb_path_query_array(data, '$.items[*].name') FROM documents;
```

## Performance Best Practices

### DO

```sql
-- Use specific path indexes for hot paths
CREATE INDEX idx_docs_status ON documents((data ->> 'status'));

-- Use GIN index with path ops for containment-only queries
CREATE INDEX idx_docs_pathops ON documents USING GIN(data jsonb_path_ops);

-- Extract frequently queried values to columns
-- STORED is required to index it: PG 18's default (VIRTUAL) generated columns can't be indexed
ALTER TABLE documents ADD COLUMN status TEXT GENERATED ALWAYS AS (data ->> 'status') STORED;
CREATE INDEX idx_docs_status_col ON documents(status);

-- Use @> for indexed queries
SELECT id FROM documents WHERE data @> '{"status": "active"}';  -- Fast with GIN index
```

### DON'T

```sql
-- Match the stored JSON type: a number in the document needs a number in @>
SELECT id FROM documents WHERE data @> '{"score": "100"}';  -- Wrong: matches only the string "100"
SELECT id FROM documents WHERE data @> '{"score": 100}';    -- Right, and uses the GIN index
-- (data ->> 'score')::int = 100 also works but ignores the GIN index: it needs its own expression index

-- Don't query without indexes
SELECT * FROM documents WHERE data -> 'nested' -> 'deep' ->> 'value' = 'x';
-- Add index: CREATE INDEX ON documents((data #>> '{nested,deep,value}'));

-- Don't store huge arrays in JSONB
-- If you have 10k+ elements, use a separate table

-- Don't use JSONB for high-update columns
-- Extract to regular column if updated frequently

-- Don't put relational data in JSONB because the schema is "flexible"
-- Values you filter, join, sort or constrain on belong in typed columns
```

## Schema Validation (CHECK constraints)

```sql
-- Using CHECK constraints
ALTER TABLE documents
ADD CONSTRAINT check_data_schema
CHECK (
  jsonb_typeof(data) = 'object' AND
  data ? 'id' AND
  data ? 'status' AND
  data ->> 'status' IN ('active', 'pending', 'archived')
);
```

## Migration Patterns

```sql
-- Add JSONB column
ALTER TABLE users ADD COLUMN metadata JSONB DEFAULT '{}'::jsonb;

-- Migrate existing columns to JSONB
UPDATE users SET metadata = jsonb_build_object(
  'preferences', preferences,
  'settings', settings,
  'flags', flags
);

-- Drop old columns after validation
ALTER TABLE users DROP COLUMN preferences, DROP COLUMN settings, DROP COLUMN flags;
```
