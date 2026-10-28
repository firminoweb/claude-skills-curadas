# Performance Diagnosis

Adapted from [amirtaherkhani/nestjs-skills](https://github.com/amirtaherkhani/nestjs-skills) (MIT), `nestjs-features-performance`, rewritten for NestJS 12 + Prisma 7.

Never claim a speedup from how the code looks. Measure, change one thing, measure again with the same workload.

## Define the experiment

Before touching code, write down:

- The route/job and a realistic payload; data volume close to production.
- Concurrency and duration (e.g. `npx autocannon -c 50 -d 30 http://localhost:3000/orders`).
- p50/p95/p99 latency, requests/s, error rate.
- Event-loop delay, CPU, heap; database time and pool waits; upstream call time.
- The target ("p95 under 200 ms at 100 req/s"). "Make it faster" is not a target.

Tools that are already in Node or the stack:

| What | How |
|------|-----|
| Event-loop delay | `perf_hooks.monitorEventLoopDelay()`, log p99 periodically |
| CPU hotspots | `node --cpu-prof dist/main.js` under load, open the `.cpuprofile` in Chrome DevTools |
| Memory leak | `node --heapsnapshot-signal=SIGUSR2`, take two snapshots under load, compare retainers |
| Slow queries | Prisma `log: [{ emit: 'event', level: 'query' }]` + `$on('query', e => e.duration)`; then `EXPLAIN ANALYZE` (see the `postgres-pro` skill) |
| Where time goes per request | OpenTelemetry tracing (`@prisma/instrumentation` for Prisma spans) |

## Find the dominant constraint first

| Constraint | Evidence | First moves |
|------------|----------|-------------|
| Event loop blocked | High loop delay, CPU in your own callbacks | Remove sync work, bound input size, move CPU work to a worker/queue |
| Database | Query time dominates, pool waits, N+1 in the query log | Fix query shape and indexes, bound results, shorten transactions |
| External calls | Upstream spans dominate | Timeouts, run independent calls in parallel, cache, limit concurrency |
| Memory / GC | Heap grows under steady load, GC pauses, OOM restarts | Find the retainer, stream or page large data, bound caches and maps |
| Serialization / logging | CPU grows with response size | `select` fewer fields, paginate, log less on the hot path |
| Capacity | Each instance is healthy but all are saturated | Add instances (the app must be stateless) |

Don't switch to Fastify, add Redis, add worker threads or split services until the evidence points there.

## Event loop

Node runs your JavaScript on one thread: one slow synchronous callback delays every other request.

Look for: `*Sync` fs/crypto/zlib calls in handlers, `bcryptjs` (pure JS; `argon2` is native and async), huge `JSON.parse`/`stringify`, sorting or mapping unbounded arrays, regexes that can backtrack on user input, PDF/image generation in the request.

Fixes: bound input (`@MaxLength`, array size limits, body size limit), async APIs, move heavy work to a BullMQ job or a worker-thread pool (one pool, not one worker per request). Worker threads help CPU-bound JS; they do nothing for I/O.

## Database (Prisma)

- **N+1:** a `findMany` then a query per row. Use nested `select`/`include`, or collect ids and do one `findMany({ where: { id: { in: ids } } })`.
- **Unbounded results:** every list has `take`. Deep `skip` is slow; use cursor pagination (`cursor: { id }, skip: 1, take: 20`) for large tables.
- **Overfetching:** `select` only what the response needs; wide `include` trees cost database time and serialization CPU.
- **Missing index:** check the real `WHERE`/`ORDER BY` with `EXPLAIN ANALYZE` on production-like data; add `@@index` in the schema and a migration. Each index slows writes.
- **Pool:** size comes from `pg` (`max` in `PrismaPg` options). Pool waits with an idle database → pool too small; database at its connection limit → too many instances × `max` (use a pooler, see `postgres-pro`).
- **Transactions:** short, no HTTP calls inside. Long transactions hold locks and pool connections.
- **Counts on hot paths:** `count()` on big tables per request adds up; cache it or drop the total.

## External calls

- Timeout on every call (`references/errors-resilience.md`).
- Independent calls: `Promise.all`, but bound concurrency for fan-out over a list (don't fire 1,000 requests at once).
- Reuse connections: one client per upstream as a provider, not a new client per request.

## Provider scope

Singletons are the default and the fast path. `Scope.REQUEST` creates the provider **and every consumer up the chain** per request. Use it only for a real per-request lifetime and measure; pass the user or tenant as an argument instead. Never store request data in a singleton field.

## Caching

Cache only repeated expensive work with acceptable staleness, after measuring.

- `@nestjs/cache-manager` (Redis store when there's more than one instance).
- The key includes everything that changes the result: user/tenant, permissions, locale, query params.
- Define TTL and invalidation on write. Add jitter to TTLs so keys don't expire together.
- Measure hit ratio. A cache with a low hit ratio only adds latency and bugs.

## Fastify

It can raise HTTP-layer throughput, but in a typical API the database dominates. Switch only after profiling shows adapter overhead matters, and after checking every Express-specific middleware and `@Res()` use. Generic benchmarks are not evidence for this app.

## Report format

```text
Baseline:    workload + p95/p99 + req/s + errors
Bottleneck:  evidence (profile, query log, span) and the dominant resource
Change:      one intervention
Result:      same metrics, same workload
Correctness: tests still pass, failure behavior unchanged
Next limit:  what saturates next
```
