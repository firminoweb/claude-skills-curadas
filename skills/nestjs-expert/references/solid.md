# SOLID and Design Smells in NestJS

Adapted from [amirtaherkhani/nestjs-skills](https://github.com/amirtaherkhani/nestjs-skills) (MIT), `nestjs-oop-design-patterns`, condensed.

Use SOLID as **questions when something hurts**, not as a checklist to apply up front. A smell is a reason to look, not proof of a defect. The simplest design that keeps likely changes local wins.

## The five as questions

| Principle | Ask | Nest + Prisma answer |
|-----------|-----|----------------------|
| **S**ingle responsibility | Do different people/reasons change this class? Do its dependencies form separate groups? Does a test need unrelated setup? | Split by reason to change, not by method count. `OrdersService` coordinating an order is one responsibility; the Stripe call and the PDF are others |
| **O**pen/closed | Has the same `switch` on type/provider changed several times already? | Only then extract a strategy map (below). A small, stable `if` stays |
| **L**iskov | Does every implementation keep the contract: same errors, side effects, idempotency? | A `CachedPaymentGateway` that silently skips the charge when Redis is down is not a substitute, even with matching types |
| **I**nterface segregation | Does a consumer depend on methods it never calls? | Ports shaped by the consumer (`FindUser`, `ChargePayment`), not a mirror of the whole SDK |
| **D**ependency inversion | Does business code import a vendor SDK? | The service depends on a token it owns; the module wires the adapter. Only for things that vary or need faking in tests: Prisma itself doesn't need a port in Level 1 |

## DI in practice

```ts
// payments/payment-gateway.ts — owned by the business side
export interface PaymentGateway {
  charge(input: { amountCents: number; customerId: string; idempotencyKey: string }): Promise<{ chargeId: string }>;
}
export const PAYMENT_GATEWAY = Symbol('PAYMENT_GATEWAY');

// payments/payments.module.ts — the only place that knows Stripe exists
providers: [{ provide: PAYMENT_GATEWAY, useClass: StripePaymentGateway }],

// orders/orders.service.ts
constructor(@Inject(PAYMENT_GATEWAY) private readonly payments: PaymentGateway) {}
```

Tests swap it with `{ provide: PAYMENT_GATEWAY, useValue: fakeGateway }`. The adapter translates Stripe errors into `AppError` (`references/errors-resilience.md`) so no Stripe type leaks out.

## Strategy without a hierarchy

When the variation is real (repeated `switch` on region, carrier, plan):

```ts
export const SHIPPING_POLICIES = Symbol('SHIPPING_POLICIES');

providers: [
  CorreiosPolicy, JadlogPolicy,
  {
    provide: SHIPPING_POLICIES,
    useFactory: (c: CorreiosPolicy, j: JadlogPolicy) => new Map<Carrier, ShippingPolicy>([['correios', c], ['jadlog', j]]),
    inject: [CorreiosPolicy, JadlogPolicy],
  },
],

// service
const policy = this.policies.get(dto.carrier); // carrier already validated by the DTO enum
if (!policy) throw new AppError('invalid', 'UNKNOWN_CARRIER', `Unknown carrier ${dto.carrier}`);
```

Never pick a class or provider by name straight from unvalidated input.

## Smells and the fix

| Smell | Sign | Fix |
|-------|------|-----|
| God service | 8+ dependencies in clusters, 1,000+ lines, every change touches it | Split by operation or subdomain; keep a thin facade only if callers benefit |
| `BaseService<T>` / generic CRUD inheritance | Children override auth, filters or events inconsistently | Explicit methods per feature; share small helpers by composition |
| Leaky adapter | Services import SDK types, Prisma errors bubble as 500, `HttpException` in a queue worker | Translate at the edge; Prisma errors in the filter or the adapter |
| Anemic rules scattered | The same status check (`if (order.status !== 'draft')`) in 4 places | One function/method owns the rule (`assertCanPay(order)`), even with plain Prisma types |
| Primitive obsession | Money as `number` floats, cents vs reais mix-ups, ids of different entities swapped | Integer cents or `Prisma.Decimal`; a branded type where mix-ups actually happened |
| Circular deps / `forwardRef` | Two services call each other | Extract the shared rule to a third provider, or have one side emit an event |
| Service locator | `moduleRef.get()` inside business methods | Constructor injection; `ModuleRef` only for real plugin/dynamic cases |
| Flag arguments | `update(dto, isAdmin, skipEmail)` changes behavior a lot | Separate named methods (`adminUpdate`) |
| Speculative generality | Interface with one implementation and no test fake, factory with one product, unused options | Delete it. Git keeps it if a real second case appears |

## Refactoring loop

1. Pin current behavior with a test (E2E or service test).
2. Make one structural change.
3. Keep the public contract (routes, GraphQL schema, error codes) unchanged.
4. Run the tests; remove transitional code.

Success is a change that now touches fewer files, not a deeper folder tree.
