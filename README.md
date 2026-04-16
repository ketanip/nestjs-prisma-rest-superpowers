# nestjs-prisma-rest-superpowers

> Give any AI agent a production-grade NestJS + Prisma playbook — consistent folder structure, DTOs, filtering, Prisma integration, and response standards, injected into context automatically so you never re-explain conventions again.

Every time you start a new chat, your agent reinvents the wheel — different folder structure, different response shape, different way of handling filters. It's not the agent's fault. It has no memory of what you decided last time.

This skill fixes that. It gives your agent a single authoritative reference it reads before writing a single line of NestJS + Prisma code. Same patterns. Every session. Every teammate. Every agent.

```bash
npx skills add ketanip/nestjs-prisma-rest-superpowers
```

> Works with Claude, Cursor, Copilot, ChatGPT, Windsurf, Cline, Continue — any agent that supports [skills.sh](https://skills.sh).

---

## The before and after

**Before** — you ask your agent to add a users endpoint:

> Generates `UserController` in the root. Puts filters inside the service. Sends back raw Prisma objects. No Swagger. Wrong status codes. Tomorrow you ask again and get something completely different.

**After** — with this skill installed:

> The agent reads the playbook first. Generates the right folder structure, DTOs with Swagger decorators already on every field, a uniform response envelope, and Prisma calls wired correctly. Ask again tomorrow — identical output.

---

## One command. Six battle-tested references injected into your agent.

| # | Reference | What your agent learns |
|---|-----------|------------------------|
| 01 | **Project Structure** | Exact folder layout, naming conventions, barrel exports, module organization |
| 02 | **DTOs & Validation** | `class-validator` + `class-transformer` patterns, Swagger `@ApiProperty` on every field |
| 03 | **Filtering & Pagination** | `?search=`, `?role=`, `?createdAfter=` — query params wired correctly to Prisma |
| 04 | **Prisma Integration** | `PrismaService` patterns, typed `findMany`, transactions, Prisma error mapping |
| 05 | **Response & Error Standards** | Uniform envelope, correct HTTP status codes, global exception filters |
| 06 | **Dynamic Query Engine** | Full `?filter={...}` system — `AND`/`OR`/`NOT`, 12 operators, relation filtering, complexity guards, unit tests |

The agent loads only what your request needs. No bloat, no irrelevant context stuffed into the prompt.

---

## What gets generated — consistently, every time

### The same folder layout. Always.

```
src/
├── common/
│   ├── dto/response.dto.ts
│   ├── filters/prisma-exception.filter.ts
│   └── interceptors/transform.interceptor.ts
├── prisma/
│   └── prisma.service.ts
└── modules/
    └── users/
        ├── users.controller.ts
        ├── users.service.ts
        └── dto/
            ├── create-user.dto.ts
            ├── user-response.dto.ts
            └── user-filter.dto.ts
```

### The same response shape. Success and error.

```json
{
  "success": true,
  "data": { "id": 1, "email": "jane@example.com" },
  "meta": { "page": 1, "limit": 20, "total": 142 }
}
```

### DTOs with Swagger — zero extra work.

```typescript
export class CreateUserDto {
  @ApiProperty({ example: 'jane@example.com' })
  @IsEmail()
  email: string;

  @ApiProperty({ example: 'Jane Doe' })
  @IsString()
  @MinLength(2)
  name: string;
}
```

---

## The part that changes how you build APIs

Reference 06 is a complete **Dynamic Query Engine** — the kind of filtering system that usually takes a senior engineer two days to build correctly.

Your agent generates the whole thing in one pass:

```
GET /api/v1/users
  ?filter={"AND":[{"field":"role","op":"eq","value":"ADMIN"},{"field":"createdAt","op":"gte","value":"2024-01-01"}]}
  &sort=-createdAt
  &page=1
  &limit=20
```

What's included:

- **Logical operators** — `AND`, `OR`, `NOT`, nestable to any depth
- **12 field operators** — `eq`, `neq`, `gt`, `gte`, `lt`, `lte`, `in`, `notIn`, `contains`, `startsWith`, `endsWith`, `isNull`
- **Relation filtering** — dot-notation (`profile.city`) + `some` / `every` / `none` for one-to-many
- **Field whitelists per resource** — no accidental schema exposure
- **Complexity guards** — prevents runaway queries, configurable per resource
- **Unit tests** — generated alongside the implementation, not as an afterthought

One prompt. Complete, production-ready, tested.

---

## Just talk to your agent normally

The skill activates automatically. You don't need to change how you work.

**"Add a GET /users endpoint with filtering by role and pagination"**
→ Agent reads references 01, 02, 03, 04, 05 — generates consistent code.

**"Set up Swagger docs for the orders module"**
→ Agent follows the OpenAPI-first pattern. Decorators as source of truth. No hand-written YAML.

**"I need AND/OR filtering on the products endpoint"**
→ Agent reaches for reference 06 and generates the full dynamic query engine.

You can also be explicit when you want to:

```
"Use the project structure from the skill"
"Follow the response envelope standard"
"Generate dynamic filtering like reference 06"
```

---

## Install in 30 seconds

```bash
# This project only
npx skills add ketanip/nestjs-prisma-rest-superpowers

# Every project on this machine
npx skills add ketanip/nestjs-prisma-rest-superpowers --global

# One specific agent
npx skills add ketanip/nestjs-prisma-rest-superpowers --agent cursor
npx skills add ketanip/nestjs-prisma-rest-superpowers --agent claude

# All agents at once
npx skills add ketanip/nestjs-prisma-rest-superpowers --agent '*'

# See what's inside before committing
npx skills add ketanip/nestjs-prisma-rest-superpowers --list
```

---

## Five principles baked into every reference

1. **Consistency over cleverness** — same patterns across every endpoint, every module, every developer
2. **Fail fast at the boundary** — DTOs reject bad input before it reaches the service layer
3. **Prisma is the single source of truth** — no raw SQL except in migrations
4. **Every response has the same shape** — success and error use the same envelope
5. **OpenAPI is generated, never hand-written** — decorators are the source of truth

---

## Built for

- **AI-assisted teams** where multiple developers use different agents — everyone gets the same conventions
- **Developers new to NestJS** who want production patterns from day one, not after the first rewrite
- **Experienced devs** tired of re-explaining folder structure and response format every new chat session
- **Shops moving fast** who can't afford the tech debt of inconsistent patterns across modules

---

## Requirements

- NestJS v9+
- Prisma v4+
- TypeScript
- `class-validator` + `class-transformer`
- `@nestjs/swagger` *(optional but recommended)*

---

## Skill details

| | |
|---|---|
| **Skill name** | `nestjs-prisma-rest-superpowers` |
| **GitHub** | [ketanip/nestjs-prisma-rest-superpowers](https://github.com/ketanip/nestjs-prisma-rest-superpowers) |
| **Agent compatibility** | Claude, Cursor, Copilot, ChatGPT, Windsurf, Cline, Continue, and any agent that supports skills.sh |
| **Activation** | Automatic on NestJS + Prisma work |
| **Reference files** | 6 focused files, loaded on demand |
| **Dynamic query engine** | Full implementation + unit tests included |
| **Author** | [Ketan Iralepatil](https://github.com/ketanip) |

---

## Feedback & contributions

Found a gap? Have a pattern that belongs here?

Open an issue or PR at [github.com/ketanip/nestjs-prisma-rest-superpowers](https://github.com/ketanip/nestjs-prisma-rest-superpowers) — contributions welcome.
