# Managed Postgres: Neon and alternatives

## Choosing

| Provider | Best when | Watch out |
|----------|-----------|-----------|
| **Neon** | Default for apps on Render/Railway/Vercel/Cloudflare: branching per PR, scale to zero, generous free tier | Cold start of a few seconds after idle; pooled vs direct URLs |
| **Render Postgres** | App on Render and you want one bill and the private network | Free DB expires after 30 days; no branching |
| **Railway Postgres** | App on Railway; private networking (`*.railway.internal`) | Runs on a volume you manage (size, backups settings); check backup retention |
| **Supabase** | You also want auth, storage, realtime or PostgREST | Pooler ports differ by mode; RLS matters when clients query directly |

Put the database in the **same region** as the app. Cross-region adds latency to every query, and a request that runs 20 queries pays it 20 times.

## Neon

### Two connection strings

| Variable | Host | Used by |
|----------|------|---------|
| `DATABASE_URL` | `ep-xxx-pooler.<region>.aws.neon.tech` (pooled, PgBouncer transaction mode) | The app at runtime |
| `DIRECT_URL` (Neon's integrations call it `DATABASE_URL_UNPOOLED`) | `ep-xxx.<region>.aws.neon.tech` (direct) | Migrations, `pg_dump`, anything needing session state |

Both with `?sslmode=require`. Add `&connect_timeout=15` if the compute scales to zero, so the first connection after idle doesn't fail (Prisma P1001).

### Prisma 7

```ts
// prisma.config.ts — the CLI (migrate) uses the direct URL
import 'dotenv/config';
import { defineConfig, env } from 'prisma/config';

export default defineConfig({
  schema: 'prisma/schema.prisma',
  migrations: { path: 'prisma/migrations' },
  datasource: { url: env('DIRECT_URL') },
});
```

```ts
// Runtime: pooled URL
// Long-running server (NestJS on Render/Railway): @prisma/adapter-pg over TCP
new PrismaClient({ adapter: new PrismaPg({ connectionString: process.env.DATABASE_URL }) });

// Serverless/edge (Vercel functions, Cloudflare Workers): @prisma/adapter-neon (WebSocket/HTTP driver)
new PrismaClient({ adapter: new PrismaNeon({ connectionString: process.env.DATABASE_URL }) });
```

Cloudflare Workers + Neon: also consider Hyperdrive in front of the direct URL (see `nextjs-developer` → `deploy-cloudflare.md`).

### Branching

- A branch is a copy-on-write clone of the database (schema + data) created in seconds.
- **Preview per PR:** create a branch from `main` (Neon's Vercel integration or GitHub Action does it automatically), run `prisma migrate deploy` against it, point the preview deployment at it, and delete the branch when the PR closes.
- **Safe migration rehearsal:** branch production, run the migration there, time it, then run it on production.
- Branches copy real data. For previews that external people can open, use a branch with anonymized/seed data (or a schema-only branch), not production PII.

### Scale to zero and limits

- Production: disable scale to zero (or set a longer suspend timeout) when cold starts matter; set autoscaling min/max compute.
- Point-in-time restore window depends on the plan. Check it, and test a restore into a branch.
- Pooled connections don't support session features: `SET` (outside a transaction), `LISTEN/NOTIFY`, session advisory locks, temp tables across statements. Use the direct URL for those.

## Render Postgres

- Use the **internal** connection string for services in the same region; the external one only for local tools.
- `render.yaml`: `databases:` block, attached with `fromDatabase` (check property names in the Blueprint spec).
- Paid plans include daily backups and PITR on higher tiers. Free has no backups and expires after 30 days.

## Railway Postgres

- Reference it with `${{Postgres.DATABASE_URL}}` (dashboard) or `db.env.DATABASE_URL` (`.railway/railway.ts`). The default URL uses the private network.
- It's a container with a volume: watch disk usage and configure backups in the service settings.

## Supabase

- Session pooler (port 5432) or direct connection for migrations; transaction pooler (port 6543) for serverless runtime traffic.
- If the frontend talks to Supabase directly (supabase-js), every exposed table needs RLS policies. A table without RLS is readable by anyone with the anon key.

## Moving between providers

`pg_dump --format=custom --no-owner --no-privileges "$SOURCE_DIRECT_URL" > db.dump` then `pg_restore --no-owner --dbname "$TARGET_DIRECT_URL" db.dump`. Use direct URLs, the target's same or newer major version, and verify row counts and `prisma migrate status` afterwards. For near-zero downtime use logical replication (if both providers allow it).
