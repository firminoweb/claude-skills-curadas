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

## Social Login (Google)

Social login only proves who the user is. After that, issue **our own** tokens with `issueTokens()` above; never use Google's tokens as the API session.

```prisma
model User {
  id           String    @id @default(uuid())
  email        String    @unique
  name         String?
  role         String    @default("user")
  passwordHash String?   // null for social-only accounts
  accounts     Account[]
}

model Account {
  id         String @id @default(uuid())
  provider   String // 'google', 'apple'
  providerId String // the provider's `sub`, stable; email can change
  userId     String
  user       User   @relation(fields: [userId], references: [id], onDelete: Cascade)
  @@unique([provider, providerId])
}
```

```ts
// Shared by both flows below
async loginWithProvider(p: { provider: string; providerId: string; email: string; emailVerified: boolean; name?: string }) {
  const account = await this.prisma.account.findUnique({
    where: { provider_providerId: { provider: p.provider, providerId: p.providerId } },
    include: { user: true },
  });
  if (account) return this.issueTokens(account.user.id, account.user.role);

  // Linking to an existing user by email ONLY if the provider verified it; otherwise anyone
  // could create a Google account with the victim's email and take over their account
  if (!p.emailVerified) throw new UnauthorizedException('Email not verified by provider');

  const user = await this.prisma.user.upsert({
    where: { email: p.email },
    update: { accounts: { create: { provider: p.provider, providerId: p.providerId } } },
    create: { email: p.email, name: p.name, accounts: { create: { provider: p.provider, providerId: p.providerId } } },
  });
  return this.issueTokens(user.id, user.role);
}
```

**A) Mobile app or SPA with the Google SDK (simplest, stateless).** The client signs in with Google (Expo/React Native `@react-native-google-signin/google-signin`, Google Identity Services on the web) and sends the ID token. The API verifies it:

```ts
import { OAuth2Client } from 'google-auth-library';

private readonly google = new OAuth2Client();

@Public()
@Throttle({ default: { limit: 10, ttl: 60_000 } })
@Post('auth/google')
async googleLogin(@Body() { idToken }: GoogleLoginDto) {
  const ticket = await this.google.verifyIdToken({
    idToken,
    audience: this.config.getOrThrow<string>('GOOGLE_CLIENT_IDS').split(','), // web, iOS and Android client ids
  }).catch(() => { throw new UnauthorizedException(); });
  const t = ticket.getPayload()!;
  return this.auth.loginWithProvider({
    provider: 'google', providerId: t.sub, email: t.email!, emailVerified: t.email_verified === true, name: t.name,
  });
}
```

`verifyIdToken` checks signature, expiry, issuer and audience. The `audience` check is what stops a token issued to someone else's app from logging in here.

**B) Server redirect flow (web without the Google SDK).** `passport-google-oauth20`:

```ts
@Injectable()
export class GoogleStrategy extends PassportStrategy(Strategy, 'google') {
  constructor(config: ConfigService, private readonly auth: AuthService) {
    super({
      clientID: config.getOrThrow('GOOGLE_CLIENT_ID'),
      clientSecret: config.getOrThrow('GOOGLE_CLIENT_SECRET'),
      callbackURL: config.getOrThrow('GOOGLE_CALLBACK_URL'), // absolute, registered in Google Console
      scope: ['openid', 'email', 'profile'],
      state: true, // CSRF protection on the callback
    });
  }

  validate(_at: string, _rt: string, profile: Profile) {
    return this.auth.loginWithProvider({
      provider: 'google', providerId: profile.id, email: profile.emails![0].value,
      emailVerified: profile._json.email_verified === true, name: profile.displayName,
    });
  }
}

// controller
@Public() @UseGuards(AuthGuard('google')) @Get('auth/google') start() {}

@Public() @UseGuards(AuthGuard('google')) @Get('auth/google/callback')
callback(@Req() req, @Res() res: Response) {
  const { refreshToken } = req.user; // what validate() returned
  res.cookie('refresh_token', refreshToken, { httpOnly: true, secure: true, sameSite: 'lax', path: '/auth' });
  res.redirect(this.config.getOrThrow('FRONTEND_URL') + '/auth/done'); // frontend then calls /auth/refresh
}
```

- `state: true` needs somewhere to keep the state between redirect and callback: passport stores it in `req.session`, so add a session middleware scoped to `/auth/google*` (or pass a custom `store`). Without it the callback fails with "requires session support".
- **Never put tokens in the redirect URL** (they end up in history, logs and `Referer`). Set the refresh cookie and let the frontend get an access token via `/auth/refresh`. The refresh cookie needs `sameSite: 'lax'` here: `strict` isn't sent on the redirect back from Google.
- Redirect only to a configured `FRONTEND_URL`, never to a `?redirect=` value from the request (open redirect).
- `passport-google-oauth20` hasn't had a release since 2019 but still works. For several OIDC providers, `openid-client` (v6) handles PKCE and discovery.
- Apple, GitHub: same `Account` table, same `loginWithProvider`; only the verification step changes. GitHub's email may be unverified or private: fetch `/user/emails` and use only `verified: true`.

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
