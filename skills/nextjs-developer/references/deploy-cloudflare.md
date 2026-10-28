# Deploying to Cloudflare Workers (OpenNext)

Next.js runs on Workers through `@opennextjs/cloudflare`. It uses the Node.js runtime with `nodejs_compat`, **not** the Edge runtime. `@cloudflare/next-on-pages` is the old Edge-only path; migrate away from it.

Checked against adapter 1.20.12 (Oct 2026), whose peer range is `next >=15.5.28 <16 || >=16.4.1` and `wrangler ^4.148.0`. Re-check `npm view @opennextjs/cloudflare peerDependencies` before pinning versions.

## Setup

```bash
# New app
npm create cloudflare@latest -- my-app --framework=next --platform=workers

# Existing app (creates wrangler.jsonc, open-next.config.ts, scripts, R2 bucket if available)
npx @opennextjs/cloudflare migrate
```

Manual equivalent:

```bash
npm i @opennextjs/cloudflare@latest
npm i -D wrangler@latest
```

```jsonc
// package.json scripts
{
  "dev": "next dev",
  "build": "next build",
  "preview": "opennextjs-cloudflare build && opennextjs-cloudflare preview",
  "deploy": "opennextjs-cloudflare build && opennextjs-cloudflare deploy",
  "upload": "opennextjs-cloudflare build && opennextjs-cloudflare upload",
  "cf-typegen": "wrangler types --env-interface CloudflareEnv cloudflare-env.d.ts"
}
```

```ts
// next.config.ts
import type { NextConfig } from 'next'
import { initOpenNextCloudflareForDev } from '@opennextjs/cloudflare'

const nextConfig: NextConfig = {}

export default nextConfig

initOpenNextCloudflareForDev() // exposes bindings (D1, R2, KV…) to `next dev`
```

Also:
- Add `.open-next` to `.gitignore`.
- `.dev.vars` holds local secrets and `NEXTJS_ENV=development`. It is **not** uploaded on deploy.
- Add `public/_headers` with `/_next/static/*` → `Cache-Control: public,max-age=31536000,immutable`.

## wrangler.jsonc

```jsonc
{
  "$schema": "node_modules/wrangler/config-schema.json",
  "name": "my-app",
  "main": ".open-next/worker.js",
  "compatibility_date": "2026-10-01",            // recent date; must be >= 2025-04-01 for process.env
  "compatibility_flags": ["nodejs_compat", "global_fetch_strictly_public"],
  "assets": { "directory": ".open-next/assets", "binding": "ASSETS" },
  "services": [{ "binding": "WORKER_SELF_REFERENCE", "service": "my-app" }], // must equal "name"
  "images": { "binding": "IMAGES" },             // next/image via Cloudflare Images
  "r2_buckets": [{ "binding": "NEXT_INC_CACHE_R2_BUCKET", "bucket_name": "my-app-cache" }],
  "d1_databases": [
    { "binding": "NEXT_TAG_CACHE_D1", "database_name": "my-app-tags", "database_id": "<id>" },
    { "binding": "DB", "database_name": "my-app", "database_id": "<id>" }
  ],
  "durable_objects": {
    "bindings": [{ "name": "NEXT_CACHE_DO_QUEUE", "class_name": "DOQueueHandler" }]
  },
  "migrations": [{ "tag": "v1", "new_sqlite_classes": ["DOQueueHandler"] }],
  "observability": { "enabled": true }
}
```

After changing bindings, run `npm run cf-typegen`. If the generated global types clash with DOM types in Next.js, keep `cloudflare-env.d.ts` limited to the `CloudflareEnv` interface.

## Caching (required for ISR, `revalidateTag`, `'use cache'` revalidation)

Without these overrides, revalidation does nothing in production, and `next dev` won't show the problem.

```ts
// open-next.config.ts — small/medium app
import { defineCloudflareConfig } from '@opennextjs/cloudflare'
import r2IncrementalCache from '@opennextjs/cloudflare/overrides/incremental-cache/r2-incremental-cache'
import doQueue from '@opennextjs/cloudflare/overrides/queue/do-queue'
import d1NextTagCache from '@opennextjs/cloudflare/overrides/tag-cache/d1-next-tag-cache'

export default defineCloudflareConfig({
  incrementalCache: r2IncrementalCache, // stores ISR / cache entries
  queue: doQueue,                       // time-based revalidation
  tagCache: d1NextTagCache,             // revalidateTag / revalidatePath / updateTag
})
```

| Need | Use |
|------|-----|
| Fully static site, no revalidation | `staticAssetsIncrementalCache` + `enableCacheInterception: true` (no queue/tag cache) |
| Small/medium app | R2 + `doQueue` + `d1NextTagCache` (D1 needs the `revalidations` table — see OpenNext caching docs) |
| High traffic | `withRegionalCache(r2IncrementalCache, { mode: 'long-lived' })` + `doShardedTagCache({ baseShardSize: 12 })` + `purgeCache({ type: 'direct' })` (needs a custom domain zone and `CACHE_PURGE_API_TOKEN`/`CACHE_PURGE_ZONE_ID`) |
| Avoid | Workers KV as incremental cache (eventually consistent), `memoryQueue` in production |

Don't use tag cache if the app never calls `revalidateTag`/`revalidatePath`/`updateTag`.

## Accessing bindings (D1, R2, KV, Queues, AI…)

```ts
// lib/db.ts
import { getCloudflareContext } from '@opennextjs/cloudflare'
import { drizzle } from 'drizzle-orm/d1'
import * as schema from './schema'

export function getDb() {
  const { env } = getCloudflareContext() // request scope: Server Components, actions, route handlers
  return drizzle(env.DB, { schema })
}

// In static generation / generateStaticParams (no request):
// const { env } = await getCloudflareContext({ async: true })
```

Rules specific to Workers:
- **Create DB clients per request**, not as a module-level singleton with a pool. Workers can't reuse I/O objects across requests. For Postgres, use Hyperdrive (`env.HYPERDRIVE.connectionString`) and a new client per request. Watch open issue opennextjs-cloudflare#1322 (Hyperdrive + `pg`/`@prisma/adapter-pg`).
- Secrets: `wrangler secret put NAME` (production). Use `.dev.vars` locally. Plain vars go in `vars` in `wrangler.jsonc`.
- `NEXT_PUBLIC_*` is inlined at **build** time, so it must be set in the build environment (CI or Workers Builds).

## Proxy / middleware

- Remove every `export const runtime = 'edge'`. The adapter doesn't support the Edge runtime.
- `proxy.ts` (Node.js middleware) support landed in Aug 2026 and still has open bugs. It can add ~4.6 MB to the bundle even when empty (#1373), and it fails with `@opentelemetry/api` installed, e.g. via `@sentry/nextjs` (#1400). Keep proxy minimal, or do auth checks in layouts/actions instead.

## Limits and checks

- Worker size: **3 MiB** compressed (Free) / **10 MiB** (Paid). Wrangler prints both sizes on deploy, and only the compressed size counts. Heavy server deps (ORMs with engines, SDKs, `proxy.ts`) are the usual cause.
- CPU time per request is limited. Move long jobs to Queues/Workflows instead of doing them inside a request.
- Test with `npm run preview` (real `workerd` runtime) before deploying. `next dev` runs Node and hides runtime gaps.
- Deploy: `npm run deploy`, or connect the repo in Workers Builds (CI deploys each merge to the production branch; preview URLs per branch with `upload`).
- A 404 or error 1042 for a short time right after the first deploy is propagation. Wait before debugging.
