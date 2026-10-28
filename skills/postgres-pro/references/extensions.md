# PostgreSQL Extensions

## Extension Management

```sql
-- List available extensions
SELECT * FROM pg_available_extensions ORDER BY name;

-- List installed extensions
SELECT * FROM pg_extension;

-- Install extension
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
CREATE EXTENSION IF NOT EXISTS pg_trgm;

-- Drop extension
DROP EXTENSION pg_stat_statements;

-- Update extension
ALTER EXTENSION pg_stat_statements UPDATE;  -- to the version shipped with the server
```

## pg_stat_statements (Query Performance)

```sql
-- Install and configure
CREATE EXTENSION pg_stat_statements;

-- postgresql.conf:
-- shared_preload_libraries = 'pg_stat_statements'
-- pg_stat_statements.max = 10000
-- pg_stat_statements.track = all

-- Top 10 slowest queries by mean time
SELECT
  query,
  calls,
  total_exec_time,
  mean_exec_time,
  max_exec_time,
  stddev_exec_time,
  rows
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;

-- Most frequently called queries
SELECT
  query,
  calls,
  total_exec_time,
  mean_exec_time
FROM pg_stat_statements
ORDER BY calls DESC
LIMIT 10;

-- Most time-consuming queries (total time)
SELECT
  query,
  calls,
  total_exec_time / 1000 as total_seconds,
  mean_exec_time,
  (total_exec_time / sum(total_exec_time) OVER ()) * 100 as percentage
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;

-- Reset statistics
SELECT pg_stat_statements_reset();
```

## UUIDs (no extension needed)

```sql
-- gen_random_uuid() is built in since PG 13; uuidv7() since PG 18
-- uuid-ossp is legacy: uuid_generate_v1() leaks the MAC address
SELECT gen_random_uuid();  -- v4, random
SELECT uuidv7();           -- time-ordered: better B-tree locality for primary keys

CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT uuidv7(),  -- gen_random_uuid() before PG 18, or generate v7 in the app
  email TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Insert
INSERT INTO users (email) VALUES ('user@example.com')
RETURNING id;
```

## pg_trgm (Fuzzy String Matching)

```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;

-- Similarity search
SELECT
  email,
  similarity(email, 'john@example.com') as sim
FROM users
WHERE similarity(email, 'john@example.com') > 0.3
ORDER BY sim DESC;

-- LIKE optimization with trigram index
CREATE INDEX idx_users_email_trgm ON users USING GIN(email gin_trgm_ops);

-- Now these queries use index:
SELECT * FROM users WHERE email ILIKE '%john%';
SELECT * FROM users WHERE email % 'jon@example.com';  -- Similar to

-- Trigram operators
SELECT 'hello' % 'helo';              -- True (similar)
SELECT similarity('hello', 'helo');   -- 0.5
SELECT word_similarity('hello', 'hello world');  -- 1.0

-- Set similarity threshold
SET pg_trgm.similarity_threshold = 0.5;
SELECT * FROM users WHERE email % 'searchtext';
```

## PostGIS (Spatial and Geographic)

```sql
CREATE EXTENSION IF NOT EXISTS postgis;

-- Create spatial table
CREATE TABLE locations (
  id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  name TEXT NOT NULL,
  geom GEOMETRY(Point, 4326)  -- WGS84 lat/lng
);

-- Add spatial index
CREATE INDEX idx_locations_geom ON locations USING GIST(geom);

-- Insert point (longitude, latitude)
INSERT INTO locations (name, geom)
VALUES ('NYC', ST_SetSRID(ST_MakePoint(-74.0060, 40.7128), 4326));

-- Distance queries (in meters)
SELECT
  name,
  ST_Distance(
    geom::geography,
    ST_SetSRID(ST_MakePoint(-73.9857, 40.7484), 4326)::geography
  ) as distance_meters
FROM locations
ORDER BY distance_meters
LIMIT 10;

-- Within radius (1km = 1000m)
SELECT * FROM locations
WHERE ST_DWithin(
  geom::geography,
  ST_SetSRID(ST_MakePoint(-74.0060, 40.7128), 4326)::geography,
  1000
);

-- Bounding box query (very fast with GIST index)
SELECT * FROM locations
WHERE geom && ST_MakeEnvelope(-74.1, 40.6, -73.9, 40.8, 4326);

-- Contains query
SELECT * FROM zones
WHERE ST_Contains(geom, ST_SetSRID(ST_MakePoint(-74.0060, 40.7128), 4326));

-- Area calculation
SELECT
  name,
  ST_Area(geom::geography) / 1000000 as area_km2
FROM zones;

-- GeoJSON export
SELECT
  name,
  ST_AsGeoJSON(geom) as geojson
FROM locations;
```

## pgvector (Vector Similarity Search)

```sql
CREATE EXTENSION IF NOT EXISTS vector;

-- Create table with vector column
CREATE TABLE embeddings (
  id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  tenant_id bigint NOT NULL,
  content TEXT,
  embedding vector(1536)  -- must match the embedding model's dimensions
);

-- HNSW: best recall/speed trade-off, no training step (build after loading data)
CREATE INDEX ON embeddings USING hnsw (embedding vector_cosine_ops);
-- IVFFlat: smaller and faster to build, needs data before CREATE INDEX
-- Index limits: vector up to 2,000 dims; for more use halfvec (up to 4,000):
-- CREATE INDEX ON embeddings USING hnsw ((embedding::halfvec(3072)) halfvec_cosine_ops);

-- Insert vectors
INSERT INTO embeddings (content, embedding)
VALUES ('Hello world', '[0.1, 0.2, 0.3, ...]');

-- Similarity search (cosine distance)
SELECT
  content,
  1 - (embedding <=> '[0.1, 0.2, ...]') as similarity
FROM embeddings
ORDER BY embedding <=> '[0.1, 0.2, ...]'
LIMIT 10;

-- Distance operators
-- <-> L2 distance (Euclidean)
-- <#> negative inner product
-- <=> cosine distance (most common for embeddings)

-- Set index parameters for better recall
SET hnsw.ef_search = 100;  -- Higher = better recall, slower query

-- Filtering (WHERE tenant_id = ...) happens after the approximate scan and can return
-- fewer rows than LIMIT. pgvector 0.8+: keep scanning until enough rows match
SET hnsw.iterative_scan = relaxed_order;
-- Also index the filter column; for a few distinct values use partial HNSW indexes

-- Bulk insert optimization
SET maintenance_work_mem = '2GB';
```

## pgcrypto (Encryption and Hashing)

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- Passwords: hash in the application (argon2id), not with crypt() in SQL.
-- The plaintext would travel in the query text and can land in logs and pg_stat_statements.

-- Random bytes (gen_random_uuid() no longer needs pgcrypto)
SELECT gen_random_bytes(32);

-- Encrypt/decrypt data: same problem, the key is part of the query text.
-- Prefer app-side encryption (KMS) for sensitive columns; if you must use pgcrypto,
-- pass the key as a bind parameter and keep log_statement off for that role.
SELECT pgp_sym_decrypt(encrypted_column, $1) FROM table_name;

-- Digest functions
SELECT digest('data', 'sha256');
SELECT encode(digest('data', 'sha256'), 'hex');
```

## postgres_fdw (Foreign Data Wrapper)

```sql
CREATE EXTENSION IF NOT EXISTS postgres_fdw;

-- Create foreign server
CREATE SERVER remote_db
FOREIGN DATA WRAPPER postgres_fdw
OPTIONS (host 'remote-host', port '5432', dbname 'remote_db');

-- User mapping
CREATE USER MAPPING FOR current_user
SERVER remote_db
OPTIONS (user 'remote_user', password '<remote-db-password>');

-- Import foreign schema
IMPORT FOREIGN SCHEMA public
FROM SERVER remote_db
INTO remote_schema;

-- Or create specific foreign table
CREATE FOREIGN TABLE remote_users (
  id INTEGER,
  email TEXT,
  created_at TIMESTAMPTZ
)
SERVER remote_db
OPTIONS (schema_name 'public', table_name 'users');

-- Query remote table (transparent)
SELECT * FROM remote_users WHERE created_at > NOW() - INTERVAL '1 day';

-- Join local and remote tables
SELECT
  l.id,
  l.name,
  r.email
FROM local_table l
JOIN remote_users r ON l.user_id = r.id;
```

## pg_repack (Online Table Reorganization)

```sql
CREATE EXTENSION IF NOT EXISTS pg_repack;

-- Repack table (removes bloat, rebuilds indexes)
-- Run via command line, not SQL:
-- pg_repack -d mydb -t users

-- PG 19 adds a built-in REPACK (CONCURRENTLY); until then pg_repack needs the extension
-- installed on the server and a primary key or unique index on the table

-- Check bloat before repack
SELECT
  schemaname,
  tablename,
  pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) as size,
  pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename) -
                 pg_relation_size(schemaname||'.'||tablename)) as index_size
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;

-- Repack entire database
-- pg_repack -d mydb

-- Repack with custom order
-- pg_repack -d mydb -t users -o "created_at DESC"
```

## timescaledb (Time-Series Data)

```sql
CREATE EXTENSION IF NOT EXISTS timescaledb;

-- Create hypertable (must have time column)
CREATE TABLE metrics (
  time TIMESTAMPTZ NOT NULL,
  device_id INTEGER,
  temperature DOUBLE PRECISION,
  humidity DOUBLE PRECISION
);

-- Convert to hypertable
SELECT create_hypertable('metrics', 'time');

-- Set chunk interval (default 7 days)
SELECT set_chunk_time_interval('metrics', INTERVAL '1 day');

-- Add compression
ALTER TABLE metrics SET (
  timescaledb.compress,
  timescaledb.compress_segmentby = 'device_id',
  timescaledb.compress_orderby = 'time DESC'
);

-- Automatic compression policy (compress chunks older than 7 days)
SELECT add_compression_policy('metrics', INTERVAL '7 days');

-- Retention policy (drop chunks older than 30 days)
SELECT add_retention_policy('metrics', INTERVAL '30 days');

-- Continuous aggregates (materialized views for time-series)
CREATE MATERIALIZED VIEW metrics_hourly
WITH (timescaledb.continuous) AS
SELECT
  time_bucket('1 hour', time) AS bucket,
  device_id,
  AVG(temperature) as avg_temp,
  MAX(temperature) as max_temp,
  MIN(temperature) as min_temp
FROM metrics
GROUP BY bucket, device_id;

-- Refresh policy
SELECT add_continuous_aggregate_policy('metrics_hourly',
  start_offset => INTERVAL '3 hours',
  end_offset => INTERVAL '1 hour',
  schedule_interval => INTERVAL '1 hour'
);
```

## Extension Recommendations by Use Case

**Query Performance Monitoring:**
- `pg_stat_statements` (essential)
- `pg_stat_kcache` (cache hit statistics)

**Text Search:**
- `pg_trgm` (fuzzy matching, LIKE optimization)
- Built-in full-text search (no extension needed)

**Spatial Data:**
- `postgis` (comprehensive spatial features)

**Vector Embeddings / AI:**
- `pgvector` (for semantic search, RAG applications)

**Time-Series:**
- `timescaledb` (automatic partitioning, compression)

**Data Security:**
- `pgcrypto` (hashing, encryption)
- `pgaudit` (audit logging)

**UUID Support:**
- Built in: `gen_random_uuid()` (13+), `uuidv7()` (18). `uuid-ossp` is no longer needed

**Cross-Database Queries:**
- `postgres_fdw` (query remote PostgreSQL)
- `file_fdw` (query CSV files)

**Table Maintenance:**
- `pg_repack` (online bloat removal)
