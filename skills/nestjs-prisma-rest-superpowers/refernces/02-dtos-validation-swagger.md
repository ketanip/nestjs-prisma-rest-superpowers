# 02 — DTOs, Validation & Swagger Decorators

## Required Packages

```bash
npm install class-validator class-transformer @nestjs/swagger
```

## Core DTO Patterns

### Create DTO

```typescript
// dto/create-user.dto.ts
import { ApiProperty, ApiPropertyOptional } from '@nestjs/swagger';
import {
  IsEmail, IsString, IsOptional, IsEnum, IsUUID, MinLength,
  MaxLength, IsUrl, IsBoolean, IsInt, Min, Max, ValidateNested,
  IsArray, ArrayMinSize,
} from 'class-validator';
import { Type } from 'class-transformer';
import { UserRole } from '@prisma/client';

export class CreateUserDto {
  @ApiProperty({ example: 'john@example.com', description: 'User email address' })
  @IsEmail()
  email: string;

  @ApiProperty({ example: 'John', minLength: 1, maxLength: 100 })
  @IsString()
  @MinLength(1)
  @MaxLength(100)
  firstName: string;

  @ApiProperty({ example: 'Doe', minLength: 1, maxLength: 100 })
  @IsString()
  @MinLength(1)
  @MaxLength(100)
  lastName: string;

  @ApiPropertyOptional({ enum: UserRole, default: UserRole.USER })
  @IsOptional()
  @IsEnum(UserRole)
  role?: UserRole = UserRole.USER;

  @ApiPropertyOptional({ example: 'https://example.com/avatar.jpg' })
  @IsOptional()
  @IsUrl()
  avatarUrl?: string;

  // Nested object DTO
  @ApiPropertyOptional({ type: () => AddressDto })
  @IsOptional()
  @ValidateNested()
  @Type(() => AddressDto)
  address?: AddressDto;

  // Array of UUIDs
  @ApiPropertyOptional({ type: [String], format: 'uuid' })
  @IsOptional()
  @IsArray()
  @IsUUID('4', { each: true })
  @ArrayMinSize(1)
  tagIds?: string[];
}

export class AddressDto {
  @ApiProperty({ example: '123 Main St' })
  @IsString()
  street: string;

  @ApiProperty({ example: 'New York' })
  @IsString()
  city: string;

  @ApiPropertyOptional({ example: 'NY' })
  @IsOptional()
  @IsString()
  state?: string;

  @ApiProperty({ example: 'US' })
  @IsString()
  @MinLength(2)
  @MaxLength(2)
  countryCode: string;
}
```

### Update DTO — use `PartialType`

```typescript
// dto/update-user.dto.ts
import { PartialType, OmitType } from '@nestjs/swagger';
import { CreateUserDto } from './create-user.dto';

// PartialType makes all fields optional AND preserves all Swagger decorators
export class UpdateUserDto extends PartialType(
  OmitType(CreateUserDto, ['email'] as const), // email not updatable
) {}
```

### Response DTO — controls what is serialized back

```typescript
// dto/user-response.dto.ts
import { ApiProperty, ApiPropertyOptional } from '@nestjs/swagger';
import { Exclude, Expose, Transform, Type } from 'class-transformer';
import { UserRole } from '@prisma/client';

@Exclude() // exclude everything by default
export class UserResponseDto {
  @Expose()
  @ApiProperty({ format: 'uuid' })
  id: string;

  @Expose()
  @ApiProperty()
  email: string;

  @Expose()
  @ApiProperty()
  firstName: string;

  @Expose()
  @ApiProperty()
  lastName: string;

  @Expose()
  @ApiProperty({ enum: UserRole })
  role: UserRole;

  @Expose()
  @ApiPropertyOptional()
  avatarUrl?: string;

  // Computed field
  @Expose()
  @ApiProperty()
  @Transform(({ obj }) => `${obj.firstName} ${obj.lastName}`)
  fullName: string;

  @Expose()
  @ApiProperty({ type: () => AddressResponseDto })
  @Type(() => AddressResponseDto)
  address?: AddressResponseDto;

  @Expose()
  @ApiProperty()
  createdAt: Date;

  @Expose()
  @ApiProperty()
  updatedAt: Date;

  // NEVER expose password, tokens, internal fields
  // @Exclude() is the default due to class-level @Exclude()
  password?: string; // stays hidden
}

@Exclude()
export class AddressResponseDto {
  @Expose() @ApiProperty() street: string;
  @Expose() @ApiProperty() city: string;
  @Expose() @ApiPropertyOptional() state?: string;
  @Expose() @ApiProperty() countryCode: string;
}
```

### DTOs for Path/Query Params

```typescript
// dto/user-params.dto.ts
import { ApiProperty } from '@nestjs/swagger';
import { IsUUID } from 'class-validator';

export class UserParamsDto {
  @ApiProperty({ format: 'uuid' })
  @IsUUID('4')
  id: string;
}
```

## Barrel Export

```typescript
// dto/index.ts
export * from './create-user.dto';
export * from './update-user.dto';
export * from './user-response.dto';
export * from './user-filter.dto';
export * from './user-params.dto';
```

## Swagger Property Annotations — Complete Reference

```typescript
// All @ApiProperty options worth knowing:
@ApiProperty({
  description: 'Human readable description shown in Swagger UI',
  example: 'john@example.com',      // shown in example box
  examples: { valid: { value: 'john@example.com' } }, // multiple examples
  type: String,                      // explicit type (auto-detected usually)
  type: () => UserResponseDto,       // lazy ref for circular deps
  isArray: true,                     // marks as array of that type
  enum: UserRole,                    // for enum types
  enumName: 'UserRole',              // avoids name collision in OpenAPI
  format: 'uuid',                    // OpenAPI format hint
  format: 'date-time',
  nullable: true,                    // marks field as nullable
  required: false,                   // marks as optional
  default: UserRole.USER,
  minimum: 1,
  maximum: 100,
  minLength: 1,
  maxLength: 255,
  pattern: '^[a-z]+$',
  deprecated: true,                  // marks field as deprecated
})
```

## Enabling class-transformer Globally

```typescript
// main.ts — critical setup
app.useGlobalPipes(
  new ValidationPipe({
    whitelist: true,               // strip unknown properties
    forbidNonWhitelisted: true,   // throw if unknown props sent
    transform: true,               // auto-transform to DTO types
    transformOptions: {
      enableImplicitConversion: true, // converts "1" -> 1 for @IsInt()
    },
    exceptionFactory: (errors) => {
      // Custom validation error format
      const messages = errors.map((e) => ({
        field: e.property,
        errors: Object.values(e.constraints ?? {}),
      }));
      return new BadRequestException({ statusCode: 400, errors: messages });
    },
  }),
);
```

## Using `plainToInstance` in Services

```typescript
import { plainToInstance } from 'class-transformer';

// Always use this when returning DTOs — respects @Exclude/@Expose
return plainToInstance(UserResponseDto, prismaResult, {
  excludeExtraneousValues: true, // REQUIRED when using @Exclude at class level
});

// For arrays
return plainToInstance(UserResponseDto, prismaResults, {
  excludeExtraneousValues: true,
});
```

## Common Validation Decorators Quick Reference

```typescript
// Strings
@IsString()
@IsNotEmpty()           // not empty string
@MinLength(n)
@MaxLength(n)
@Matches(/pattern/)
@IsEmail()
@IsUrl()
@IsUUID('4')
@IsAlpha()
@IsAlphanumeric()

// Numbers
@IsNumber()
@IsInt()
@Min(n)
@Max(n)
@IsPositive()
@IsNegative()

// Booleans
@IsBoolean()

// Dates
@IsDate()
@IsDateString()         // ISO 8601 string

// Arrays
@IsArray()
@ArrayMinSize(n)
@ArrayMaxSize(n)
@ArrayUnique()

// Enums
@IsEnum(MyEnum)

// Optional / conditional
@IsOptional()           // skip other validators if value is null/undefined
@ValidateIf(o => o.type === 'advanced')  // conditional validation

// Nested
@ValidateNested()
@Type(() => AddressDto) // REQUIRED alongside @ValidateNested

// Custom
@Transform(({ value }) => value.trim())   // transform before validating
@Transform(({ value }) => value?.toLowerCase())
```

## DTO Inheritance Utilities (`@nestjs/mapped-types` / `@nestjs/swagger`)

```typescript
import { PartialType, PickType, OmitType, IntersectionType } from '@nestjs/swagger';
// Note: import from @nestjs/swagger (not @nestjs/mapped-types) to preserve Swagger decorators!

PartialType(CreateUserDto)                         // all fields optional
PickType(CreateUserDto, ['email', 'name'])          // only those fields
OmitType(CreateUserDto, ['password'])               // exclude those fields
IntersectionType(CreateUserDto, AddressDto)         // merge both

// Combine:
export class UpdateUserDto extends PartialType(OmitType(CreateUserDto, ['email'])) {}
```
