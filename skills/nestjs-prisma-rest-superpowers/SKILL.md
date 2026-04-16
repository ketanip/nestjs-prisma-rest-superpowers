---
name: nestjs-rest-api-standards
description: >
  Comprehensive REST API standards for NestJS + Prisma projects. Use this skill whenever the user is:
  - Designing or structuring REST API endpoints in NestJS
  - Setting up filtering, sorting, pagination (including complex/nested filters)
  - Working with DTOs, validation, serialization in NestJS
  - Integrating Prisma with NestJS for data access
  - Setting up Swagger/OpenAPI documentation
  - Handling errors, responses, and API versioning
  - Writing interceptors, guards, pipes for REST APIs
  - Building query builders or filter parsers for APIs
  - Asking about best practices for any REST API concern in a NestJS project
  Trigger this skill even if the user only mentions one of these topics — the full context often helps catch issues early.
---

# NestJS REST API Standards Skill

This skill covers the complete standard for building production-quality REST APIs with NestJS and Prisma. It is organized into focused reference files. **Always read the relevant reference files before generating code** — do not rely on memory alone.

## Reference Files (read as needed)

| File | When to read it |
|------|----------------|
| `references/01-project-structure.md` | API structuring, module layout, naming conventions |
| `references/02-dtos-validation-swagger.md` | DTOs, class-validator, class-transformer, Swagger decorators |
| `references/03-filtering-sorting-pagination.md` | Basic/simple filtering patterns (use 06 for dynamic queries) |
| `references/04-prisma-integration.md` | Prisma service patterns, query building, transactions, error mapping |
| `references/05-response-error-standards.md` | Response envelope, HTTP status codes, error handling, exception filters |
| `references/06-dynamic-query-engine.md` | **Full dynamic query system** — `?filter={}`, operators, relations, nesting, complexity limits, per-resource builders, unit tests |

## Quick Decision Guide

**User asks about…**
- Folder layout, module naming → `01-project-structure.md`
- DTO fields, `@ApiProperty`, `@IsString`, transforms → `02-dtos-validation-swagger.md`
- Simple `?search=`, `?role=`, `?createdAfter=` style filters → `03-filtering-sorting-pagination.md`
- **Dynamic `?filter={...}` with AND/OR/NOT, operators, relations** → `06-dynamic-query-engine.md` ← **this is the main one**
- `PrismaService`, `findMany`, `where` clauses, transactions → `04-prisma-integration.md`
- `ResponseDto`, error codes, exception filters → `05-response-error-standards.md`

## Core Philosophy

1. **Consistency over cleverness** — uniform patterns across every endpoint.
2. **Fail fast at the boundary** — DTOs with class-validator reject bad input before it reaches the service.
3. **Prisma is the single source of truth** — no raw SQL except in migrations.
4. **Every response has the same shape** — success and error responses use the same envelope.
5. **OpenAPI is generated, never hand-written** — every decorator is the source of truth.

## Minimal Working Example (copy-paste bootstrap)

```typescript
// main.ts
import { NestFactory } from '@nestjs/core';
import { ValidationPipe, VersioningType } from '@nestjs/common';
import { SwaggerModule, DocumentBuilder } from '@nestjs/swagger';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Global prefix
  app.setGlobalPrefix('api');

  // URI versioning: /api/v1/...
  app.enableVersioning({ type: VersioningType.URI });

  // Global validation pipe — transforms + whitelist
  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true,
      forbidNonWhitelisted: true,
      transform: true,
      transformOptions: { enableImplicitConversion: true },
    }),
  );

  // Swagger
  const config = new DocumentBuilder()
    .setTitle('My API')
    .setVersion('1.0')
    .addBearerAuth()
    .build();
  const document = SwaggerModule.createDocument(app, config);
  SwaggerModule.setup('api/docs', app, document);

  await app.listen(3000);
}
bootstrap();
```

For deep-dives on any part of this, load the relevant reference file above.
