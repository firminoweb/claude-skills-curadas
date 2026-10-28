# Deploy: Render and Railway (NestJS + Prisma or MongoDB)

Both run the app as a long-lived Node process (not serverless), so `PrismaService` with `@prisma/adapter-pg` and a normal pool works as-is. Database hosting (Neon, Render/Railway Postgres, Supabase): the `postgres-pro` skill, hosting section.

## App requirements (both platforms)

```typescript
// main.ts
async function bootstrap() {
  const app = await NestFactory.create<NestExpressApplication>(AppModule, { bufferLogs: true });
  configureApp(app);              // pipes, filters, prefix (shared with E2E tests)
  app.set('trust proxy', 1);      // behind the platform's load balancer: real client IP for throttler/logs
  app.enableShutdownHooks();      // SIGTERM → onModuleDestroy → Prisma disconnects
  await app.listen(process.env.PORT ?? 3000, '0.0.0.0'); // both platforms inject PORT
}
```

```typescript
// health/health.controller.ts — @nestjs/terminus
@Controller('health')
export class HealthController {
  constructor(private health: HealthCheckService, private db: PrismaHealthIndicator, private prisma: PrismaService) {}

  @Public() // skip the global JWT guard
  @Get()
  @HealthCheck()
  check() {
    return this.health.check([() => this.db.pingCheck('database', this.prisma)]);
  }
}
```

- The health check gates zero-downtime deploys: a new version that can't reach the database never receives traffic. Keep it fast (a ping, not business queries) and unauthenticated, and return nothing sensitive.
- `ConfigModule.forRoot({ validationSchema })` must fail at boot on a missing variable, so a bad deploy dies at the health check instead of at the first request.
- `package.json`: `"engines": { "node": "22.x" }` (or 24) so the platform doesn't pick a different Node major. NestJS 12 needs ≥ 20.19.

### Build and migrations

| Step | Command |
|------|---------|
| Build | `npm ci && npx prisma generate && npm run build` (the generated client is gitignored) |
| Migrate | `npx prisma migrate deploy`, as a **pre-deploy** step: runs once per deploy, before traffic switches |
| Start | `node dist/main` (not `nest start`, not `npm run start:dev`) |

- `prisma` is a devDependency. Keep dev deps installed during the build (don't set `NODE_ENV=production` for the install step), or move `prisma` to dependencies.
- Migrations must be backward-compatible with the version still running (expand → contract), because the old instances serve traffic while the new ones start.
- No pre-deploy available (Render free instances)? `npx prisma migrate deploy && node dist/main` as the start command works: `migrate deploy` takes an advisory lock, so parallel instances don't race. A failed migration then crash-loops the service instead of blocking the deploy.
- `DATABASE_URL` for the app and a direct (non-pooled) URL for migrations when the database has a pooler (Neon, Supabase). The `prisma.config.ts` for that is in the `postgres-pro` skill, hosting section (Neon).

## Render

```yaml
# render.yaml (Blueprint) at the repo root
services:
  - type: web
    name: api
    runtime: node
    plan: starter              # free spins down after 15 min idle (~1 min cold start) and has no pre-deploy
    region: oregon             # same region as the database
    buildCommand: npm ci && npx prisma generate && npm run build
    preDeployCommand: npx prisma migrate deploy
    startCommand: node dist/main
    healthCheckPath: /health
    autoDeployTrigger: checksPass   # deploy only after CI passes (commit | checksPass | off)
    envVars:
      - key: NODE_VERSION
        value: "22"
      - key: DATABASE_URL
        sync: false            # prompted on first Blueprint sync (e.g. Neon pooled URL)
      - key: DIRECT_URL
        sync: false
      - key: JWT_SECRET
        generateValue: true
      - key: JWT_REFRESH_SECRET
        generateValue: true
```

- **Render Postgres instead of Neon:** add a `databases:` block and `fromDatabase: { name: <db>, property: connectionString }` on the variable. Check the `fromDatabase` keys against the Blueprint spec when you use it.
- **Free plan limits:** free Postgres expires after 30 days, then has a 14-day grace period before deletion. Free web services have no shell, no scaling and no pre-deploy. Fine for demos, not for anything with users.
- **Deploys:** zero-downtime. The old instance gets `SIGTERM` after the switch, with 30 s to finish requests (configurable up to 300 s). `enableShutdownHooks` makes Nest close cleanly within that window.
- **Previews:** `previews.generation: automatic` in `render.yaml` creates an environment per PR. Point it at a Neon branch, never at the production database.
- **Background jobs** (BullMQ consumers, crons): a separate `type: worker` service or a `type: cron` job, not `setInterval` in the web service, because web instances get replaced and scaled.

## Railway

Railway replaced Config as Code (`railway.json`/`railway.toml`) with Infrastructure as Code. Those files stop being read on **2026-12-01**, and new services can't use them. Migrate with `railway config migrate --apply`.

```typescript
// .railway/railway.ts   (npm i -D railway)
import { defineRailway, postgres, project, service } from 'railway/iac';

export default defineRailway(() => {
  const db = postgres('postgres'); // remove if the database lives on Neon
  const api = service('api', {
    build: 'npm ci && npx prisma generate && npm run build',
    start: 'node dist/main',
    healthcheck: '/health',
    env: {
      DATABASE_URL: db.env.DATABASE_URL, // reference: the value stays on Railway, not in git
    },
  });

  return project('my-app', { resources: [api, db] });
});
```

```bash
railway config plan     # preview changes
railway config apply    # apply (CI: --yes --confirm-destructive, only if you mean it)
```

- **Pre-deploy:** the IaC page doesn't document a pre-deploy key yet. Set the pre-deploy command (`npx prisma migrate deploy`) in the service settings, or use the start-command fallback above. Check the current docs before relying on either.
- **Builder:** Railpack is the default. A `Dockerfile` at the root always wins, so delete stray Dockerfiles or use them on purpose.
- **Private networking:** with Railway Postgres, use the internal `DATABASE_URL` (`*.railway.internal`) for the app: lower latency and no egress cost. The public URL is only for local access.
- **Secrets:** set them in the dashboard or with `railway variables`. Use references (`db.env.X`, `${{Postgres.DATABASE_URL}}`), never values, in versioned files.
- **Environments:** create a `staging` environment (PR environments are optional) with its own database or Neon branch.

## MongoDB (Mongoose) instead of Prisma

Same app requirements (shutdown hooks, `/health`, config validation); only the database steps change. Details in `references/mongodb.md`.

| Step | Prisma | Mongoose |
|------|--------|----------|
| Build | `npm ci && npx prisma generate && npm run build` | `npm ci && npm run build` |
| Pre-deploy | `npx prisma migrate deploy` | `npx migrate-mongo up && node dist/scripts/create-indexes.js` |
| Health | `PrismaHealthIndicator` | `MongooseHealthIndicator` (`pingCheck('mongodb')`) |

```ts
// src/scripts/create-indexes.ts — run before traffic switches; autoIndex is off in production
import type { Connection } from 'mongoose'; // type-only: see references/mongodb.md (ESM)
const app = await NestFactory.createApplicationContext(AppModule, { logger: ['error', 'warn'] });
const connection = app.get<Connection>(getConnectionToken());
for (const model of Object.values(connection.models)) {
  await model.createIndexes(); // creates missing indexes, drops nothing
}
await app.close();
```

- `createIndexes()` only adds. Don't use `syncIndexes()` in the deploy: it also **drops** indexes missing from the schema, which the old version still serving traffic may need. Drop old indexes in a later deploy, once no running code uses them (expand → contract, same as SQL migrations).
- Building an index on a big collection takes time and load. Create it before the deploy that needs it, outside peak hours, and check progress in Atlas.
- `migrate-mongo` keeps applied migrations in a `changelog` collection, so `up` is safe to run on every deploy. Its config reads `MONGODB_URI` from the environment (`migrate-mongo-config.js`); never hardcode the URI.
- **Hosting:** MongoDB Atlas, in the same cloud region as the Render/Railway service. Atlas IP access list: Render and Railway outbound IPs aren't fixed on every plan; check the platform's static outbound IP option before falling back to `0.0.0.0/0`, and if you do use it, rely on a strong password and TLS (on by default in Atlas). Railway also has a MongoDB template (single node, no replica set: no transactions).
- `MONGODB_URI` includes the database name (`...mongodb.net/myapp?retryWrites=true&w=majority`); without it, data goes to the `test` database.

## Checklist before the first production deploy

1. `CORS` origin list set to the real frontend domains (Vercel/Cloudflare), not `*`, when cookies are used.
2. Health check passes locally with `node dist/main` and production-like env vars.
3. Migrations (`prisma migrate deploy`, or `migrate-mongo up` + index creation) ran against a staging database first.
4. Secrets are only in the platform, `.env*` is gitignored, and JWT secrets are generated (not copied from `.env.example`).
5. Logs are structured JSON (NestJS 12 `ConsoleLogger({ json: true })` or pino) and contain no tokens, passwords or full request bodies.
6. Error monitoring (Sentry `@sentry/nestjs`) set up, with source maps uploaded during the build.
