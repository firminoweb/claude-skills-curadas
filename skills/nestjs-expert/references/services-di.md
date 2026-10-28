# Services & Dependency Injection

## Service Pattern

```typescript
import { Injectable, Logger, NotFoundException } from '@nestjs/common';
import { PrismaService } from '../prisma/prisma.service';
import { EmailService } from '../email/email.service';
import { hash } from 'argon2';

const publicUser = { id: true, email: true, name: true } as const;

@Injectable()
export class UsersService {
  private readonly logger = new Logger(UsersService.name);

  constructor(
    private readonly prisma: PrismaService,
    private readonly emailService: EmailService,
  ) {}

  async findOne(id: string) {
    const user = await this.prisma.user.findUnique({ where: { id }, select: publicUser });
    if (!user) throw new NotFoundException(`User ${id} not found`);
    return user;
  }

  async update(id: string, dto: UpdateUserDto) {
    // P2025 (not found) becomes 404 through the global PrismaExceptionFilter
    return this.prisma.user.update({ where: { id }, data: dto, select: publicUser });
  }

  async remove(id: string): Promise<void> {
    await this.prisma.user.delete({ where: { id } });
  }

  async register(dto: CreateUserDto) {
    const user = await this.prisma.user.create({
      data: { email: dto.email, name: dto.name, passwordHash: await hash(dto.password) },
      select: publicUser,
    });
    // Side effect after the write; a failed email must not undo the signup
    this.emailService.sendWelcome(user.email).catch((e) => this.logger.error(e));
    return user;
  }
}
```

- Inject `PrismaService` directly. Add a repository class only when the same queries are reused by several services or you need to swap the data source in tests.
- Duplicate-email handling (P2002) and the filter: `prisma.md`.

## Module with Providers

```typescript
@Module({
  imports: [EmailModule], // PrismaModule is @Global(); otherwise import it here
  controllers: [UsersController],
  providers: [UsersService],
  exports: [UsersService],  // Make available to other modules
})
export class UsersModule {}
```

## Custom Providers

```typescript
// Value provider
{ provide: 'FEATURE_FLAGS', useValue: { newCheckout: false } }

// Factory provider
{
  provide: 'CONFIG',
  useFactory: (configService: ConfigService) => ({
    apiUrl: configService.getOrThrow('API_URL'),
  }),
  inject: [ConfigService],
}

// Class provider
{ provide: LoggerService, useClass: CustomLoggerService }

// Async factory (Nest awaits it before the app starts)
{
  provide: 'REDIS',
  useFactory: async (config: ConfigService) => {
    const client = createClient({ url: config.getOrThrow('REDIS_URL') });
    await client.connect();
    return client;
  },
  inject: [ConfigService],
}
```

## Injection Patterns

```typescript
// Constructor injection (preferred)
constructor(private readonly usersService: UsersService) {}

// Token injection
constructor(@Inject('FEATURE_FLAGS') private flags: { newCheckout: boolean }) {}

// Optional injection
constructor(@Optional() private readonly cache?: CacheService) {}

// Property injection (use sparingly)
@Inject() private readonly logger: Logger;
```

## Scope

```typescript
// Default: Singleton (shared across app)
@Injectable()
export class SharedService {}

// Request-scoped: New instance per request
@Injectable({ scope: Scope.REQUEST })
export class RequestService {
  constructor(@Inject(REQUEST) private request: Request) {}
}

// Transient: New instance every injection
@Injectable({ scope: Scope.TRANSIENT })
export class HelperService {}
```

## Quick Reference

| Pattern | Use When |
|---------|----------|
| Constructor DI | Most cases (recommended) |
| `@Inject(token)` | Non-class tokens |
| `@Optional()` | Optional dependency |
| Factory provider | Dynamic configuration |
| Scope.REQUEST | Per-request state (makes every dependent provider request-scoped too; prefer `ClsModule`/AsyncLocalStorage for request context) |
| `onModuleInit` / `onModuleDestroy` | Connect / release resources. In NestJS 12 hooks run by module hierarchy level, so don't rely on sibling order |
