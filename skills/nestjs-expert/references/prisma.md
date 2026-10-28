# Prisma ORM 7 with NestJS

Prisma 8 is a release candidate with a different schema and client API. Everything below is Prisma 7.

## Install (PostgreSQL)

```bash
npm i -D prisma@7 dotenv
npm i @prisma/client@7 @prisma/adapter-pg @nestjs/config
npx prisma init --datasource-provider postgresql --output ../src/generated/prisma
```

- Pin `@7` on both: `prisma`'s npm `latest` tag is the Prisma 8 RC. Keep `prisma` and `@prisma/client` on the same version.
- Other databases: `@prisma/adapter-mariadb` (MySQL/MariaDB), `@prisma/adapter-mssql`, `@prisma/adapter-better-sqlite3`. MongoDB isn't supported in Prisma 7 (stay on 6).
- `init` adds `.env` and the generated folder to `.gitignore`.

## Config and schema

`prisma init` on 7.10+ writes `prisma7.config.ts` (so it can coexist with Prisma 8); older projects have `prisma.config.ts`. Prisma 7 loads `prisma7.config.*` first, so edit the file the CLI reports loading.

```ts
// prisma.config.ts (or prisma7.config.ts)
import 'dotenv/config'; // Prisma 7 doesn't load .env by itself
import { defineConfig, env } from 'prisma/config';

export default defineConfig({
  schema: 'prisma/schema.prisma',
  migrations: { path: 'prisma/migrations', seed: 'tsx prisma/seed.ts' },
  datasource: { url: env('DATABASE_URL') },
});
```

```prisma
// prisma/schema.prisma
generator client {
  provider     = "prisma-client"
  output       = "../src/generated/prisma"
  moduleFormat = "cjs" // only for CommonJS Nest projects; omit for ESM
}

datasource db {
  provider = "postgresql" // url lives in the config file
}

model User {
  id           String   @id @default(uuid(7))
  email        String   @unique
  name         String
  passwordHash String
  role         Role     @default(USER)
  posts        Post[]
  createdAt    DateTime @default(now())
  updatedAt    DateTime @updatedAt
}

model Post {
  id        String   @id @default(uuid(7))
  title     String
  authorId  String
  author    User     @relation(fields: [authorId], references: [id], onDelete: Cascade)
  createdAt DateTime @default(now())

  @@index([authorId, createdAt])
}

enum Role {
  USER
  ADMIN
}
```

- Index every foreign key you filter by (Prisma doesn't create them on PostgreSQL).
- `uuid(7)` gives time-ordered IDs (better index locality than v4).

## PrismaService and PrismaModule

```ts
// src/prisma/prisma.service.ts
import { Injectable, OnModuleDestroy, OnModuleInit } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { PrismaPg } from '@prisma/adapter-pg';
import { PrismaClient } from '../generated/prisma/client';

@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit, OnModuleDestroy {
  constructor(config: ConfigService) {
    super({ adapter: new PrismaPg({ connectionString: config.getOrThrow<string>('DATABASE_URL') }) });
  }

  async onModuleInit() {
    await this.$connect(); // fail at boot, not on the first request
  }

  async onModuleDestroy() {
    await this.$disconnect();
  }
}

// src/prisma/prisma.module.ts
@Global() // most modules need it; drop @Global and import it explicitly if you prefer
@Module({ providers: [PrismaService], exports: [PrismaService] })
export class PrismaModule {}
```

- `main.ts`: `app.enableShutdownHooks()` so `onModuleDestroy` runs on SIGTERM. `beforeExit` hooks don't exist in Prisma 7.
- `ConfigModule.forRoot({ isGlobal: true })` in `AppModule`; Nest doesn't read `.env` on its own.
- ESM projects: add `.js` to these relative imports (`'../generated/prisma/client.js'`).
- The `pg` driver validates TLS certificates. Error P1010 on a managed database usually means a missing CA, not a wrong password. Configure the CA; don't set `rejectUnauthorized: false` in production.
- Pool size and timeouts now come from `pg` (`max`, `connectionTimeoutMillis` in the adapter options), not from URL parameters.

## Migrations workflow

| Command | When |
|---------|------|
| `npx prisma migrate dev --name add_posts` | Local: creates and applies a migration. Read the generated SQL before committing |
| `npx prisma generate` | After every schema change, on fresh clones, and in CI before `nest build` (`migrate dev` no longer runs it) |
| `npx prisma db seed` | Seeding is no longer automatic after `migrate dev`/`reset` |
| `npx prisma migrate deploy` | Production/CI: applies pending migrations, never generates new ones |
| `npx prisma migrate reset` | Local only: drops the database |

- Never run `migrate dev` or `db push` against production.
- Destructive changes (drop/rename a column) on a live app: expand → deploy code that uses both → backfill → contract, in separate migrations. Prisma reads a rename as drop + add, so edit the SQL to `ALTER ... RENAME`.
- Commit `prisma/migrations/` and `migration_lock.toml`.

## Queries

```ts
// Ownership in the where clause: another user's ID returns 404, not their data
const post = await this.prisma.post.findFirst({ where: { id, authorId: user.id } });
if (!post) throw new NotFoundException();

// Cursor pagination (stable under inserts; prefer over skip for large tables)
const page = await this.prisma.post.findMany({
  where: { authorId },
  orderBy: [{ createdAt: 'desc' }, { id: 'desc' }],
  take: limit + 1,
  ...(cursor && { cursor: { id: cursor }, skip: 1 }),
  select: { id: true, title: true, createdAt: true },
});
const nextCursor = page.length > limit ? page.pop()!.id : null;

// Avoid N+1: load relations in one query
const users = await this.prisma.user.findMany({ include: { posts: { take: 5 } } });
```

- `select` only what the response needs. Use `omit: { passwordHash: true }` when you want everything except secrets (or set it globally: `new PrismaClient({ adapter, omit: { user: { passwordHash: true } } })`).
- `findUnique` only works on unique fields; for `id + ownerId` use `findFirst` or a compound unique.
- Raw SQL: `$queryRaw` tagged template (parameterized). **Never** `$queryRawUnsafe` with user input.

## Transactions

```ts
// Interactive: reads and writes that must succeed together
await this.prisma.$transaction(async (tx) => {
  const from = await tx.account.update({ where: { id: fromId }, data: { balance: { decrement: amount } } });
  if (from.balance < 0) throw new BadRequestException('Insufficient funds'); // rolls back
  await tx.account.update({ where: { id: toId }, data: { balance: { increment: amount } } });
});
// Send emails/webhooks after this line, not inside the callback
```

- Use the array form (`$transaction([a, b])`) for independent writes, and the interactive form when one step depends on another.
- Keep transactions short: no HTTP calls inside (timeout defaults to 5 s).
- Money: `Decimal` columns, never `Float`.

## Errors: one global filter

```ts
// src/prisma/prisma-exception.filter.ts
import { ArgumentsHost, Catch, ConflictException, NotFoundException } from '@nestjs/common';
import { BaseExceptionFilter } from '@nestjs/core';
import { Prisma } from '../generated/prisma/client';

@Catch(Prisma.PrismaClientKnownRequestError)
export class PrismaExceptionFilter extends BaseExceptionFilter {
  catch(e: Prisma.PrismaClientKnownRequestError, host: ArgumentsHost) {
    if (e.code === 'P2002') return super.catch(new ConflictException('Already exists'), host);
    if (e.code === 'P2025') return super.catch(new NotFoundException(), host);
    return super.catch(e, host); // anything else stays a 500, without leaking the message
  }
}

// main.ts
const { httpAdapter } = app.get(HttpAdapterHost);
app.useGlobalFilters(new PrismaExceptionFilter(httpAdapter));
```

| Code | Meaning | HTTP |
|------|---------|------|
| P2002 | Unique constraint | 409 |
| P2003 | Foreign key constraint | 409 or 400 |
| P2025 | Record not found (update/delete/`findUniqueOrThrow`) | 404 |

The filter is the fallback; a service can still catch P2002 itself when it needs a specific message.

## Client extensions (replace `$use`)

```ts
// Soft delete: hide deleted rows from reads
const extended = prisma.$extends({
  query: {
    post: {
      async findMany({ args, query }) {
        args.where = { ...args.where, deletedAt: null };
        return query(args);
      },
    },
  },
});
```

Expose an extended client through its own provider (`{ provide: EXTENDED_PRISMA, useFactory: (p: PrismaService) => p.$extends(...), inject: [PrismaService] }`), since `$extends` returns a new client and `PrismaService` can't extend its own result.
