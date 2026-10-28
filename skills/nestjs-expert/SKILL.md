---
name: nestjs-expert
description: Use when building or reviewing a NestJS backend (REST or GraphQL) — modules, controllers, services, providers, DTOs, validation, guards, interceptors, JWT auth, Google/social login, Swagger, GraphQL resolvers and DataLoader, error contracts, timeouts/retries/idempotency, slow routes and performance diagnosis, module architecture, Jest/Vitest tests, deploying to Render or Railway — especially with Prisma as the ORM (or MongoDB with Mongoose), BullMQ background jobs and scheduled jobs, Kafka producers/consumers and event-driven messaging, when touching .module.ts/.controller.ts/.service.ts or schema.prisma files, upgrading to NestJS 12 or Prisma 7, or migrating an Express app to NestJS.
license: MIT
metadata:
  author: https://github.com/Jeffallan (updated by curator)
  version: "2.5.0"
  baseline: "NestJS 12.3, Prisma ORM 7.10, Mongoose 9.11, BullMQ 6.5, @nestjs/graphql 14 + Apollo Server 5, kafkajs 2.2.4, Node 22 LTS (Oct 2026)"
  domain: backend
  triggers: NestJS, Nest, Prisma, Render, Railway, render.yaml, schema.prisma, PrismaService, dependency injection, controller, service, module, guard, interceptor, DTO
  related-skills: postgres-pro, secure-code-guardian
  role: specialist
  scope: implementation
  output-format: code
---

# NestJS Expert (NestJS 12 + Prisma 7)

## First: read the project

1. **Versions** in `package.json`: `@nestjs/core`, `prisma`, `@prisma/client`. Don't use an API newer than what's installed. NestJS 11 projects: everything here applies except Standard Schema pipes, `errorFormat` and `nest upgrade`.
2. **Module system:** `"type": "module"` in `package.json` means ESM. In that case relative imports need `.js` extensions (`'./users.service.js'`), and new projects use Vitest + oxlint. CommonJS projects keep Jest. Match what exists.
3. **Database layer:** `@nestjs/mongoose` in `package.json` = MongoDB with Mongoose: follow `references/mongodb.md` instead of the Prisma examples (the rules below still apply). **Prisma major:** `schema.prisma` + `prisma.config.ts`/`prisma7.config.ts` = Prisma 7 (this skill). A `prisma8/contract.prisma` or `@prisma/orm-postgres` = Prisma 8, a different client API: don't apply Prisma 7 code, read its docs. `schema.prisma` with `provider = "prisma-client-js"` and no adapter = Prisma 6 or older.
4. **Validation style:** `class-validator` DTOs or Standard Schema (Zod/Valibot) on `@Body({ schema })`. Use the one already in the project; don't mix in one module.
5. **Where the generated client lives:** the `output` of the `generator` block (usually `src/generated/prisma`). It's gitignored and must be regenerated (`npx prisma generate`) after every schema change and in CI.

## Outdated habits

| Outdated | Current |
|----------|---------|
| `npm i prisma` (unpinned) | `prisma@7` + `@prisma/client@7`: the npm `latest` tag of `prisma` points to the Prisma 8 RC |
| `provider = "prisma-client-js"`, import from `@prisma/client` | `provider = "prisma-client"` with `output`, import from the generated `client` path |
| `new PrismaClient()` with `url` in the schema | Driver adapter: `new PrismaClient({ adapter: new PrismaPg({ connectionString }) })`, URL in `prisma.config.ts` |
| Expecting `migrate dev` to run `generate`/seed | Run `prisma generate` and `prisma db seed` explicitly |
| `prisma.$use()` middleware | Client extensions (`$extends`) |
| `this.$on('beforeExit')` / `enableShutdownHooks` workaround in PrismaService | `onModuleDestroy() { await this.$disconnect() }` + `app.enableShutdownHooks()` |
| TypeORM entities + `@InjectRepository` | `PrismaService` injected in services; Prisma models are the types |
| Hand-written Zod pipe per route | `@Body({ schema })` + global `StandardSchemaValidationPipe` (NestJS 12) |
| `--webpack` in `nest-cli.json` | `--builder rspack` (webpack workflows deprecated in 12) |
| `import * as request from 'supertest'` | `import request from 'supertest'` |
| Manually bumping `@nestjs/*` | `nest upgrade --dry-run`, then `nest upgrade` |
| Route wildcards `'*'` | Express 5 syntax: `'*splat'` / `'{*splat}'` |

## Core example: feature module with Prisma

```typescript
// users/users.service.ts
import { ConflictException, Injectable, NotFoundException } from '@nestjs/common';
import { PrismaService } from '../prisma/prisma.service';
import { Prisma } from '../generated/prisma/client';
import { CreateUserDto } from './dto/create-user.dto';
import { hash } from 'argon2';

// Never return passwordHash: select the public fields once and reuse them
const publicUser = { id: true, email: true, name: true, createdAt: true } satisfies Prisma.UserSelect;

@Injectable()
export class UsersService {
  constructor(private readonly prisma: PrismaService) {}

  async create(dto: CreateUserDto) {
    try {
      return await this.prisma.user.create({
        data: { email: dto.email, name: dto.name, passwordHash: await hash(dto.password) },
        select: publicUser,
      });
    } catch (e) {
      if (e instanceof Prisma.PrismaClientKnownRequestError && e.code === 'P2002') {
        throw new ConflictException('Email already registered');
      }
      throw e;
    }
  }

  async findOne(id: string) {
    const user = await this.prisma.user.findUnique({ where: { id }, select: publicUser });
    if (!user) throw new NotFoundException(`User ${id} not found`);
    return user;
  }
}
```

```typescript
// users/users.module.ts
@Module({
  imports: [PrismaModule], // omit if PrismaModule is @Global()
  controllers: [UsersController],
  providers: [UsersService],
  exports: [UsersService], // only if another module needs it
})
export class UsersModule {}
```

`PrismaService`, schema, migrations, transactions and the global P2002/P2025 filter: `references/prisma.md`.

## Where each concern goes

```text
request → middleware → guards → interceptors (before) → pipes → controller → service
        ← interceptors (after, reverse order) ← exception filters (only uncaught errors) ← response
```

| Piece | Use for | Not for |
|-------|---------|---------|
| Middleware | Raw request prep, request id, helmet | Authorization, business validation |
| Guard | Authentication and coarse authorization (roles) | Ownership checks that need the row (do those in the query) |
| Pipe | Validate/transform one input | Database calls |
| Interceptor | Timing, tracing, response mapping | Hidden business policy |
| Exception filter | Map uncaught errors to a response | Swallowing errors, retrying |
| Service | Business logic, Prisma, orchestration | Building HTTP responses |
| Queue (`@nestjs/bullmq`) | Slow, retryable or bursty work (email, PDF, webhooks) | Work whose result the request needs now |

Bind at the narrowest level that fits (param → route → controller → global). Global enhancers that need DI go through `APP_GUARD`/`APP_PIPE`/`APP_INTERCEPTOR`/`APP_FILTER`. `@Cron` runs on **every** instance: with more than one replica, use a BullMQ job scheduler (`references/bullmq.md`).

## Architecture: stay on the lowest level that works

| Level | Shape | Move up when |
|-------|-------|--------------|
| 1. Feature modules (default) | `orders/` with controller, service, DTOs, module; Prisma in the service | Business rules can't be tested without Prisma, or one feature has several entry points (HTTP + queue + cron) |
| 2. Layered feature | `domain/` (no Nest, no Prisma imports), `application/` with ports, `infrastructure/` adapters, `http/` | A vendor changes often (payments, email): put a port in front of it, only there |
| 3. CQRS / hexagonal | Command and query handlers, separate read models | Reads need different models/stores than writes, or auditability is a requirement. Not for plain CRUD |
| 4. Separate services | Independent deploys | Independent team, scaling, compliance zone or failure isolation, **with** its own data. Shared tables = distributed monolith |

Apply a level per feature, not to the whole app. A circular import is a module-ownership problem; fix the boundary, don't split into services. Organize by business capability (`orders/`, `billing/`), not by technical type (`controllers/`, `services/`).

## Rules

- Constructor injection everywhere; never `new` a provider. Circular imports mean the module boundaries are wrong; `forwardRef()` is a last resort.
- Every input crosses a validation pipe: global `ValidationPipe({ whitelist: true, forbidNonWhitelisted: true, transform: true })` for class DTOs, global `StandardSchemaValidationPipe` for schemas. Params too (`ParseUUIDPipe` or a schema).
- Services throw Nest HTTP exceptions (`NotFoundException`, `ConflictException`); don't build responses by hand. When the same logic also runs from a queue, cron or another transport, throw an application error and map it per transport (`references/errors-resilience.md`).
- Every outbound call (HTTP, database, SDK) has a timeout. Retry only transient failures of idempotent operations, with backoff and jitter, at one layer.
- **Never return Prisma models directly** when they hold secrets (password hashes, tokens). Use `select`/`omit` or a response serializer.
- **Authorization in the query**, not after it: `where: { id, ownerId: user.id }` so a user can't read another user's row by ID.
- Multi-step writes go in `prisma.$transaction`. External calls (email, payments) go after the commit, not inside it.
- Config through `ConfigModule.forRoot({ isGlobal: true, validationSchema })` and `ConfigService.getOrThrow()`; the app fails at boot if a variable is missing. Nothing hardcoded.
- Rate-limit auth routes (`@nestjs/throttler`), enable `helmet` and an explicit CORS origin list.
- Swagger: CLI plugin (`@nestjs/swagger` in `nest-cli.json` plugins) for class DTOs, instead of `@ApiProperty()` on every field.
- No `any`. Prisma generates the types: `Prisma.UserCreateInput`, `Prisma.UserGetPayload<{ select: typeof publicUser }>`.

## References

| Topic | File | Load when |
|-------|------|-----------|
| Prisma | `references/prisma.md` | Setup, PrismaService, schema, migrations, transactions, errors, pagination, extensions |
| Controllers | `references/controllers-routing.md` | Routes, params, versioning, Swagger |
| Services and DI | `references/services-di.md` | Providers, tokens, scopes, lifecycle |
| DTOs and validation | `references/dtos-validation.md` | class-validator, Standard Schema (Zod), transforms |
| Authentication | `references/authentication.md` | JWT, Passport, guards, roles, refresh tokens, Google login (ID token or redirect flow), linking accounts |
| MongoDB | `references/mongodb.md` | Mongoose 9 schemas, services, errors (11000), indexes, transactions, data migrations |
| Background jobs | `references/bullmq.md` | BullMQ queues and workers, retries, `UnrecoverableError`, scheduled jobs across replicas, rate limits, separate worker process |
| Kafka | `references/kafka.md` | Producing/consuming events, delivery semantics, dead-letter topic, idempotent consumers, outbox, event contracts |
| GraphQL | `references/graphql.md` | Code-first schema, resolvers, guards in GraphQL, DataLoader, depth/complexity limits, GraphQL errors, subscriptions |
| SOLID and smells | `references/solid.md` | Refactoring a god service, ports for vendors, strategies, deciding whether an abstraction is worth it |
| Testing | `references/testing-patterns.md` | Unit tests with a mocked Prisma, E2E with a real database |
| Errors and resilience | `references/errors-resilience.md` | Error codes and contract, errors shared with queues, timeouts, retries, idempotency keys, outbox, fatal errors |
| Performance | `references/performance.md` | Slow route, high latency/CPU/memory, N+1, pool waits, caching, Fastify question |
| Express migration | `references/migration-from-express.md` | Moving an Express app to NestJS |
| Deploy | `references/deploy-render-railway.md` | Render (`render.yaml`), Railway (`.railway/railway.ts`), health check, migrations on deploy (Prisma or MongoDB) |

## Verify before claiming done

1. Data layer:
   - **Prisma:** `npx prisma validate` and `npx prisma generate` succeed; a new migration exists for every schema change (`prisma migrate dev --name <change>`), and you read its SQL.
   - **Mongoose:** every new filter/sort used in a hot path has an index in the schema (`.explain('executionStats')` shows no `COLLSCAN`); renamed or reshaped fields have a `migrate-mongo` script that works while old and new code run side by side; secret fields have `select: false`.
2. `tsc --noEmit` (or `nest build`) and the lint script are clean.
3. Unit tests for services and an E2E test for each new route (happy path, 400, 401/403, 404) pass, against a real database (Postgres, or `mongodb-memory-server`/Testcontainers).
4. Responses contain no secret fields (check the E2E body).
5. Jobs and event consumers (if touched): running the same job/message twice has one effect, and a permanent failure doesn't retry forever.
