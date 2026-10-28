---
name: postgres-pro
description: Use when designing PostgreSQL schemas, writing or reviewing migrations, diagnosing slow queries (EXPLAIN, pg_stat_statements), choosing indexes, working with JSONB, full-text search, pg_trgm or pgvector, tuning VACUUM/autovacuum, configuring connection pooling, replication, backups, upgrading Postgres versions, or setting up hosted Postgres (Neon, Render, Railway, Supabase) — including databases accessed through Prisma or another ORM.
license: MIT
metadata:
  author: https://github.com/Jeffallan (updated by curator)
  version: "2.0.0"
  baseline: "PostgreSQL 18 (19 GA expected 2026-11-19), pgvector 0.8, PgBouncer 1.26 (Oct 2026)"
  domain: infrastructure
  triggers: PostgreSQL, Postgres, Neon, Supabase, EXPLAIN ANALYZE, pg_stat_statements, index, migration, JSONB, pgvector, VACUUM, replication, PgBouncer
  related-skills: nestjs-expert
  role: specialist
  scope: implementation
  output-format: code
---

# PostgreSQL Pro

## First: read the environment

1. **Version:** `SELECT version();`. Supported majors: 14 (EOL 2026-11-12), 15, 16, 17, 18. 19 is RC with GA planned for 2026-11-19. Don't use a feature newer than the server (version map below).
2. **Where it runs:** managed (Neon, Supabase, RDS/Aurora, Cloud SQL, Prisma Postgres) or self-hosted. Managed services decide replication, backups, extensions and most `postgresql.conf` settings, so skip those sections and use the provider's console/API.
3. **Who owns the schema:** an ORM's migrations (Prisma `prisma/migrations`, Drizzle, TypeORM) or raw SQL files. Write changes in that tool, and edit its generated SQL when it can't express something (partial or concurrent indexes, `RENAME`, constraints).
4. **Connection path:** direct, PgBouncer/Supavisor (transaction mode), or Hyperdrive. Transaction pooling changes what's safe (session `SET`, advisory locks, `LISTEN`).

## Outdated habits

| Outdated | Current |
|----------|---------|
| `id SERIAL` | `id bigint GENERATED ALWAYS AS IDENTITY` |
| `uuid-ossp` + `uuid_generate_v4()` | Built-in `gen_random_uuid()` (13+), `uuidv7()` for time-ordered keys (18) |
| `timestamp` | `timestamptz` (stores an instant; display in the client's zone) |
| `varchar(255)` by habit | `text` + `CHECK (char_length(x) <= n)` when there's a real limit |
| `crypt(password, gen_salt('bf'))` in SQL | Hash in the application (argon2id). Plaintext in SQL ends up in logs |
| `EXPLAIN (ANALYZE, BUFFERS)` | Same, but 18 already includes `BUFFERS` with `ANALYZE` |
| "A multicolumn index is useless without its leading column" | 18 has B-tree skip scan when the leading column has few distinct values. Still verify with `EXPLAIN` |
| `OFFSET` pagination on large tables | Keyset: `WHERE (created_at, id) < ($1, $2) ORDER BY created_at DESC, id DESC LIMIT n` |
| `VACUUM FULL` in production | `pg_repack` (19 adds `REPACK (CONCURRENTLY)`) |
| `md5` auth in `pg_hba.conf` | `scram-sha-256` (`md5` is deprecated in 18) |
| pgvector `ivfflat` by default | `hnsw`; `halfvec` for > 2,000 dimensions; `hnsw.iterative_scan` when filtering |
| Prepared statements disabled for PgBouncer | PgBouncer ≥ 1.21 supports them in transaction mode (`max_prepared_statements`) |

## Version map

| Feature | Since |
|---------|-------|
| `gen_random_uuid()` built in, `REINDEX CONCURRENTLY` | 12–13 |
| `MERGE`, `UNIQUE NULLS NOT DISTINCT`, publication row filters | 15 |
| `JSON_TABLE`, SQL/JSON functions, `MERGE ... RETURNING`, incremental backup | 17 |
| `uuidv7()`, virtual generated columns (default), `OLD`/`NEW` in `RETURNING`, skip scan, `WITHOUT OVERLAPS`, async I/O, stats kept by `pg_upgrade` | 18 |

## Core workflow: slow query → fix → proof

```sql
-- 1. Find what costs the most in total, not only the slowest single call
SELECT queryid, calls, round(total_exec_time) AS total_ms, round(mean_exec_time, 1) AS mean_ms, rows, query
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;

-- 2. Plan with real numbers (run on a copy or inside BEGIN ... ROLLBACK for writes)
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, total FROM orders WHERE customer_id = 42 AND status = 'pending' ORDER BY created_at DESC LIMIT 20;
-- Look for: estimated vs actual rows far apart (stale stats), Seq Scan on a big table,
-- "Rows Removed by Filter" in the thousands, Sort spilling to disk

-- 3. Index that matches filter + order
CREATE INDEX CONCURRENTLY idx_orders_customer_pending
  ON orders (customer_id, created_at DESC)
  WHERE status = 'pending';

-- 4. Same EXPLAIN again: confirm the index is used and the time dropped
-- 5. After bulk loads or big deletes
ANALYZE orders;
```

`CREATE INDEX CONCURRENTLY` can't run inside a transaction block. If the migration tool wraps files in a transaction, put it in its own migration and check how the tool runs it. If it fails halfway it leaves an `INVALID` index: drop it and retry.

## Safe migrations on a live database

- `SET lock_timeout = '5s';` at the top of DDL migrations, so a blocked `ALTER TABLE` fails instead of queueing every query behind it.
- Adding a column with a constant default is instant (11+). Adding `NOT NULL` to an existing column scans the table: add `CHECK (col IS NOT NULL) NOT VALID`, then `VALIDATE CONSTRAINT`, then `SET NOT NULL`.
- Foreign keys on big tables: `ADD CONSTRAINT ... NOT VALID`, then `VALIDATE CONSTRAINT` in a separate step.
- Renames and type changes break running app versions: expand (new column) → dual-write/backfill in batches → switch reads → contract.
- Index every foreign key column you filter or join on. Postgres doesn't create them automatically.

## Rules

- Parameterized queries only (`$1`, ORM placeholders). Never string-concatenate user input, including in `ORDER BY` (allow-list the column names).
- Select the columns you need, not `SELECT *`, in application queries.
- Money in `numeric`, never `float`. Constraints in the database (`NOT NULL`, `CHECK`, `UNIQUE`, FKs), not only in app validation.
- Use connection pooling for anything serverless or with many app instances. Keep `max_connections` modest and let the pooler multiplex.
- Least privilege: the app role owns no schema and has no superuser. For multi-tenant data exposed to clients (Supabase/PostgREST), enable Row-Level Security and test policies with the client's role.
- Don't disable autovacuum. Tune it per table for high-churn tables.
- Large files go to object storage. Store the key in the database.
- Back up and **test restores**. A backup never restored is a hope, not a backup.

## References

| Topic | File | Load when |
|-------|------|-----------|
| Performance | `references/performance.md` | EXPLAIN, index types, statistics, config tuning, pagination |
| JSONB | `references/jsonb.md` | JSONB operators, GIN, SQL/JSON, when to use columns instead |
| Extensions | `references/extensions.md` | pg_stat_statements, pg_trgm, PostGIS, pgvector, pgcrypto, timescaledb |
| Maintenance | `references/maintenance.md` | VACUUM, bloat, locks, wraparound, monitoring views |
| Replication and backups | `references/replication.md` | Streaming/logical replication, failover, PITR, PgBouncer, major upgrades |
| Hosting | `references/hosting.md` | Neon (pooled/direct URLs, branches, Prisma 7), Render/Railway Postgres, Supabase, moving providers |

## Verify before claiming done

1. `EXPLAIN (ANALYZE, BUFFERS)` before and after, with the numbers in the answer.
2. Migrations ran against a copy of production-sized data, or the lock impact is explained.
3. New indexes are used (`pg_stat_user_indexes.idx_scan` grows) and no index is left `INVALID`.
4. Config changes list the old value, the new value and whether they need a restart (`pg_settings.context`).
