# 05 — Response & Error Standards

## Response Envelope

Every endpoint returns the same shape. No raw objects — always go through the envelope.

### Success (single resource)
```json
{
  "data": { "id": "uuid", "email": "john@example.com", ... },
  "meta": null
}
```

### Success (collection with pagination)
```json
{
  "data": [...],
  "meta": {
    "page": 1, "limit": 20, "total": 142,
    "totalPages": 8, "hasNextPage": true, "hasPrevPage": false
  }
}
```

### Error
```json
{
  "statusCode": 400,
  "error": "Bad Request",
  "message": "Validation failed",
  "errors": [
    { "field": "email", "errors": ["email must be an email"] }
  ],
  "timestamp": "2024-01-15T12:00:00.000Z",
  "path": "/api/v1/users"
}
```

## Response DTOs

```typescript
// common/dto/response.dto.ts
import { ApiProperty, ApiPropertyOptional } from '@nestjs/swagger';

export class ApiResponseDto<T> {
  @ApiProperty()
  data: T;

  @ApiPropertyOptional()
  meta?: PaginationMetaDto | null;
}

export class PaginationMetaDto {
  @ApiProperty() page: number;
  @ApiProperty() limit: number;
  @ApiProperty() total: number;
  @ApiProperty() totalPages: number;
  @ApiProperty() hasNextPage: boolean;
  @ApiProperty() hasPrevPage: boolean;
}

export class ErrorResponseDto {
  @ApiProperty() statusCode: number;
  @ApiProperty() error: string;
  @ApiProperty() message: string;
  @ApiPropertyOptional({ type: [Object] }) errors?: { field: string; errors: string[] }[];
  @ApiProperty() timestamp: string;
  @ApiProperty() path: string;
}
```

## Transform Interceptor (applies envelope globally)

```typescript
// common/interceptors/transform.interceptor.ts
import {
  CallHandler, ExecutionContext, Injectable, NestInterceptor,
} from '@nestjs/common';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

export interface Response<T> {
  data: T;
  meta?: any;
}

@Injectable()
export class TransformInterceptor<T> implements NestInterceptor<T, Response<T>> {
  intercept(_ctx: ExecutionContext, next: CallHandler): Observable<Response<T>> {
    return next.handle().pipe(
      map((result) => {
        // If service returned { data, meta } (paginated), keep it
        if (result && typeof result === 'object' && 'data' in result && 'meta' in result) {
          return result;
        }
        // Otherwise wrap in { data }
        return { data: result, meta: null };
      }),
    );
  }
}
```

Register globally:
```typescript
// main.ts
app.useGlobalInterceptors(new TransformInterceptor());
```

## All-Exception Filter

```typescript
// common/filters/all-exceptions.filter.ts
import {
  ArgumentsHost, Catch, ExceptionFilter, HttpException,
  HttpStatus, Logger,
} from '@nestjs/common';
import { Request, Response } from 'express';

@Catch()
export class AllExceptionsFilter implements ExceptionFilter {
  private readonly logger = new Logger(AllExceptionsFilter.name);

  catch(exception: unknown, host: ArgumentsHost) {
    const ctx      = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request  = ctx.getRequest<Request>();

    let statusCode = HttpStatus.INTERNAL_SERVER_ERROR;
    let message: string | object = 'Internal server error';
    let errors: any[] | undefined;

    if (exception instanceof HttpException) {
      statusCode = exception.getStatus();
      const responseBody = exception.getResponse();

      if (typeof responseBody === 'string') {
        message = responseBody;
      } else if (typeof responseBody === 'object') {
        const body = responseBody as any;
        message = body.message ?? message;
        errors  = body.errors;
      }
    } else if (exception instanceof Error) {
      this.logger.error(exception.stack);
    }

    const body: Record<string, any> = {
      statusCode,
      error: HttpStatus[statusCode] ?? 'Unknown',
      message,
      timestamp: new Date().toISOString(),
      path: request.url,
    };

    if (errors) body.errors = errors;

    response.status(statusCode).json(body);
  }
}
```

Register globally (order matters — AllExceptions last):
```typescript
// main.ts
app.useGlobalFilters(
  new AllExceptionsFilter(),
  new PrismaExceptionFilter(), // more specific, registered after so it overrides
);
```

## HTTP Status Code Standards

| Scenario | Status |
|----------|--------|
| Successful GET / PATCH | 200 |
| Successful POST (created) | 201 |
| Successful DELETE (no body) | 204 |
| Validation failed | 400 |
| Not authenticated | 401 |
| Not authorized (wrong role) | 403 |
| Resource not found | 404 |
| Unique constraint violated | 409 |
| Rate limit exceeded | 429 |
| Server error | 500 |

## NestJS Exception Usage

```typescript
import {
  NotFoundException, BadRequestException, ConflictException,
  ForbiddenException, UnauthorizedException, InternalServerErrorException,
} from '@nestjs/common';

// 404
throw new NotFoundException(`User with id ${id} not found`);

// 400 — structured
throw new BadRequestException({
  message: 'Validation failed',
  errors: [{ field: 'email', errors: ['Email already in use'] }],
});

// 409
throw new ConflictException('Email already registered');

// 403
throw new ForbiddenException('You do not have permission to perform this action');
```

## Custom Business Exception

```typescript
// common/exceptions/business.exception.ts
import { HttpException, HttpStatus } from '@nestjs/common';

export class BusinessException extends HttpException {
  constructor(
    message: string,
    public readonly code: string,   // machine-readable code e.g. 'USER_SUSPENDED'
    statusCode = HttpStatus.UNPROCESSABLE_ENTITY,
  ) {
    super({ message, code, statusCode }, statusCode);
  }
}

// Usage:
throw new BusinessException(
  'User account is suspended',
  'USER_SUSPENDED',
  HttpStatus.FORBIDDEN,
);
```
