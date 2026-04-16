# 01 — Project Structure & API Structuring

## Directory Layout

```
src/
├── main.ts                          # Bootstrap
├── app.module.ts                    # Root module
├── common/                          # Shared across all modules
│   ├── decorators/
│   │   ├── api-paginated-response.decorator.ts
│   │   └── current-user.decorator.ts
│   ├── dto/
│   │   ├── pagination.dto.ts
│   │   ├── filter-query.dto.ts
│   │   └── response.dto.ts
│   ├── filters/
│   │   └── prisma-exception.filter.ts
│   ├── guards/
│   │   └── jwt-auth.guard.ts
│   ├── interceptors/
│   │   ├── transform.interceptor.ts
│   │   └── logging.interceptor.ts
│   ├── pipes/
│   │   └── parse-sort.pipe.ts
│   └── utils/
│       ├── prisma-query-builder.util.ts
│       └── filter-parser.util.ts
├── prisma/
│   ├── prisma.module.ts
│   └── prisma.service.ts
└── modules/
    └── users/                       # One folder per resource
        ├── users.module.ts
        ├── users.controller.ts
        ├── users.service.ts
        ├── dto/
        │   ├── create-user.dto.ts
        │   ├── update-user.dto.ts
        │   ├── user-response.dto.ts
        │   ├── user-filter.dto.ts
        │   └── index.ts             # Barrel export
        └── entities/
            └── user.entity.ts       # Optional: mirrors Prisma model
```

## Naming Conventions

| Thing | Convention | Example |
|-------|-----------|---------|
| Module folder | `kebab-case` | `user-profiles/` |
| Class names | `PascalCase` | `UserProfilesController` |
| File names | `kebab-case.type.ts` | `user-profiles.controller.ts` |
| DTO class | `{Verb}{Resource}Dto` | `CreateUserDto`, `UpdateUserDto` |
| Response DTO | `{Resource}ResponseDto` | `UserResponseDto` |
| Filter DTO | `{Resource}FilterDto` | `UserFilterDto` |
| Service method | camelCase verb-noun | `findManyUsers`, `createUser` |
| Controller route | plural noun | `users`, `user-profiles` |

## URL Design Rules

```
GET    /api/v1/users                 # list with filter/sort/page
GET    /api/v1/users/:id             # single resource
POST   /api/v1/users                 # create
PATCH  /api/v1/users/:id             # partial update (prefer PATCH over PUT)
DELETE /api/v1/users/:id             # soft or hard delete

# Nested resources (use sparingly — max 1 level deep)
GET    /api/v1/users/:userId/posts
POST   /api/v1/users/:userId/posts

# Actions that don't map to CRUD — use verb suffix
POST   /api/v1/users/:id/activate
POST   /api/v1/auth/refresh-token

# Bulk operations
POST   /api/v1/users/bulk-create
PATCH  /api/v1/users/bulk-update
DELETE /api/v1/users/bulk-delete
```

**Rules:**
- Always plural nouns for collections (`/users`, not `/user`)
- Never verbs in the path for CRUD (`/getUser` is wrong)
- Use hyphens, not underscores (`/user-profiles`)
- IDs always in path for single-resource ops, never in query string

## Module Anatomy

```typescript
// users.module.ts
import { Module } from '@nestjs/common';
import { UsersController } from './users.controller';
import { UsersService } from './users.service';
import { PrismaModule } from '../../prisma/prisma.module';

@Module({
  imports: [PrismaModule],
  controllers: [UsersController],
  providers: [UsersService],
  exports: [UsersService],  // export if other modules need it
})
export class UsersModule {}
```

## Controller Anatomy

```typescript
// users.controller.ts
import {
  Controller, Get, Post, Patch, Delete, Body, Param, Query,
  ParseUUIDPipe, HttpCode, HttpStatus, UseGuards, Version,
} from '@nestjs/common';
import {
  ApiTags, ApiOperation, ApiResponse, ApiBearerAuth,
  ApiParam, ApiQuery,
} from '@nestjs/swagger';
import { JwtAuthGuard } from '../../common/guards/jwt-auth.guard';
import { UsersService } from './users.service';
import { CreateUserDto, UpdateUserDto, UserFilterDto } from './dto';
import { UserResponseDto } from './dto/user-response.dto';
import { PaginatedResponseDto } from '../../common/dto/response.dto';

@ApiTags('Users')
@ApiBearerAuth()
@UseGuards(JwtAuthGuard)
@Controller({ path: 'users', version: '1' })
export class UsersController {
  constructor(private readonly usersService: UsersService) {}

  @Get()
  @ApiOperation({ summary: 'List users with filtering, sorting, pagination' })
  @ApiResponse({ status: 200, type: PaginatedResponseDto })
  findAll(@Query() filterDto: UserFilterDto): Promise<PaginatedResponseDto<UserResponseDto>> {
    return this.usersService.findMany(filterDto);
  }

  @Get(':id')
  @ApiOperation({ summary: 'Get a single user by ID' })
  @ApiParam({ name: 'id', type: String, format: 'uuid' })
  @ApiResponse({ status: 200, type: UserResponseDto })
  @ApiResponse({ status: 404, description: 'User not found' })
  findOne(@Param('id', ParseUUIDPipe) id: string): Promise<UserResponseDto> {
    return this.usersService.findOneOrThrow(id);
  }

  @Post()
  @HttpCode(HttpStatus.CREATED)
  @ApiOperation({ summary: 'Create a new user' })
  @ApiResponse({ status: 201, type: UserResponseDto })
  create(@Body() createUserDto: CreateUserDto): Promise<UserResponseDto> {
    return this.usersService.create(createUserDto);
  }

  @Patch(':id')
  @ApiOperation({ summary: 'Partially update a user' })
  update(
    @Param('id', ParseUUIDPipe) id: string,
    @Body() updateUserDto: UpdateUserDto,
  ): Promise<UserResponseDto> {
    return this.usersService.update(id, updateUserDto);
  }

  @Delete(':id')
  @HttpCode(HttpStatus.NO_CONTENT)
  @ApiOperation({ summary: 'Delete a user' })
  @ApiResponse({ status: 204, description: 'Deleted successfully' })
  remove(@Param('id', ParseUUIDPipe) id: string): Promise<void> {
    return this.usersService.remove(id);
  }
}
```

## Service Anatomy

```typescript
// users.service.ts
import { Injectable, NotFoundException } from '@nestjs/common';
import { PrismaService } from '../../prisma/prisma.service';
import { CreateUserDto, UpdateUserDto, UserFilterDto } from './dto';
import { UserResponseDto } from './dto/user-response.dto';
import { buildUserFilter } from './users.filter-builder';
import { PaginatedResult, paginate } from '../../common/utils/paginate.util';
import { plainToInstance } from 'class-transformer';

@Injectable()
export class UsersService {
  constructor(private readonly prisma: PrismaService) {}

  async findMany(filterDto: UserFilterDto): Promise<PaginatedResult<UserResponseDto>> {
    const { where, orderBy } = buildUserFilter(filterDto);
    const result = await paginate(this.prisma.user, { where, orderBy }, filterDto);
    return {
      ...result,
      data: plainToInstance(UserResponseDto, result.data),
    };
  }

  async findOneOrThrow(id: string): Promise<UserResponseDto> {
    const user = await this.prisma.user.findUnique({ where: { id } });
    if (!user) throw new NotFoundException(`User ${id} not found`);
    return plainToInstance(UserResponseDto, user);
  }

  async create(dto: CreateUserDto): Promise<UserResponseDto> {
    const user = await this.prisma.user.create({ data: dto });
    return plainToInstance(UserResponseDto, user);
  }

  async update(id: string, dto: UpdateUserDto): Promise<UserResponseDto> {
    await this.findOneOrThrow(id); // throws 404 if not found
    const user = await this.prisma.user.update({ where: { id }, data: dto });
    return plainToInstance(UserResponseDto, user);
  }

  async remove(id: string): Promise<void> {
    await this.findOneOrThrow(id);
    await this.prisma.user.delete({ where: { id } });
  }
}
```
