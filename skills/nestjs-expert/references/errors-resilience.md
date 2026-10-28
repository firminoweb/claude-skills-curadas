# Errors and Resilience

Adapted from [amirtaherkhani/nestjs-skills](https://github.com/amirtaherkhani/nestjs-skills) (MIT), `nestjs-features-performance`, rewritten for NestJS 12 + Prisma 7.

## Classify before mapping

| Category | Example | Status | Retry? |
|----------|---------|--------|--------|
| Invalid input | DTO/schema rejected | 400 (or 422, pick one per API) | No |
| Not authenticated / not allowed | Missing token / other user's resource | 401 / 403 (404 to hide existence) | No |
| Not found | `findUnique` returned null, P2025 | 404 | No |
| Conflict | P2002, stale version, state doesn't allow the action | 409 | No (except P2034, below) |
| Rate limited | Throttler, upstream 429 | 429 + `Retry-After` | After the hint |
| Transient dependency | Connection reset, upstream 503, P1001 | 503 (or 502/504 as a gateway) | Yes, if idempotent |
| Timeout | Upstream didn't answer in time | 504 / 503 | **Outcome unknown**: reconcile first |
| Bug / unknown | Anything unclassified | 500, generic message | No |

Classify by type or driver code (`e.code === 'P2002'`), never by parsing `e.message`.

## Where the mapping lives

- **HTTP-only feature:** throwing `NotFoundException`/`ConflictException` from the service is fine (the default in `SKILL.md`).
- **Logic shared with a queue worker, cron, microservice or WebSocket gateway:** HTTP exceptions mean nothing there. Throw an application error and map it once per transport.

```ts
// common/app-error.ts
export type ErrorKind = 'invalid' | 'forbidden' | 'not_found' | 'conflict' | 'unavailable';

export class AppError extends Error {
  constructor(
    readonly kind: ErrorKind,
    readonly code: string, // stable, machine-readable: 'ORDER_NOT_PAYABLE'
    message: string,
    options?: ErrorOptions, // { cause } keeps the original error for logs
  ) {
    super(message, options);
  }
}
```

```ts
// common/app-error.filter.ts
import { ArgumentsHost, Catch, ExceptionFilter } from '@nestjs/common';
import { HttpAdapterHost } from '@nestjs/core';
import { AppError, ErrorKind } from './app-error';

const STATUS: Record<ErrorKind, number> = {
  invalid: 400, forbidden: 403, not_found: 404, conflict: 409, unavailable: 503,
};

@Catch(AppError)
export class AppErrorFilter implements ExceptionFilter {
  constructor(private readonly adapterHost: HttpAdapterHost) {}

  catch(e: AppError, host: ArgumentsHost) {
    const status = STATUS[e.kind];
    const { httpAdapter } = this.adapterHost; // works on Express and Fastify
    httpAdapter.reply(host.switchToHttp().getResponse(), { statusCode: status, code: e.code, message: e.message }, status);
  }
}

// app.module.ts — APP_FILTER keeps it inside DI
providers: [{ provide: APP_FILTER, useClass: AppErrorFilter }]
```

A queue processor catches the same `AppError` and decides: `invalid`/`not_found`/`conflict` → fail the job without retry (BullMQ: throw `UnrecoverableError`); `unavailable` → rethrow so the queue retries with backoff.

## Public error contract

- Add a stable `code` next to Nest's `statusCode`/`message`. Clients branch on `code`; the message can change wording.
- Never expose stack traces, SQL, Prisma messages, hostnames, tokens or raw upstream bodies. The Prisma filter in `references/prisma.md` already keeps unknown Prisma errors as a generic 500.
- Validation errors list field paths and reasons, nothing internal.
- Once a project picks an envelope (Nest default + `code`, or RFC 9457 `application/problem+json`), keep it. Don't switch formats for aesthetics: codes and shapes are a public contract.
- Log an unexpected error **once**, at the boundary, with a request id. Don't log and rethrow at every layer.

## Deadlines

Every outbound call has a timeout. A request without one holds a connection and an event-loop slot forever.

```ts
// External HTTP: fetch + AbortSignal (Node 22)
const res = await fetch(url, { signal: AbortSignal.timeout(3_000) });

// Combine with a caller's signal when you have one
const signal = AbortSignal.any([callerSignal, AbortSignal.timeout(3_000)]);
```

```ts
// Database: statement timeout on the pool, transaction limits on the call
new PrismaPg({ connectionString, statement_timeout: 5_000, connectionTimeoutMillis: 5_000 });
await this.prisma.$transaction(async (tx) => { /* ... */ }, { timeout: 5_000, maxWait: 2_000 });
```

- The sum of dependency timeouts must fit inside the platform's request limit and the client's patience.
- A route-level `timeout()` interceptor (RxJS) only stops **waiting**; the work keeps running. Use it as a last guard, not as the deadline mechanism.
- A timeout does **not** prove a write failed. Before repeating a write, check its outcome or use an idempotency key.

## Retries

Retry only when **all** are true: the failure is transient, the operation is idempotent (or carries an idempotency key), and the total time stays inside the deadline.

```ts
// common/retry.ts
export async function withRetry<T>(
  fn: () => Promise<T>,
  isTransient: (e: unknown) => boolean,
  { attempts = 3, baseMs = 200 } = {},
): Promise<T> {
  for (let i = 1; ; i++) {
    try {
      return await fn();
    } catch (e) {
      if (i >= attempts || !isTransient(e)) throw e;
      const delay = baseMs * 2 ** (i - 1) * (0.5 + Math.random()); // exponential + jitter
      await new Promise((r) => setTimeout(r, delay));
    }
  }
}
```

- Prisma transient codes: `P1001` (can't reach database), `P2034` (write conflict/deadlock: retry the **whole** `$transaction`, never one statement).
- Never retry 400/401/403/404/409, validation or business-rule errors.
- Retry at one layer only. SDK retries + service retries + queue retries multiply into an outage.
- Circuit breakers and fallbacks are not defaults; add one only for a named dependency with a defined degraded behavior.

## Idempotency for writes that matter

Payments, orders, anything a client may resend after a timeout: accept an `Idempotency-Key` header and let a unique constraint do the work.

```prisma
model IdempotencyKey {
  key       String   @id
  userId    String
  response  Json?
  createdAt DateTime @default(now())
}
```

```ts
// Inside the same $transaction as the write:
// 1. tx.idempotencyKey.create({ data: { key, userId } })
//    → P2002 means seen before: return the stored response (or 409 if response is still null, i.e. in progress)
// 2. do the write
// 3. tx.idempotencyKey.update({ where: { key }, data: { response } })
```

Scope the key to the user (`userId` check on lookup) and expire old rows with a scheduled job.

## Side effects outside the transaction

- Emails, webhooks and payment calls go **after** the commit (rule in `SKILL.md`).
- If losing that side effect is not acceptable, write an "outbox" row in the same transaction and let a worker deliver it, retrying until done. The worker must tolerate duplicates.
- No floating promises in request handlers (`void this.mail.send(...)`): a failure vanishes and shutdown cuts it off. Await it, or enqueue it (`@nestjs/bullmq`). Enable the `no-floating-promises` lint rule.

## Fatal errors and shutdown

- `uncaughtException` / `unhandledRejection`: log once and **exit**. State may be corrupt; the platform (Render, Railway) restarts a clean instance. Never keep serving.
- Graceful shutdown (`SIGTERM` on deploy) is different: `app.enableShutdownHooks()` → stop taking requests → finish in-flight work → `onModuleDestroy` closes Prisma and queue workers. It must finish inside the platform window (`references/deploy-render-railway.md`).

## Tests for failure paths

| Boundary | Minimum |
|----------|---------|
| Service | Each `AppError`/HTTP exception it can throw, and that nothing was written on failure |
| Prisma mapping | P2002 → 409, P2025 → 404, unknown → 500 with no Prisma text in the body |
| E2E | Body has `statusCode` + `code`, no stack/SQL/secrets |
| External call | Timeout, 5xx, malformed response; retry stops at the limit |
| Idempotent write | Same key twice → one row, same response |
| Worker | Job fails permanently on `invalid`, retries on `unavailable`, survives duplicates |
