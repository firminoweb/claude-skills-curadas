---
name: nestjs-expert
description: Use when building or reviewing a NestJS backend (REST or GraphQL) — modules, controllers, services, providers, DTOs, validation, guards, interceptors, JWT auth, Swagger, Jest/Vitest tests, deploying to Render or Railway — especially with Prisma as the ORM, when touching .module.ts/.controller.ts/.service.ts or schema.prisma files, upgrading to NestJS 12 or Prisma 7, or migrating an Express app to NestJS.
license: MIT
metadata:
  author: https://github.com/Jeffallan (updated by curator)
  version: "2.0.0"
  baseline: "NestJS 12.2, Prisma ORM 7.10, Node 22 LTS (Oct 2026)"
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
3. **Prisma major:** `schema.prisma` + `prisma.config.ts`/`prisma7.config.ts` = Prisma 7 (this skill). A `prisma8/contract.prisma` or `@prisma/orm-postgres` = Prisma 8, a different client API: don't apply Prisma 7 code, read its docs. `schema.prisma` with `provider = "prisma-client-js"` and no adapter = Prisma 6 or older.
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

## Rules

- Constructor injection everywhere; never `new` a provider. Circular imports mean the module boundaries are wrong; `forwardRef()` is a last resort.
- Every input crosses a validation pipe: global `ValidationPipe({ whitelist: true, forbidNonWhitelisted: true, transform: true })` for class DTOs, global `StandardSchemaValidationPipe` for schemas. Params too (`ParseUUIDPipe` or a schema).
- Services throw Nest HTTP exceptions (`NotFoundException`, `ConflictException`); don't build responses by hand.
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
| Authentication | `references/authentication.md` | JWT, Passport, guards, roles, refresh tokens |
| Testing | `references/testing-patterns.md` | Unit tests with a mocked Prisma, E2E with a real database |
| Express migration | `references/migration-from-express.md` | Moving an Express app to NestJS |
| Deploy | `references/deploy-render-railway.md` | Render (`render.yaml`), Railway (`.railway/railway.ts`), health check, migrations on deploy |

## Verify before claiming done

1. `npx prisma validate` and `npx prisma generate` succeed; a new migration exists for every schema change (`prisma migrate dev --name <change>`), and you read its SQL.
2. `tsc --noEmit` (or `nest build`) and the lint script are clean.
3. Unit tests for services and an E2E test for each new route (happy path, 400, 401/403, 404) pass.
4. Responses contain no secret fields (check the E2E body).
