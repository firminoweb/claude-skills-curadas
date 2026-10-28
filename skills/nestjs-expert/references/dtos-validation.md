# DTOs & Validation

Two supported styles in NestJS 12. Use the one the project already has:
- **class-validator DTOs** + `ValidationPipe`: decorators, Swagger via the CLI plugin.
- **Standard Schema** (Zod 4, Valibot, ArkType) + `StandardSchemaValidationPipe`: one schema shared with the frontend (e.g. a monorepo `packages/validation`).

## DTO Patterns

```typescript
import {
  IsEmail, IsString, IsOptional, IsBoolean, IsInt,
  MinLength, MaxLength, Min, Max, IsUUID, IsEnum,
  IsArray, ArrayMinSize, ValidateNested, Matches
} from 'class-validator';
import { Type, Transform } from 'class-transformer';
import { ApiProperty, ApiPropertyOptional, PartialType, OmitType, PickType } from '@nestjs/swagger';

export class CreateUserDto {
  @ApiProperty({ example: 'user@example.com' })
  @IsEmail()
  email: string;

  @ApiProperty({ minLength: 8 })
  @IsString()
  @MinLength(8)
  @Matches(/^(?=.*[A-Z])(?=.*\d)/, { message: 'Password must contain uppercase and digit' })
  password: string;

  @ApiProperty()
  @IsString()
  @MinLength(2)
  @MaxLength(50)
  name: string;

  @ApiPropertyOptional({ enum: UserRole, default: UserRole.USER })
  @IsOptional()
  @IsEnum(UserRole)
  role?: UserRole = UserRole.USER;
}

// Partial for updates (all fields optional)
export class UpdateUserDto extends PartialType(
  OmitType(CreateUserDto, ['password'] as const)
) {}

// Pick specific fields
export class LoginDto extends PickType(CreateUserDto, ['email', 'password'] as const) {}
```

## Nested Validation

```typescript
export class CreateOrderDto {
  @ApiProperty({ type: [OrderItemDto] })
  @IsArray()
  @ArrayMinSize(1)
  @ValidateNested({ each: true })
  @Type(() => OrderItemDto)
  items: OrderItemDto[];

  @ApiProperty({ type: AddressDto })
  @ValidateNested()
  @Type(() => AddressDto)
  shippingAddress: AddressDto;
}

export class OrderItemDto {
  @IsUUID()
  productId: string;

  @IsInt()
  @Min(1)
  @Max(100)
  quantity: number;
}
```

## Custom Validation

```typescript
import { registerDecorator, ValidationOptions, ValidationArguments } from 'class-validator';

// Custom decorator
export function IsStrongPassword(options?: ValidationOptions) {
  return function (object: object, propertyName: string) {
    registerDecorator({
      name: 'isStrongPassword',
      target: object.constructor,
      propertyName,
      options,
      validator: {
        validate(value: string) {
          return /^(?=.*[A-Z])(?=.*[a-z])(?=.*\d)(?=.*[@$!%*?&]).{8,}$/.test(value);
        },
        defaultMessage(): string {
          return 'Password must contain uppercase, lowercase, digit, and special character';
        },
      },
    });
  };
}

// Usage
@IsStrongPassword()
password: string;
```

## Transform & Sanitize

```typescript
export class QueryDto {
  @Type(() => Number)
  @IsInt()
  @Min(1)
  page: number = 1;

  @Transform(({ value }) => value?.trim().toLowerCase())
  @IsString()
  @IsOptional()
  search?: string;

  // Query strings are always strings: map explicitly, never Boolean('false') (= true)
  @Transform(({ obj }) => obj.isActive === 'true')
  @IsBoolean()
  isActive: boolean = true;
}
```

Don't combine `enableImplicitConversion: true` with boolean query params: it turns `'false'` into `true` before your `@Transform` sees it. Convert explicitly with `@Type`/`@Transform` instead.

## Enable Validation Globally

```typescript
// configureApp(app), called from main.ts and from E2E tests
app.useGlobalPipes(new ValidationPipe({
  whitelist: true,            // Strip unknown properties
  forbidNonWhitelisted: true, // 400 on unknown properties (mass-assignment guard)
  transform: true,            // Plain object -> DTO instance, primitives converted
  errorFormat: 'grouped',     // NestJS 12: { field: [messages] } instead of a flat list (default 'list')
}));
```

`errorFormat: 'grouped'` is easier to map onto form fields on the frontend; keep the default if clients already parse the list.

## Standard Schema (Zod) — NestJS 12

```typescript
// users/users.schemas.ts (or a shared package)
import { z } from 'zod';

export const createUserSchema = z.object({
  email: z.email(),
  name: z.string().trim().min(2).max(50),
  password: z.string().min(12).max(128),
}).strict(); // reject unknown keys, like forbidNonWhitelisted

export type CreateUserInput = z.infer<typeof createUserSchema>;

// users.controller.ts
@Post()
create(@Body({ schema: createUserSchema }) body: CreateUserInput) {
  return this.usersService.create(body);
}

@Get(':id')
findOne(@Param('id', { schema: z.uuid() }) id: string) {
  return this.usersService.findOne(id);
}

// configureApp(app)
app.useGlobalPipes(new StandardSchemaValidationPipe()); // from @nestjs/common
```

- The `schema` option only attaches metadata; without the global (or route-level) pipe **nothing is validated**.
- The pipe skips parameters that have no schema, so it can run globally next to `ValidationPipe` during a migration.
- Responses: `@UseInterceptors(StandardSchemaSerializerInterceptor)` + `@SerializeOptions({ schema: userResponseSchema })` to validate and strip outgoing data.
- Error shape: customize with `exceptionFactory`.
- Swagger: the docs say the same schemas can feed OpenAPI, but check the generated document. If a route shows an empty body, document it with `@ApiBody({ schema: z.toJSONSchema(createUserSchema) })`.
- Before NestJS 12: a small custom `ZodValidationPipe` (`schema.safeParse` → `BadRequestException`) or `nestjs-zod`.

## Quick Reference

| Decorator | Purpose |
|-----------|---------|
| `@IsString()` | String type |
| `@IsEmail()` | Valid email |
| `@MinLength(n)` | Min string length |
| `@IsInt()`, `@Min(n)` | Integer validation |
| `@IsEnum(Enum)` | Enum value |
| `@IsOptional()` | Optional field |
| `@ValidateNested()` | Validate nested object |
| `@Type(() => Class)` | Transform to class |
| `@Transform()` | Custom transform |
| `@Body({ schema })` | Standard Schema validation (NestJS 12) |
| `StandardSchemaValidationPipe` | Pipe that runs those schemas |
| `PartialType()` | All fields optional |
| `OmitType()` | Exclude fields |
| `PickType()` | Include only fields |
