# GraphQL (code-first, Apollo)

Baseline: `@nestjs/graphql` 14, `@nestjs/apollo` 14, `@apollo/server` 5 (Oct 2026).

## Install

```bash
npm i @nestjs/graphql@14 @nestjs/apollo@14 @apollo/server@5 @as-integrations/express5 graphql@16
# Fastify: @as-integrations/fastify instead of express5, or @nestjs/mercurius + mercurius
```

Pin `graphql@16`: npm's `latest` is 17, but `@apollo/server` 5 still requires `^16.11`.

## Module

```ts
// app.module.ts
GraphQLModule.forRootAsync<ApolloDriverConfig>({
  driver: ApolloDriver,
  imports: [LoadersModule],
  inject: [LoadersFactory, ConfigService],
  useFactory: (loaders: LoadersFactory, config: ConfigService) => {
    const prod = config.get('NODE_ENV') === 'production';
    return {
      autoSchemaFile: join(process.cwd(), 'src/schema.gql'), // commit it: the schema diff shows up in PRs
      sortSchema: true,
      graphiql: !prod,        // `playground` is a deprecated alias
      introspection: !prod,
      validationRules: [depthLimit(8)], // graphql-depth-limit
      context: ({ req, res }) => ({ req, res, loaders: loaders.create() }), // new loaders per request
    };
  },
}),
```

## Types: separate from Prisma models

```ts
@ObjectType()
export class Post {
  @Field(() => ID) id: string;
  @Field() title: string;
  @Field() authorId: string; // used by the author field resolver
  @Field() createdAt: Date;
}

@ObjectType()
export class User {
  @Field(() => ID) id: string;
  @Field() name: string;
  // passwordHash simply doesn't exist here; the resolver still `select`s only these fields
}
```

Inputs and args are classes with `class-validator`; the global `ValidationPipe` validates them like REST DTOs.

```ts
@ArgsType()
export class PageArgs {
  @Field(() => Int, { defaultValue: 20 }) @IsInt() @Min(1) @Max(100) take: number;
  @Field({ nullable: true }) @IsOptional() @IsUUID() cursor?: string;
}
```

## Resolver

```ts
@Resolver(() => Post)
export class PostsResolver {
  constructor(private readonly posts: PostsService) {}

  @Query(() => [Post])
  myPosts(@Args() { take, cursor }: PageArgs, @CurrentUser() user: AuthUser) {
    return this.posts.listByOwner(user.userId, take, cursor); // ownership in the where
  }

  @Mutation(() => Post)
  createPost(@Args('input') input: CreatePostInput, @CurrentUser() user: AuthUser) {
    return this.posts.create(user.userId, input);
  }

  @ResolveField(() => User)
  author(@Parent() post: Post, @Context('loaders') loaders: Loaders) {
    return loaders.userById.load(post.authorId); // never prisma.user.findUnique here: N+1
  }
}
```

Resolvers stay thin like controllers: business logic and Prisma live in the service, shared with REST.

## DataLoader (N+1)

```ts
// loaders.factory.ts — singleton provider; the loaders it creates live for one request
@Injectable()
export class LoadersFactory {
  constructor(private readonly prisma: PrismaService) {}

  create() {
    return {
      userById: new DataLoader<string, PublicUser | null>(async (ids) => {
        const rows = await this.prisma.user.findMany({ where: { id: { in: [...ids] } }, select: publicUser });
        const byId = new Map(rows.map((u) => [u.id, u]));
        return ids.map((id) => byId.get(id) ?? null); // same order and length as ids
      }),
    };
  }
}
export type Loaders = ReturnType<LoadersFactory['create']>;
```

Create loaders in the context, not as `Scope.REQUEST` providers: request scope would make every resolver that injects them request-scoped too (`references/performance.md`). A loader caches per request only; never share one across requests (it would leak data between users).

## Auth: guards need the GraphQL request

`context.switchToHttp().getRequest()` returns undefined in a resolver. Use one helper in both guards from `references/authentication.md`:

```ts
// auth/get-request.ts
export function getRequest(ctx: ExecutionContext) {
  return ctx.getType<GqlContextType>() === 'graphql'
    ? GqlExecutionContext.create(ctx).getContext().req
    : ctx.switchToHttp().getRequest();
}

// JwtAuthGuard: add
getRequest(ctx: ExecutionContext) { return getRequest(ctx); }
// RolesGuard: const { user } = getRequest(context);

export const CurrentUser = createParamDecorator((_: unknown, ctx: ExecutionContext) => getRequest(ctx).user);
```

- The global `APP_GUARD` covers resolvers too; `@Public()` works the same way.
- `@nestjs/throttler` needs a GraphQL-aware guard: extend `ThrottlerGuard` and override `getRequestResponse` to return `{ req, res }` from the GQL context.
- Field-level authorization (a field only admins see): guard on `@ResolveField`, or return null by rule in the service. Check ownership in the query, same as REST.

## Limit query cost

Depth limit (above) plus a complexity limit, so one query can't ask for 100 posts × 100 comments × 100 authors:

```ts
// Plugin from '@nestjs/apollo' (not '@nestjs/graphql'); GraphQLSchemaHost from '@nestjs/graphql'
@Plugin()
export class ComplexityPlugin implements ApolloServerPlugin {
  constructor(private readonly schemaHost: GraphQLSchemaHost) {}

  async requestDidStart(): Promise<GraphQLRequestListener<BaseContext>> {
    const { schema } = this.schemaHost;
    return {
      async didResolveOperation({ request, document }) {
        const complexity = getComplexity({
          schema, operationName: request.operationName, query: document, variables: request.variables,
          estimators: [fieldExtensionsEstimator(), simpleEstimator({ defaultComplexity: 1 })],
        });
        if (complexity > 1000) throw new GraphQLError(`Query too complex: ${complexity}`, { extensions: { code: 'QUERY_TOO_COMPLEX' } });
      },
    };
  }
}
// List fields: @Field(() => [Post], { complexity: ({ args, childComplexity }) => args.take * childComplexity })
```

Register `ComplexityPlugin` as a provider. Public APIs with known clients: consider persisted queries (only allow known operations).

## Errors

- GraphQL responses are HTTP 200 with an `errors` array; clients read `errors[].extensions.code`.
- Nest HTTP exceptions are mapped automatically (`autoTransformHttpErrors`, default on): `NotFoundException` → `extensions.code: 'NOT_FOUND'`.
- The `AppErrorFilter` from `references/errors-resilience.md` writes an HTTP response; in GraphQL it must return an error instead:

```ts
if (host.getType<GqlContextType>() === 'graphql') {
  return new GraphQLError(e.message, { extensions: { code: e.code } });
}
```

- Hide internals in production with `formatError`: keep `message`/`code` for known codes, replace anything with `INTERNAL_SERVER_ERROR` by a generic message. Stack traces are omitted when `NODE_ENV=production`.
- Partial data is normal: one failing field nulls that field (if nullable) and the rest returns. Make fields that can fail nullable on purpose.

## Subscriptions

`subscriptions: { 'graphql-ws': true }`. Authenticate in `onConnect` (token from `connectionParams`), not per message. The default in-memory `PubSub` only works on one instance; with replicas use `graphql-redis-subscriptions`.

## Tests

E2E with supertest: `POST /graphql` with `{ query, variables }`. Assert `body.errors?.[0].extensions.code` (status is 200 even on error), that a non-owner gets nothing, that a deep or expensive query is rejected, and that the user field resolver runs one query for N posts (count queries with Prisma's query log).

## Outdated habits

| Outdated | Current |
|----------|---------|
| `playground: true` | `graphiql: true` (non-production only) |
| `apollo-server-express` | `@apollo/server` 5 + `@as-integrations/express5` |
| `graphql@latest` | `graphql@16` (Apollo Server 5 peer) |
| `ApolloError` / `UserInputError` classes | `new GraphQLError(msg, { extensions: { code } })` |
| `findUnique` inside `@ResolveField` | DataLoader from the context |
| `forRoot` with `process.env` | `forRootAsync` + `ConfigService` |
