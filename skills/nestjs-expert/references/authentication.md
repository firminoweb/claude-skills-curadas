# Authentication & Guards

## JWT Strategy

```typescript
// jwt.strategy.ts
import { Injectable } from '@nestjs/common';
import { PassportStrategy } from '@nestjs/passport';
import { ExtractJwt, Strategy } from 'passport-jwt';
import { ConfigService } from '@nestjs/config';

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor(private config: ConfigService) {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      ignoreExpiration: false,
      secretOrKey: config.getOrThrow<string>('JWT_SECRET'),
    });
  }

  async validate(payload: { sub: string; email: string; role: string }) {
    return { userId: payload.sub, email: payload.email, role: payload.role };
  }
}
```

## JWT Auth Guard

```typescript
// jwt-auth.guard.ts
import { Injectable, ExecutionContext, UnauthorizedException } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';
import { Reflector } from '@nestjs/core';

@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {
  constructor(private reflector: Reflector) {
    super();
  }

  canActivate(context: ExecutionContext) {
    const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);
    if (isPublic) return true;
    return super.canActivate(context);
  }

  handleRequest(err: any, user: any) {
    if (err || !user) {
      throw err || new UnauthorizedException('Invalid token');
    }
    return user;
  }
}

// Public decorator (works on a method or a whole controller)
export const IS_PUBLIC_KEY = 'isPublic';
export const Public = () => SetMetadata(IS_PUBLIC_KEY, true);
```

## Roles Guard

```typescript
// roles.decorator.ts
export const Roles = (...roles: string[]) => SetMetadata('roles', roles);

// roles.guard.ts
@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const roles = this.reflector.getAllAndOverride<string[]>('roles', [
      context.getHandler(),
      context.getClass(),
    ]);
    if (!roles) return true;

    const { user } = context.switchToHttp().getRequest();
    return !!user && roles.includes(user.role);
  }
}

// Usage
@UseGuards(JwtAuthGuard, RolesGuard)
@Roles('admin')
@Get('admin')
adminEndpoint() {}
```

## Auth Service

```typescript
import { verify } from 'argon2';
import { createHash, randomUUID } from 'node:crypto';

@Injectable()
export class AuthService {
  constructor(
    private prisma: PrismaService,
    private jwt: JwtService,
    private config: ConfigService,
  ) {}

  async login(email: string, password: string) {
    const user = await this.prisma.user.findUnique({ where: { email } });
    // Same error for unknown email and wrong password (no user enumeration)
    if (!user || !(await verify(user.passwordHash, password))) {
      throw new UnauthorizedException('Invalid credentials');
    }
    return this.issueTokens(user.id, user.role);
  }

  private async issueTokens(userId: string, role: string) {
    const accessToken = await this.jwt.signAsync({ sub: userId, role }); // 15m, from JwtModule
    // Refresh token: different secret, stored hashed so a DB leak can't be replayed.
    // SHA-256 is enough here (the token is random, not a password) and allows lookup by jti
    const jti = randomUUID();
    const refreshToken = await this.jwt.signAsync(
      { sub: userId, jti },
      { secret: this.config.getOrThrow('JWT_REFRESH_SECRET'), expiresIn: '7d' },
    );
    await this.prisma.refreshToken.create({
      data: {
        id: jti,
        userId,
        tokenHash: createHash('sha256').update(refreshToken).digest('hex'),
        expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000),
      },
    });
    return { accessToken, refreshToken };
  }
}
```

- Hash passwords with argon2id (`argon2`) or bcrypt cost ≥ 12. Never store or log plain passwords or tokens.
- **Rotate** refresh tokens: on `/auth/refresh`, verify with the refresh secret, load the row by `jti`, compare the SHA-256, delete it and issue a new pair. A reused (already deleted) token means theft: revoke all of that user's tokens.
- Web clients: send the refresh token as an `httpOnly`, `Secure`, `SameSite` cookie, not in the JSON body. Mobile clients keep it in secure storage.
- Keep the JWT payload minimal (`sub`, `role`); no email or PII. If roles change often, load the user in `validate()` instead of trusting the token's role.
- Rate-limit `/auth/login` and `/auth/refresh` with `@nestjs/throttler` (`@Throttle({ default: { limit: 5, ttl: 60_000 } })`).

## Auth Module Setup

```typescript
@Module({
  imports: [
    PassportModule.register({ defaultStrategy: 'jwt' }),
    JwtModule.registerAsync({
      inject: [ConfigService],
      useFactory: (config: ConfigService) => ({
        secret: config.getOrThrow('JWT_SECRET'),
        signOptions: { expiresIn: '15m' },
      }),
    }),
    UsersModule,
  ],
  providers: [AuthService, JwtStrategy],
  exports: [AuthService],
})
export class AuthModule {}
```

## Apply Guards Globally

```typescript
// app.module.ts
@Module({
  providers: [
    { provide: APP_GUARD, useClass: JwtAuthGuard }, // secure by default; opt out with @Public()
    { provide: APP_GUARD, useClass: RolesGuard },
  ],
})
export class AppModule {}
```

## Quick Reference

| Component | Purpose |
|-----------|---------|
| `JwtStrategy` | Validate JWT tokens |
| `JwtAuthGuard` | Protect routes |
| `RolesGuard` | Role-based access |
| `@Public()` | Skip auth |
| `@Roles('admin')` | Require role |
| `@UseGuards()` | Apply guard |
