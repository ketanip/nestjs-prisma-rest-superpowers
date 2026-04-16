# 04 — Prisma Integration with NestJS

## PrismaService Setup

```typescript
// prisma/prisma.service.ts
import { Injectable, OnModuleInit, OnModuleDestroy, Logger } from '@nestjs/common';
import { PrismaClient } from '@prisma/client';

@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit, OnModuleDestroy {
  private readonly logger = new Logger(PrismaService.name);

  constructor() {
    super({
      log: [
        { emit: 'event', level: 'query' },
        { emit: 'stdout', level: 'error' },
        { emit: 'stdout', level: 'warn' },
      ],
    });
  }

  async onModuleInit() {
    await this.$connect();
    // Log slow queries in development
    if (process.env.NODE_ENV === 'development') {
      (this.$on as any)('query', (e: any) => {
        if (e.duration > 100) {
          this.logger.warn(`Slow query (${e.duration}ms): ${e.query}`);
        }
      });
    }
  }

  async onModuleDestroy() {
    await this.$disconnect();
  }

  // Utility: clean DB in tests
  async cleanDatabase() {
    if (process.env.NODE_ENV !== 'test') {
      throw new Error('cleanDatabase only allowed in test environment');
    }
    const tablenames = await this.$queryRaw<{ tablename: string }[]>`
      SELECT tablename FROM pg_tables WHERE schemaname = 'public'
    `;
    for (const { tablename } of tablenames) {
      if (tablename !== '_prisma_migrations') {
        await this.$executeRawUnsafe(`TRUNCATE TABLE "${tablename}" CASCADE;`);
      }
    }
  }
}
```

```typescript
// prisma/prisma.module.ts
import { Global, Module } from '@nestjs/common';
import { PrismaService } from './prisma.service';

@Global() // makes PrismaService available everywhere without re-importing
@Module({
  providers: [PrismaService],
  exports: [PrismaService],
})
export class PrismaModule {}
```

## Prisma Error Mapping

Map Prisma-specific errors to NestJS HTTP exceptions.

```typescript
// common/filters/prisma-exception.filter.ts
import {
  ArgumentsHost, Catch, ExceptionFilter, HttpStatus, Logger,
} from '@nestjs/common';
import { Prisma } from '@prisma/client';
import { Response } from 'express';

@Catch(Prisma.PrismaClientKnownRequestError)
export class PrismaExceptionFilter implements ExceptionFilter {
  private readonly logger = new Logger(PrismaExceptionFilter.name);

  catch(exception: Prisma.PrismaClientKnownRequestError, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();

    const { statusCode, message } = this.mapError(exception);

    this.logger.error(`Prisma error ${exception.code}: ${exception.message}`);

    response.status(statusCode).json({
      statusCode,
      error: HttpStatus[statusCode],
      message,
      timestamp: new Date().toISOString(),
    });
  }

  private mapError(e: Prisma.PrismaClientKnownRequestError) {
    switch (e.code) {
      case 'P2002': // Unique constraint
        const field = (e.meta?.target as string[])?.join(', ') ?? 'field';
        return { statusCode: 409, message: `A record with this ${field} already exists` };

      case 'P2025': // Record not found (update/delete)
        return { statusCode: 404, message: 'Record not found' };

      case 'P2003': // Foreign key constraint
        return { statusCode: 400, message: 'Referenced record does not exist' };

      case 'P2014': // Required relation violation
        return { statusCode: 400, message: 'Required relation missing' };

      case 'P2016': // Query interpretation error
        return { statusCode: 400, message: 'Invalid query' };

      default:
        return { statusCode: 500, message: 'Database error' };
    }
  }
}
```

Register globally in `main.ts`:
```typescript
app.useGlobalFilters(new PrismaExceptionFilter());
```

## Service Patterns

### Transaction Pattern

```typescript
// Use $transaction for operations that must succeed or fail together
async transferCredits(fromUserId: string, toUserId: string, amount: number) {
  return this.prisma.$transaction(async (tx) => {
    const from = await tx.user.findUnique({ where: { id: fromUserId } });
    if (!from || from.credits < amount) throw new BadRequestException('Insufficient credits');

    await tx.user.update({ where: { id: fromUserId }, data: { credits: { decrement: amount } } });
    await tx.user.update({ where: { id: toUserId },   data: { credits: { increment: amount } } });

    return tx.transaction.create({
      data: { fromUserId, toUserId, amount },
    });
  });
}

// Interactive transactions (when you need to do logic between queries)
// Note: same as above but more explicit about the tx object

// Batch transactions (parallel, atomic)
await this.prisma.$transaction([
  this.prisma.user.update({ where: { id: '1' }, data: { isActive: true } }),
  this.prisma.auditLog.create({ data: { action: 'ACTIVATE', userId: '1' } }),
]);
```

### Upsert Pattern

```typescript
async upsertUserProfile(userId: string, dto: UpsertProfileDto) {
  return this.prisma.userProfile.upsert({
    where: { userId },
    create: { userId, ...dto },
    update: dto,
  });
}
```

### Select Only What You Need

```typescript
// Use select to avoid over-fetching — critical for performance
const users = await this.prisma.user.findMany({
  select: {
    id: true,
    email: true,
    firstName: true,
    lastName: true,
    // NOT password, internalNotes, etc.
  },
  where,
  orderBy,
});
```

### Include Relations Safely

```typescript
// Define include objects as constants — reuse and don't over-nest
export const USER_INCLUDE = {
  address: true,
  tags: { select: { id: true, name: true } },
  _count: { select: { posts: true } },  // aggregation
} satisfies Prisma.UserInclude;

const user = await this.prisma.user.findUnique({
  where: { id },
  include: USER_INCLUDE,
});
```

### Soft Delete Pattern

```typescript
// schema.prisma
model User {
  id        String    @id @default(uuid())
  deletedAt DateTime?
}

// service: always exclude deleted records
async findMany(dto: UserFilterDto) {
  return this.prisma.user.findMany({
    where: {
      deletedAt: null,   // <-- always include this
      ...buildUserFilter(dto).where,
    },
  });
}

async softDelete(id: string) {
  return this.prisma.user.update({
    where: { id },
    data: { deletedAt: new Date() },
  });
}
```

### Prisma Extension — Adding Soft Delete Automatically

```typescript
// prisma/prisma.service.ts — using Prisma extensions
const prismaWithSoftDelete = new PrismaClient().$extends({
  query: {
    user: {
      async findMany({ args, query }) {
        args.where = { ...args.where, deletedAt: null };
        return query(args);
      },
      async findUnique({ args, query }) {
        args.where = { ...args.where, deletedAt: null } as any;
        return query(args);
      },
    },
  },
});
```

### Raw Queries (use sparingly)

```typescript
// For complex analytics only — prefer Prisma query API for everything else
const result = await this.prisma.$queryRaw<{ count: bigint }[]>`
  SELECT COUNT(*) as count FROM "User"
  WHERE "createdAt" > ${new Date(dto.createdAfter)}
`;
// BigInt conversion needed
return Number(result[0].count);
```

## Prisma Schema Best Practices

```prisma
// schema.prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id        String    @id @default(uuid())
  email     String    @unique
  firstName String    @map("first_name")       // camelCase in code, snake_case in DB
  lastName  String    @map("last_name")
  role      UserRole  @default(USER)
  isActive  Boolean   @default(true)           @map("is_active")
  deletedAt DateTime?                          @map("deleted_at")
  createdAt DateTime  @default(now())          @map("created_at")
  updatedAt DateTime  @updatedAt               @map("updated_at")

  // Relations
  address   UserAddress?
  posts     Post[]
  tags      Tag[]        @relation("UserTags")

  @@map("users")                              // snake_case table name
  @@index([email])                            // always index FK and search fields
  @@index([createdAt])
  @@index([role, isActive])                   // composite index for common filters
}

enum UserRole {
  USER
  ADMIN
  MODERATOR
}
```
