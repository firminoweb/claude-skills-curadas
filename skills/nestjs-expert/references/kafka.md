# Kafka

Baseline: `@nestjs/microservices` 12 with the `kafkajs` transport, Oct 2026.

## Kafka or a queue?

| Need | Use |
|------|-----|
| Background jobs inside one app (emails, PDFs, retries with delay) | BullMQ (`@nestjs/bullmq`) + Redis |
| Events consumed by several services, replay of history, high volume, ordering per key | Kafka |
| A synchronous answer from another service | HTTP or gRPC, not Kafka request/response |

Don't add Kafka for one app's background jobs: it brings brokers, partitions, consumer groups and lag to operate.

## Install and client choice

```bash
npm i @nestjs/microservices@12 kafkajs
```

The Nest transport only speaks `kafkajs`, whose last release was 2.2.4 in Feb 2023. It works, but nobody maintains it. When you need a maintained client or features kafkajs lacks, use `@confluentinc/kafka-javascript` (Confluent, KafkaJS-compatible API) directly in your own provider instead of the Nest transport. Decide this per project; don't mix both.

## Consumer (hybrid app: HTTP + Kafka in one process)

```ts
// main.ts
const app = await NestFactory.create(AppModule);
app.connectMicroservice<MicroserviceOptions>({
  transport: Transport.KAFKA,
  options: {
    client: {
      clientId: 'orders-api',
      brokers: config.getOrThrow<string>('KAFKA_BROKERS').split(','),
      ssl: true,
      sasl: { mechanism: 'scram-sha-512', username: config.getOrThrow('KAFKA_USER'), password: config.getOrThrow('KAFKA_PASSWORD') },
    },
    consumer: { groupId: 'orders-api' },
    postfixId: '', // otherwise Nest appends "-server": the real group would be "orders-api-server"
    run: { partitionsConsumedConcurrently: 3 },
  },
}, { inheritAppConfig: true }); // global pipes/filters/interceptors also apply to Kafka handlers
app.enableShutdownHooks();
await app.startAllMicroservices();
await app.listen(port);
```

```ts
@Controller()
export class PaymentsConsumer {
  constructor(private readonly payments: PaymentsService, private readonly dlt: DeadLetterPublisher) {}

  // <string>: without it, Nest 12's typed overload types extra params as `unknown` and `ctx: KafkaContext` fails to compile
  @EventPattern<string>('payments.confirmed.v1')
  async onPaymentConfirmed(@Payload() event: PaymentConfirmedEvent, @Ctx() ctx: KafkaContext) {
    try {
      await this.payments.applyConfirmation(event); // idempotent, see below
    } catch (e) {
      if (e instanceof AppError && e.kind !== 'unavailable') {
        await this.dlt.publish(ctx, e); // permanent: park it and move on
        return;
      }
      throw e; // transient: let it be redelivered
    }
  }
}
```

## Delivery semantics (read before writing a handler)

- **At least once.** The same message can arrive twice (rebalance, crash before commit). Every handler must be idempotent.
- **A handler that throws is not committed.** kafkajs retries it, and when retries run out it restarts the consumer from the last committed offset. A message that always fails ("poison message") blocks its partition forever. So:
  - permanent failure (invalid payload, business rule, missing entity) → publish to a dead-letter topic (`<topic>.dlt`, original value + error headers) and **return normally**;
  - transient failure (database down, upstream 503) → throw, and it will be retried.
- **Ordering is per partition only.** Messages with the same `key` go to the same partition, so key by the entity whose order matters (`orderId`). Nothing is ordered across keys.
- Long handlers: call `await ctx.getHeartbeat()()` during the work, or the broker thinks the consumer died and rebalances.

Idempotency with a unique constraint (Prisma shown; MongoDB is the same with a unique index and code 11000):

```ts
async applyConfirmation(event: PaymentConfirmedEvent) {
  await this.prisma.$transaction(async (tx) => {
    try {
      await tx.processedEvent.create({ data: { id: event.eventId } }); // @id = eventId
    } catch (e) {
      if (e instanceof Prisma.PrismaClientKnownRequestError && e.code === 'P2002') return; // already done
      throw e;
    }
    await tx.order.update({ where: { id: event.data.orderId }, data: { status: 'PAID' } });
  });
}
```

## Producer

```ts
// orders.module.ts
ClientsModule.registerAsync([{
  name: 'KAFKA',
  inject: [ConfigService],
  useFactory: (config: ConfigService) => ({
    transport: Transport.KAFKA,
    options: { client: { clientId: 'orders-api', brokers: /* same as above */ }, producerOnlyMode: true },
  }),
}]),

// service
constructor(@Inject('KAFKA') private readonly kafka: ClientKafka) {}

async publish(event: OrderCreatedEvent) {
  // emit() sends immediately, but a failure is only seen if you await it
  await lastValueFrom(this.kafka.emit('orders.created.v1', {
    key: event.data.orderId,
    value: event,
    headers: { 'event-type': 'OrderCreated', 'event-version': '1' },
  }));
}
```

**Publishing after a database write:** writing to the database and then publishing can lose the event (crash in between) or publish an event for a write that rolled back. When the event matters, use an outbox: insert an `OutboxEvent` row in the same transaction as the write, and a worker (cron or loop) publishes pending rows and marks them sent. The worker can publish twice; consumers are idempotent anyway (`references/errors-resilience.md`).

## Event contract

```ts
export class OrderCreatedEvent {
  @IsUUID() eventId: string;          // dedupe key for consumers
  @IsISO8601() occurredAt: string;
  @ValidateNested() @Type(() => OrderCreatedData) data: OrderCreatedData;
}
```

- Validate incoming events like HTTP DTOs (with `inheritAppConfig: true`, the global `ValidationPipe` runs). A validation failure is a permanent error → dead-letter topic.
- Name topics `<domain>.<event>.v<major>`. Additive changes (new optional field) keep the version; removing or changing a field means a new topic version, with both published during the migration.
- Events are facts in the past tense (`OrderCreated`), carry ids and the data consumers need, never secrets or full user records.
- With several teams/services, use a schema registry (Avro/Protobuf/JSON Schema) instead of only TypeScript classes.

## Avoid

- `@MessagePattern` request/response over Kafka: needs reply topics and `subscribeToResponseOf`, adds latency and couples services. Use HTTP/gRPC for synchronous calls.
- Auto-creating topics in production: create topics with explicit partitions and retention (IaC or the provider console).
- Huge messages: Kafka's default limit is about 1 MB; put the file in object storage and send the URL.

## Tests and operations

- Unit: call the handler method directly with a payload and a fake `KafkaContext`.
- Integration: `@testcontainers/kafka`, then test duplicate delivery (same `eventId` twice → one effect) and a poison message (goes to the DLT, the next message is still processed).
- Monitor **consumer lag** per group and partition, DLT volume, and handler error rate. Rising lag means consumers can't keep up: more partitions + consumers, or a faster handler.
- Managed Kafka: Confluent Cloud, Aiven or Redpanda Cloud (Render and Railway have no managed Kafka). The number of partitions caps consumer parallelism in a group.
