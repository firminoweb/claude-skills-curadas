# Background Jobs with BullMQ

Baseline: `@nestjs/bullmq` 12, `bullmq` 6 (released Jul 2026), `ioredis`, Oct 2026. For events shared between services, see `references/kafka.md` instead.

## Use a job when

- The request doesn't need the result now: emails, push notifications, PDFs, image processing, webhooks out, exports.
- The work must survive a crash or a deploy and be retried (a floating promise in a handler is lost).
- Something must run on a schedule exactly once across all replicas (`@Cron` runs on every instance).
- An upstream has a rate limit you must respect.

## Install and connect

```bash
npm i @nestjs/bullmq@12 bullmq@6 ioredis
```

BullMQ 6 no longer brings `ioredis` with it; install it explicitly.

```ts
// app.module.ts
BullModule.forRootAsync({
  inject: [ConfigService],
  useFactory: (config: ConfigService) => ({
    connection: redisOptions(config.getOrThrow<string>('REDIS_URL')),
    defaultJobOptions: {
      attempts: 5,
      backoff: { type: 'exponential', delay: 2_000, jitter: 0.5 },
      removeOnComplete: { age: 24 * 3600, count: 1_000 }, // otherwise Redis fills up with finished jobs
      removeOnFail: { age: 7 * 24 * 3600 },               // keep failures long enough to inspect
    },
  }),
}),
BullModule.registerQueue({ name: 'email' }),
```

```ts
// Options, not a `new IORedis()` instance: BullMQ then owns the connections and closes them on
// shutdown. A shared instance stays open after `app.close()` (tests hang, Jest reports open handles).
function redisOptions(url: string) {
  const u = new URL(url); // redis:// or rediss:// (TLS)
  return {
    host: u.hostname,
    port: Number(u.port || 6379),
    username: u.username || undefined,
    password: u.password ? decodeURIComponent(u.password) : undefined,
    tls: u.protocol === 'rediss:' ? {} : undefined,
    maxRetriesPerRequest: null, // required by workers
  };
}
```

Redis must run with `maxmemory-policy noeviction`. With any eviction policy, Redis silently deletes job keys under memory pressure. Check it on the hosting dashboard (Render Key Value, Railway Redis). Serverless Redis billed per command gets expensive, because idle workers keep issuing blocking commands; prefer a regular instance.

## Producer

```ts
@Injectable()
export class SignupService {
  constructor(@InjectQueue('email') private readonly emailQueue: Queue<WelcomeEmailJob>) {}

  async signup(dto: SignupDto) {
    const user = await this.users.create(dto);
    await this.emailQueue.add('welcome', { userId: user.id }, { jobId: `welcome-${user.id}` });
    return user;
  }
}
```

- Job data is **small and has ids**, not whole objects: the job reads fresh data when it runs. No secrets or tokens: Redis stores the data in plain text, and dashboards show it.
- `jobId` makes the add idempotent: a second add with the same id is ignored while the job exists. Custom ids can't be plain integers or contain `:`.
- `deduplication: { id, ttl }` collapses repeated adds within a window (e.g. "reindex product X" fired 10 times in a second).
- Adding after a database write has the same gap as Kafka: if the process dies in between, the job is lost. When it matters, write an outbox row in the transaction and let a scheduled job enqueue pending rows (`references/errors-resilience.md`).

## Worker

```ts
@Processor('email', { concurrency: 5 })
export class EmailProcessor extends WorkerHost {
  private readonly logger = new Logger(EmailProcessor.name);

  constructor(private readonly users: UsersService, private readonly mailer: MailerService) {
    super();
  }

  async process(job: Job<WelcomeEmailJob>) {
    switch (job.name) {
      case 'welcome': {
        const user = await this.users.findById(job.data.userId);
        if (!user) throw new UnrecoverableError(`User ${job.data.userId} gone`); // no retries
        await this.mailer.send({ to: user.email, template: 'welcome', idempotencyKey: job.id });
        return;
      }
      default:
        throw new UnrecoverableError(`Unknown job ${job.name}`);
    }
  }

  @OnWorkerEvent('failed')
  onFailed(job: Job | undefined, err: Error) {
    if (job && job.attemptsMade >= (job.opts.attempts ?? 1)) {
      this.logger.error(`Job ${job.name}:${job.id} failed for good: ${err.message}`); // alert here
    }
  }
}
```

Register `EmailProcessor` as a provider in a module that imports `BullModule.registerQueue({ name: 'email' })`.

## Delivery semantics

- **At least once.** If a worker dies mid-job, the lock expires and the job runs again ("stalled"). After `maxStalledCount` (default 1) stalls it fails. Every job must be safe to run twice: pass `job.id` as the idempotency key to upstreams, check state before writing.
- **Errors decide retries.** A thrown error retries with the backoff until `attempts` runs out. `UnrecoverableError` fails immediately: use it for invalid data, missing entities, 4xx from upstreams. Map `AppError` the same way as for Kafka: `unavailable` → throw, anything else → `UnrecoverableError`.
- **Time limits.** Give the job's own calls timeouts. A job longer than `lockDuration` (default 30 s) without the lock being renewed looks stalled. The lock is renewed automatically while the event loop is free; a CPU-bound job blocks that renewal, so use a sandboxed processor (a file path instead of a class) or raise `lockDuration`.
- Failed jobs stay in the `failed` set (per `removeOnFail`) and can be retried from the dashboard after a fix. That is the dead-letter queue.

## Scheduled jobs (instead of `@Cron` with replicas)

```ts
@Injectable()
export class ReportsScheduler implements OnModuleInit {
  constructor(@InjectQueue('reports') private readonly reports: Queue) {}

  async onModuleInit() {
    // Upsert: safe to run on every boot of every instance; one schedule, one job per tick
    await this.reports.upsertJobScheduler(
      'daily-sales',
      { pattern: '0 3 * * *', tz: 'America/Sao_Paulo' },
      { name: 'daily-sales', data: {} },
    );
  }
}
```

Remove a schedule with `removeJobScheduler('daily-sales')`; deleting the code doesn't delete the schedule in Redis.

## Rate limits and flows

- Upstream allows 10 req/s: `@Processor('sms', { limiter: { max: 10, duration: 1_000 } })`. The limit is per queue, shared by all workers.
- Parent/child jobs (resize 5 images, then build the album): `FlowProducer` (`@InjectFlowProducer`). The parent runs after all children finish.
- Priorities: `priority: 1` (higher runs first). Use sparingly; a separate queue is clearer for "urgent vs bulk".

## Separate the worker process

Start with workers in the API process. Split when jobs hurt request latency (CPU, memory) or need a different scale:

```ts
// src/worker.ts — same codebase, no HTTP server
const app = await NestFactory.createApplicationContext(WorkerModule); // imports the processors
app.enableShutdownHooks(); // Nest closes workers on SIGTERM, waiting for active jobs
```

The API module then registers only the queues (producers), not the processors.

- **Render:** a Background Worker service with start command `node dist/worker.js`; the shutdown window is the same as for web services.
- **Railway:** a second service from the same repo with a different start command.
- Workers have no health route by default: alert on queue metrics instead (below).

## Observe

- Watch per queue: waiting count and age of the oldest waiting job (backlog), failed count, completed/min. A growing backlog means more `concurrency` or more workers, or a slow upstream.
- Dashboard: `@bull-board/nestjs` with `@bull-board/api`. Mount it behind admin auth only: it shows job data and can retry/delete jobs.
- Log `job.id` and `job.name` in every log line from a job.

## Tests

- Unit: call `processor.process(fakeJob)` directly with a plain object (`{ id, name, data, attemptsMade, opts }`); assert `UnrecoverableError` for permanent cases.
- Producer: mock the queue with `getQueueToken('email')` and assert `add` was called with the right name, data and `jobId`.
- Integration: real Redis (`@testcontainers/redis` or a service container in CI), add a job, wait for `completed`; run the same job twice and check the effect happened once.

## BullMQ 6 changes

| Outdated (≤ 5) | Current (6) |
|----------------|-------------|
| `queue.add(name, data, { repeat })`, `getRepeatableJobs()` | `upsertJobScheduler()` / `removeJobScheduler()` |
| `debounce` option | `deduplication` |
| `job.discard()` | throw `UnrecoverableError` |
| `ioredis` installed by bullmq | install `ioredis` yourself |
| `queue.client`, `worker.blockingClient` | `getBackend()` (Redis internals are no longer exposed) |
| `worker.resume()` sync | `await worker.resume()` |
| `paused` job state | jobs in a paused queue count as `waiting` |
| `RepeatOptions` `utc: true` | `tz: 'UTC'` |

BullMQ 6 also ships a PostgreSQL backend (`createPostgresBackend`), which would remove Redis from a Prisma + Postgres stack. `@nestjs/bullmq` 12.0.0 has no option for it yet, so treat it as experimental and recheck in the next review.
