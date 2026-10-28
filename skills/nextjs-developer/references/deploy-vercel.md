# Deploying to Vercel

Vercel builds Next.js with zero config. Cache Components, ISR, `revalidateTag`/`updateTag`, `proxy.ts`, image optimization and streaming need no adapter setup.

## Flow

```bash
npm i -g vercel
vercel link                 # connect the folder to a project
vercel env pull .env.local  # pull Development env vars locally
vercel                      # preview deployment
vercel --prod               # production
```

Usually you just connect the Git repo. Each branch/PR gets a preview URL and each merge to the production branch deploys.

## Environment variables

- Set them per environment (Development / Preview / Production) in the dashboard or with `vercel env add`.
- Server-only values are read via `process.env.X` in server code. Never prefix secrets with `NEXT_PUBLIC_`, because that inlines them into the client bundle at **build** time.
- Changing an env var requires a redeploy to take effect.
- To read a variable at request time rather than build time, call `await connection()` first (see `caching.md`).

## Functions

- Node.js runtime by default, on Fluid compute (instances handle concurrent requests). Don't add `runtime = 'edge'` "for speed". It drops Node APIs and gains nothing for DB-bound pages.
- Raise the timeout per route when needed: `export const maxDuration = 60` (limit depends on plan).
- Module-level DB clients are fine (and preferred) on Fluid compute. Use the provider's serverless/pooled connection string (Neon, Supabase pooler, Prisma Accelerate).
- Use `after()` from `next/server` for work after the response (logging, analytics) instead of fire-and-forget promises.

## Storage and integrations

Vercel Blob for files, and Marketplace integrations (Neon/Supabase Postgres, Upstash Redis) inject env vars automatically. Configure cron jobs in `vercel.json` (`crons`) and protect them by checking `CRON_SECRET`.

## Checks before production

- `next build` passes locally (no lint step anymore, so run ESLint/Biome separately in CI).
- Preview deployment tested with Production-like env vars.
- `images.remotePatterns` covers every remote image host (`images.domains` is deprecated).
- Measure with Lighthouse / Speed Insights. `next build` no longer prints First Load JS sizes.

## Self-hosting (neither Vercel nor Cloudflare)

`output: 'standalone'` in `next.config.ts` gives a minimal Node server for Docker. Copy `.next/standalone`, `.next/static` and `public`. With several instances, configure a shared `cacheHandlers` (e.g. Redis), the same `deploymentId` and the same `NEXT_SERVER_ACTIONS_ENCRYPTION_KEY`. Otherwise caches and Server Actions diverge between instances.
