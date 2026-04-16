# 03 — Filtering, Sorting & Pagination

## Standard Filter Query DTO

All collection endpoints accept this base DTO, which resources extend.

```typescript
// common/dto/pagination.dto.ts
import { ApiPropertyOptional } from '@nestjs/swagger';
import { IsOptional, IsInt, Min, Max, IsString, IsIn } from 'class-validator';
import { Type, Transform } from 'class-transformer';

export class PaginationDto {
  @ApiPropertyOptional({ default: 1, minimum: 1 })
  @IsOptional()
  @Type(() => Number)
  @IsInt()
  @Min(1)
  page?: number = 1;

  @ApiPropertyOptional({ default: 20, minimum: 1, maximum: 100 })
  @IsOptional()
  @Type(() => Number)
  @IsInt()
  @Min(1)
  @Max(100)
  limit?: number = 20;

  @ApiPropertyOptional({
    description: 'Comma-separated sort fields. Prefix with - for DESC. e.g. -createdAt,firstName',
    example: '-createdAt,firstName',
  })
  @IsOptional()
  @IsString()
  sort?: string;
}
```

## Resource-Level Filter DTO

```typescript
// modules/users/dto/user-filter.dto.ts
import { ApiPropertyOptional } from '@nestjs/swagger';
import {
  IsOptional, IsString, IsEmail, IsEnum, IsBoolean,
  IsDateString, IsUUID, IsArray,
} from 'class-validator';
import { Type, Transform } from 'class-transformer';
import { UserRole } from '@prisma/client';
import { PaginationDto } from '../../../common/dto/pagination.dto';

export class UserFilterDto extends PaginationDto {
  // Simple equality filters
  @ApiPropertyOptional({ example: 'john' })
  @IsOptional()
  @IsString()
  search?: string;               // fulltext across firstName, lastName, email

  @ApiPropertyOptional()
  @IsOptional()
  @IsEmail()
  email?: string;

  @ApiPropertyOptional({ enum: UserRole, isArray: true })
  @IsOptional()
  @IsArray()
  @IsEnum(UserRole, { each: true })
  @Transform(({ value }) => (Array.isArray(value) ? value : [value]))
  role?: UserRole[];

  @ApiPropertyOptional()
  @IsOptional()
  @Type(() => Boolean)
  @IsBoolean()
  isActive?: boolean;

  // Date range filters
  @ApiPropertyOptional({ example: '2024-01-01T00:00:00Z' })
  @IsOptional()
  @IsDateString()
  createdAfter?: string;

  @ApiPropertyOptional({ example: '2024-12-31T23:59:59Z' })
  @IsOptional()
  @IsDateString()
  createdBefore?: string;

  // Relation filter
  @ApiPropertyOptional({ type: [String], format: 'uuid' })
  @IsOptional()
  @IsArray()
  @IsUUID('4', { each: true })
  @Transform(({ value }) => (Array.isArray(value) ? value : [value]))
  tagIds?: string[];
}
```

## Simple Filter Builder

```typescript
// modules/users/users.filter-builder.ts
import { Prisma } from '@prisma/client';
import { UserFilterDto } from './dto/user-filter.dto';
import { parseSortString } from '../../common/utils/sort-parser.util';

export function buildUserFilter(dto: UserFilterDto): {
  where: Prisma.UserWhereInput;
  orderBy: Prisma.UserOrderByWithRelationInput[];
} {
  const where: Prisma.UserWhereInput = {
    // Full-text search across multiple fields
    ...(dto.search && {
      OR: [
        { firstName: { contains: dto.search, mode: 'insensitive' } },
        { lastName:  { contains: dto.search, mode: 'insensitive' } },
        { email:     { contains: dto.search, mode: 'insensitive' } },
      ],
    }),

    // Exact match
    ...(dto.email && { email: dto.email }),

    // IN filter (array)
    ...(dto.role?.length && { role: { in: dto.role } }),

    // Boolean
    ...(dto.isActive !== undefined && { isActive: dto.isActive }),

    // Date range
    ...(dto.createdAfter || dto.createdBefore
      ? {
          createdAt: {
            ...(dto.createdAfter  && { gte: new Date(dto.createdAfter)  }),
            ...(dto.createdBefore && { lte: new Date(dto.createdBefore) }),
          },
        }
      : {}),

    // Relation: user must have ALL of these tags
    ...(dto.tagIds?.length && {
      tags: { some: { id: { in: dto.tagIds } } },
    }),
  };

  const orderBy = parseSortString(dto.sort, {
    allowedFields: ['createdAt', 'updatedAt', 'firstName', 'lastName', 'email'],
    defaultSort: [{ createdAt: 'desc' }],
  });

  return { where, orderBy };
}
```

## Complex / Nested Filter System

For advanced use cases — user-driven `AND`/`OR`/`NOT` logic, like a filter builder UI.

### Query string format accepted

```
GET /api/v1/users?filters[0][field]=role&filters[0][op]=in&filters[0][value]=ADMIN,USER
GET /api/v1/users?filters[0][field]=createdAt&filters[0][op]=gte&filters[0][value]=2024-01-01
GET /api/v1/users?filterLogic=OR
```

Or JSON-encoded for complex nesting:
```
GET /api/v1/users?filter={"AND":[{"field":"role","op":"eq","value":"ADMIN"},{"OR":[{"field":"createdAt","op":"gte","value":"2024-01-01"},{"field":"isActive","op":"eq","value":true}]}]}
```

### Complex Filter DTO

```typescript
// common/dto/complex-filter.dto.ts
import { ApiPropertyOptional } from '@nestjs/swagger';
import { IsOptional, IsString, IsIn, ValidateNested, IsArray } from 'class-validator';
import { Type, Transform } from 'class-transformer';

type FilterOperator = 'eq' | 'neq' | 'gt' | 'gte' | 'lt' | 'lte' | 'in' | 'nin' | 'contains' | 'startsWith' | 'endsWith' | 'isNull';

export class FilterConditionDto {
  @ApiPropertyOptional()
  @IsString()
  field: string;

  @ApiPropertyOptional({ enum: ['eq','neq','gt','gte','lt','lte','in','nin','contains','startsWith','endsWith','isNull'] })
  @IsIn(['eq','neq','gt','gte','lt','lte','in','nin','contains','startsWith','endsWith','isNull'])
  op: FilterOperator;

  @ApiPropertyOptional()
  @IsOptional()
  value?: any;
}

export class ComplexFilterDto {
  @ApiPropertyOptional({ type: [FilterConditionDto] })
  @IsOptional()
  @IsArray()
  @ValidateNested({ each: true })
  @Type(() => FilterConditionDto)
  AND?: FilterConditionDto[];

  @ApiPropertyOptional({ type: [FilterConditionDto] })
  @IsOptional()
  @IsArray()
  @ValidateNested({ each: true })
  @Type(() => FilterConditionDto)
  OR?: FilterConditionDto[];

  @ApiPropertyOptional({ type: [FilterConditionDto] })
  @IsOptional()
  @IsArray()
  @ValidateNested({ each: true })
  @Type(() => FilterConditionDto)
  NOT?: FilterConditionDto[];
}
```

### Universal Prisma Filter Builder

```typescript
// common/utils/prisma-filter-builder.util.ts
import { BadRequestException } from '@nestjs/common';
import { Prisma } from '@prisma/client';

type FilterOp = 'eq'|'neq'|'gt'|'gte'|'lt'|'lte'|'in'|'nin'|'contains'|'startsWith'|'endsWith'|'isNull';

interface FilterCondition {
  field: string;
  op: FilterOp;
  value?: any;
}

interface ComplexFilter {
  AND?: FilterCondition[];
  OR?:  FilterCondition[];
  NOT?: FilterCondition[];
}

/**
 * Converts a single FilterCondition to a Prisma where fragment.
 * allowedFields enforces a whitelist to prevent injection / probing.
 */
function conditionToPrisma(
  condition: FilterCondition,
  allowedFields: string[],
): Record<string, any> {
  const { field, op, value } = condition;

  // Security: only allow explicitly whitelisted fields
  if (!allowedFields.includes(field)) {
    throw new BadRequestException(`Filter field '${field}' is not allowed`);
  }

  // Handle dot-notation for relations: 'address.city' -> { address: { city: ... } }
  if (field.includes('.')) {
    const [relation, ...rest] = field.split('.');
    return {
      [relation]: conditionToPrisma({ field: rest.join('.'), op, value }, [rest.join('.')]),
    };
  }

  switch (op) {
    case 'eq':         return { [field]: { equals: castValue(value) } };
    case 'neq':        return { [field]: { not: castValue(value) } };
    case 'gt':         return { [field]: { gt: castValue(value) } };
    case 'gte':        return { [field]: { gte: castValue(value) } };
    case 'lt':         return { [field]: { lt: castValue(value) } };
    case 'lte':        return { [field]: { lte: castValue(value) } };
    case 'in':         return { [field]: { in: toArray(value).map(castValue) } };
    case 'nin':        return { [field]: { notIn: toArray(value).map(castValue) } };
    case 'contains':   return { [field]: { contains: String(value), mode: 'insensitive' } };
    case 'startsWith': return { [field]: { startsWith: String(value), mode: 'insensitive' } };
    case 'endsWith':   return { [field]: { endsWith: String(value), mode: 'insensitive' } };
    case 'isNull':     return { [field]: value === true ? null : { not: null } };
    default:
      throw new BadRequestException(`Unknown filter operator: ${op}`);
  }
}

function castValue(v: any): any {
  if (v === 'true')  return true;
  if (v === 'false') return false;
  if (v === 'null')  return null;
  if (!isNaN(Number(v)) && v !== '') return Number(v);
  // ISO date strings
  if (/^\d{4}-\d{2}-\d{2}(T.*)?$/.test(v)) return new Date(v);
  return v;
}

function toArray(v: any): any[] {
  if (Array.isArray(v)) return v;
  if (typeof v === 'string') return v.split(',');
  return [v];
}

/**
 * Main entry point.
 * 
 * @param filter  The parsed ComplexFilterDto
 * @param allowed Whitelisted field names (including dot-notation for relations)
 */
export function buildComplexPrismaFilter(
  filter: ComplexFilter,
  allowed: string[],
): Record<string, any> {
  const clauses: Record<string, any>[] = [];

  if (filter.AND?.length) {
    clauses.push({ AND: filter.AND.map((c) => conditionToPrisma(c, allowed)) });
  }
  if (filter.OR?.length) {
    clauses.push({ OR: filter.OR.map((c) => conditionToPrisma(c, allowed)) });
  }
  if (filter.NOT?.length) {
    clauses.push({ NOT: filter.NOT.map((c) => conditionToPrisma(c, allowed)) });
  }

  // If multiple top-level groups, AND them together
  if (clauses.length === 0) return {};
  if (clauses.length === 1) return clauses[0];
  return { AND: clauses };
}
```

### Usage in a Service

```typescript
// In users.service.ts
import { buildComplexPrismaFilter } from '../../common/utils/prisma-filter-builder.util';

const ALLOWED_FILTER_FIELDS = [
  'email', 'firstName', 'lastName', 'role', 'isActive',
  'createdAt', 'updatedAt', 'address.city', 'address.countryCode',
];

async findMany(filterDto: UserFilterDto) {
  const complexWhere = filterDto.filter
    ? buildComplexPrismaFilter(filterDto.filter, ALLOWED_FILTER_FIELDS)
    : {};
  
  const { where: simpleWhere, orderBy } = buildUserFilter(filterDto);
  
  // Merge simple + complex filters
  const where = { AND: [simpleWhere, complexWhere].filter(w => Object.keys(w).length > 0) };
  
  return paginate(this.prisma.user, { where, orderBy }, filterDto);
}
```

## Sort Parser Utility

```typescript
// common/utils/sort-parser.util.ts
import { BadRequestException } from '@nestjs/common';

interface SortOptions {
  allowedFields: string[];
  defaultSort: Record<string, 'asc' | 'desc'>[];
}

/**
 * Parses "-createdAt,firstName" -> [{ createdAt: 'desc' }, { firstName: 'asc' }]
 */
export function parseSortString(
  sortString: string | undefined,
  options: SortOptions,
): Record<string, 'asc' | 'desc'>[] {
  if (!sortString) return options.defaultSort;

  return sortString.split(',').map((part) => {
    const isDesc = part.startsWith('-');
    const field = isDesc ? part.slice(1) : part;

    if (!options.allowedFields.includes(field)) {
      throw new BadRequestException(
        `Sort field '${field}' is not allowed. Allowed: ${options.allowedFields.join(', ')}`,
      );
    }

    return { [field]: isDesc ? 'desc' : 'asc' };
  });
}
```

## Pagination Utility (Offset + Cursor)

### Offset-based (standard)

```typescript
// common/utils/paginate.util.ts
export interface PaginatedResult<T> {
  data: T[];
  meta: {
    page: number;
    limit: number;
    total: number;
    totalPages: number;
    hasNextPage: boolean;
    hasPrevPage: boolean;
  };
}

export async function paginate<T>(
  model: { findMany: (args: any) => Promise<T[]>; count: (args: any) => Promise<number> },
  queryArgs: { where?: any; orderBy?: any; include?: any; select?: any },
  pagination: { page?: number; limit?: number },
): Promise<PaginatedResult<T>> {
  const page  = pagination.page  ?? 1;
  const limit = pagination.limit ?? 20;
  const skip  = (page - 1) * limit;

  const [data, total] = await Promise.all([
    model.findMany({ ...queryArgs, skip, take: limit }),
    model.count({ where: queryArgs.where }),
  ]);

  const totalPages = Math.ceil(total / limit);

  return {
    data,
    meta: {
      page,
      limit,
      total,
      totalPages,
      hasNextPage: page < totalPages,
      hasPrevPage: page > 1,
    },
  };
}
```

### Cursor-based (for infinite scroll / high-volume data)

```typescript
// common/utils/cursor-paginate.util.ts
export interface CursorPaginatedResult<T> {
  data: T[];
  meta: {
    nextCursor: string | null;
    prevCursor: string | null;
    hasNextPage: boolean;
  };
}

export async function cursorPaginate<T extends { id: string }>(
  model: { findMany: (args: any) => Promise<T[]> },
  queryArgs: { where?: any; orderBy?: any; include?: any },
  options: { cursor?: string; limit?: number; direction?: 'forward' | 'backward' },
): Promise<CursorPaginatedResult<T>> {
  const limit = options.limit ?? 20;
  const take = options.direction === 'backward' ? -(limit + 1) : limit + 1;

  const data = await model.findMany({
    ...queryArgs,
    take,
    skip: options.cursor ? 1 : 0,
    cursor: options.cursor ? { id: options.cursor } : undefined,
  });

  const hasNextPage = data.length > limit;
  if (hasNextPage) data.pop();

  return {
    data,
    meta: {
      nextCursor: hasNextPage ? data[data.length - 1]?.id ?? null : null,
      prevCursor: options.cursor ?? null,
      hasNextPage,
    },
  };
}
```

## Response DTO for Paginated Endpoints

```typescript
// common/dto/response.dto.ts
import { ApiProperty } from '@nestjs/swagger';

export class PaginationMetaDto {
  @ApiProperty() page: number;
  @ApiProperty() limit: number;
  @ApiProperty() total: number;
  @ApiProperty() totalPages: number;
  @ApiProperty() hasNextPage: boolean;
  @ApiProperty() hasPrevPage: boolean;
}

export class PaginatedResponseDto<T> {
  @ApiProperty({ isArray: true })
  data: T[];

  @ApiProperty({ type: PaginationMetaDto })
  meta: PaginationMetaDto;
}
```

## Custom Swagger Decorator for Paginated Endpoints

```typescript
// common/decorators/api-paginated-response.decorator.ts
import { applyDecorators, Type } from '@nestjs/common';
import { ApiExtraModels, ApiOkResponse, getSchemaPath } from '@nestjs/swagger';
import { PaginatedResponseDto, PaginationMetaDto } from '../dto/response.dto';

export function ApiPaginatedResponse<T extends Type<unknown>>(model: T) {
  return applyDecorators(
    ApiExtraModels(PaginatedResponseDto, PaginationMetaDto, model),
    ApiOkResponse({
      schema: {
        allOf: [
          {
            properties: {
              data: {
                type: 'array',
                items: { $ref: getSchemaPath(model) },
              },
              meta: { $ref: getSchemaPath(PaginationMetaDto) },
            },
          },
        ],
      },
    }),
  );
}

// Usage in controller:
// @ApiPaginatedResponse(UserResponseDto)
// @Get()
// findAll(@Query() dto: UserFilterDto) { ... }
```
