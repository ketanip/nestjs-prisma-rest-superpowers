# 06 — Dynamic Query Engine

Complete spec for flexible, type-safe, per-resource dynamic filtering with NestJS + Prisma.

---

## Table of Contents
1. [Design Decisions & Rationale](#1-design-decisions--rationale)
2. [Query Format — `?filter={...}`](#2-query-format)
3. [Filter Tree Shape & Nesting Rules](#3-filter-tree-shape--nesting-rules)
4. [Operator Reference](#4-operator-reference)
5. [Relation Filtering](#5-relation-filtering)
6. [Core Types & Interfaces](#6-core-types--interfaces)
7. [Value Coercion Engine](#7-value-coercion-engine)
8. [Complexity Guard](#8-complexity-guard)
9. [Per-Resource Filter Builder Base Class](#9-per-resource-filter-builder-base-class)
10. [Compiling Filter Tree → Prisma WhereInput](#10-compiling-filter-tree--prisma-whereinput)
11. [Sort Parser](#11-sort-parser)
12. [Pagination Utility](#12-pagination-utility)
13. [Query DTO — the single entry point](#13-query-dto--the-single-entry-point)
14. [Wiring into a Controller & Service](#14-wiring-into-a-controller--service)
15. [Complete Resource Example — Users](#15-complete-resource-example--users)
16. [Swagger Documentation Strategy](#16-swagger-documentation-strategy)
17. [Unit Tests](#17-unit-tests)
18. [Error Reference](#18-error-reference)

---

## 1. Design Decisions & Rationale

| Decision | Choice | Why |
|---|---|---|
| Query format | `?filter=<JSON string>` | Universally supported, no bracket-clash, works in every HTTP client, curl-friendly, easy to log/debug |
| Nesting | 2 levels practical + unlimited opt-in via `nested` | 2 levels covers ~99% of real use cases; unlimited available when declared explicitly by resource |
| Field security | Strict per-resource whitelist | No accidental schema exposure, no probing internal fields |
| Invalid fields/ops | Throw `400` with exact path | Fast failure, clear DX, no silent data leaks |
| Relations | Dot-notation + `some`/`every`/`none` | Dot-notation for scalar fields on related records; list modifiers for one-to-many without over-complicating the payload |
| Architecture | Per-resource typed builders | Full TypeScript type safety on allowed fields per model; no magic strings at call sites |
| DB | Prisma-agnostic IR compiled to `WhereInput` | Swap DB engine without touching filter logic |
| Complexity | Configurable per resource | Prevents runaway queries; each resource can declare its own thresholds |
| Pagination | Offset + page | Predictable, simple, covers most use cases |

---

## 2. Query Format

### Sending a filter

All dynamic filters travel in a single `filter` query parameter as a **URL-encoded JSON string**.

```
GET /api/v1/users?filter={"AND":[{"field":"role","op":"eq","value":"ADMIN"}]}&page=1&limit=20&sort=-createdAt
```

Clients should `encodeURIComponent` the JSON value:
```typescript
// Frontend example
const filter = { AND: [{ field: 'role', op: 'eq', value: 'ADMIN' }] };
const url = `/api/v1/users?filter=${encodeURIComponent(JSON.stringify(filter))}`;
```

### Why not bracket notation (`filter[0][field]=...`)?

Bracket notation is ambiguous across frameworks (Express, Fastify parse it differently), collides with array index notation in some proxies, and becomes unreadable for nested logic. JSON string is explicit, self-describing, and works identically everywhere.

### Why not a POST body?

GET semantics are correct for read operations. POST bodies for queries break HTTP caching, break browser history, and break REST conventions. Use GET + JSON query string.

---

## 3. Filter Tree Shape & Nesting Rules

### Base shape

```typescript
// The top-level filter object
{
  "AND": [ ...conditions ],   // all must match
  "OR":  [ ...conditions ],   // at least one must match
  "NOT": [ ...conditions ]    // none must match
}
```

A **condition** is either a **leaf** (actual field comparison) or a **group** (nested AND/OR/NOT):

```typescript
// Leaf condition
{ "field": "firstName", "op": "contains", "value": "john" }

// Group condition (nesting)
{ "AND": [...], "OR": [...], "NOT": [...] }
```

### Practical 2-level example

```json
{
  "AND": [
    { "field": "isActive", "op": "eq", "value": true },
    {
      "OR": [
        { "field": "role", "op": "eq", "value": "ADMIN" },
        { "field": "role", "op": "eq", "value": "MODERATOR" }
      ]
    }
  ]
}
```

Reads as: **isActive = true AND (role = ADMIN OR role = MODERATOR)**

### Unlimited nesting (opt-in per resource)

Resources can opt into unlimited recursion by setting `maxDepth: Infinity` in their builder config. By default, depth is capped at **3** (root + 2 nested levels). Beyond that, a `400` is thrown.

### Multiple top-level keys

```json
{
  "AND": [{ "field": "isActive", "op": "eq", "value": true }],
  "NOT": [{ "field": "role",     "op": "eq", "value": "BANNED" }]
}
```

Multiple top-level keys are merged with implicit AND.

---

## 4. Operator Reference

| Operator | Meaning | Value type | DB-agnostic? |
|---|---|---|---|
| `eq` | Equals | any scalar | ✅ |
| `neq` | Not equals | any scalar | ✅ |
| `gt` | Greater than | number \| date | ✅ |
| `gte` | Greater than or equal | number \| date | ✅ |
| `lt` | Less than | number \| date | ✅ |
| `lte` | Less than or equal | number \| date | ✅ |
| `in` | Value in array | array | ✅ |
| `nin` | Value not in array | array | ✅ |
| `contains` | Case-insensitive substring | string | ⚠️ Postgres: `mode: insensitive`; MySQL: collation-dependent |
| `startsWith` | Case-insensitive prefix | string | ⚠️ Same as above |
| `endsWith` | Case-insensitive suffix | string | ⚠️ Same as above |
| `isNull` | Field is null | boolean (`true`=null, `false`=not null) | ✅ |
| `some` | Related list has ≥1 matching record | nested filter group | ✅ |
| `every` | All related records match | nested filter group | ✅ |
| `none` | No related records match | nested filter group | ✅ |

---

## 5. Relation Filtering

### Dot-notation — scalar fields on related records

Filter by a field on a directly related (belongs-to) record:

```json
{ "field": "address.city", "op": "eq", "value": "New York" }
{ "field": "address.countryCode", "op": "in", "value": ["US", "CA"] }
```

The resource's whitelist explicitly declares which dot-paths are allowed:
```typescript
allowedFields: ['email', 'firstName', 'address.city', 'address.countryCode']
```

Dot-paths are resolved recursively into nested Prisma `where` objects:
```
address.city = 'NY'  →  { address: { city: { equals: 'NY' } } }
```

### List modifiers — `some` / `every` / `none`

For one-to-many relations. The `value` is a **nested filter group**, not a scalar:

```json
{
  "field": "posts",
  "op": "some",
  "value": {
    "AND": [
      { "field": "published", "op": "eq", "value": true },
      { "field": "views",     "op": "gte", "value": 100 }
    ]
  }
}
```

Reads as: **user has at least one published post with ≥100 views**

The resource whitelist declares relations and their allowed sub-fields:
```typescript
allowedRelations: {
  posts: {
    modifier: ['some', 'every', 'none'],
    allowedFields: ['published', 'views', 'createdAt'],
  }
}
```

### What's intentionally NOT supported

- Aggregates in `WHERE` (`_count > 5`) — use a raw query or a computed column for this; it bypasses Prisma's type system
- Cross-relation joins deeper than declared in `allowedRelations` — prevents N+1 and schema probing
- `OR` across different relations in the same condition — too ambiguous; model it as separate `OR` groups at the top level

---

## 6. Core Types & Interfaces

```typescript
// common/query-engine/types.ts

export type FilterOperator =
  | 'eq' | 'neq'
  | 'gt' | 'gte' | 'lt' | 'lte'
  | 'in' | 'nin'
  | 'contains' | 'startsWith' | 'endsWith'
  | 'isNull'
  | 'some' | 'every' | 'none';

export type FieldType = 'string' | 'number' | 'boolean' | 'date' | 'enum';

// A leaf condition — compares one field to one value
export interface FilterLeaf {
  field: string;
  op: FilterOperator;
  value?: unknown;
}

// A group — contains leaves or nested groups
export interface FilterGroup {
  AND?: FilterNode[];
  OR?:  FilterNode[];
  NOT?: FilterNode[];
}

// A node is either a leaf or a group
export type FilterNode = FilterLeaf | FilterGroup;

// The top-level filter — always a group
export type FilterTree = FilterGroup;

// Describes one filterable field on a resource
export interface FieldSchema {
  type: FieldType;
  enumValues?: string[];           // required when type = 'enum'
  allowedOperators?: FilterOperator[]; // defaults to sensible set per type
  nullable?: boolean;
}

// Describes a filterable relation on a resource
export interface RelationSchema {
  modifiers: Array<'some' | 'every' | 'none'>;
  allowedFields: Record<string, FieldSchema>;  // fields on the related model
}

// Full config passed to a resource's filter builder
export interface FilterBuilderConfig {
  allowedFields: Record<string, FieldSchema>;
  allowedRelations?: Record<string, RelationSchema>;
  complexity?: {
    maxDepth?: number;        // default: 3
    maxConditions?: number;   // default: 50
  };
}
```

---

## 7. Value Coercion Engine

Query strings deliver everything as strings. This module casts values to the correct type before they reach Prisma, and validates them against the declared field schema.

```typescript
// common/query-engine/coerce.ts
import { BadRequestException } from '@nestjs/common';
import { FieldSchema, FilterOperator } from './types';

export function coerceValue(
  raw: unknown,
  schema: FieldSchema,
  field: string,
  op: FilterOperator,
): unknown {
  // List operators — coerce each element
  if (op === 'in' || op === 'nin') {
    const arr = toRawArray(raw, field);
    return arr.map((item) => coerceSingle(item, schema, field));
  }

  // isNull — must be boolean
  if (op === 'isNull') {
    if (raw === true  || raw === 'true')  return true;
    if (raw === false || raw === 'false') return false;
    throw new BadRequestException(
      `Filter on '${field}': operator 'isNull' requires a boolean value, got '${raw}'`,
    );
  }

  return coerceSingle(raw, schema, field);
}

function coerceSingle(raw: unknown, schema: FieldSchema, field: string): unknown {
  if (raw === null || raw === 'null') {
    if (!schema.nullable) {
      throw new BadRequestException(`Filter on '${field}': field is not nullable`);
    }
    return null;
  }

  switch (schema.type) {
    case 'string':
      return String(raw);

    case 'number': {
      const n = Number(raw);
      if (isNaN(n)) {
        throw new BadRequestException(
          `Filter on '${field}': expected a number, got '${raw}'`,
        );
      }
      return n;
    }

    case 'boolean': {
      if (raw === true  || raw === 'true')  return true;
      if (raw === false || raw === 'false') return false;
      throw new BadRequestException(
        `Filter on '${field}': expected a boolean (true/false), got '${raw}'`,
      );
    }

    case 'date': {
      const d = new Date(raw as string);
      if (isNaN(d.getTime())) {
        throw new BadRequestException(
          `Filter on '${field}': expected an ISO 8601 date string, got '${raw}'`,
        );
      }
      return d;
    }

    case 'enum': {
      const str = String(raw);
      if (!schema.enumValues?.includes(str)) {
        throw new BadRequestException(
          `Filter on '${field}': '${str}' is not a valid value. Allowed: ${schema.enumValues?.join(', ')}`,
        );
      }
      return str;
    }

    default:
      return raw;
  }
}

function toRawArray(value: unknown, field: string): unknown[] {
  if (Array.isArray(value)) return value;
  if (typeof value === 'string') return value.split(',');
  throw new BadRequestException(
    `Filter on '${field}': operators 'in'/'nin' require an array or comma-separated string`,
  );
}

// Default allowed operators per field type
export const DEFAULT_OPERATORS_BY_TYPE: Record<string, FilterOperator[]> = {
  string:  ['eq', 'neq', 'in', 'nin', 'contains', 'startsWith', 'endsWith', 'isNull'],
  number:  ['eq', 'neq', 'gt', 'gte', 'lt', 'lte', 'in', 'nin', 'isNull'],
  boolean: ['eq', 'neq', 'isNull'],
  date:    ['eq', 'neq', 'gt', 'gte', 'lt', 'lte', 'in', 'nin', 'isNull'],
  enum:    ['eq', 'neq', 'in', 'nin', 'isNull'],
};
```

---

## 8. Complexity Guard

Runs before compilation. Throws `400` if the filter tree exceeds the resource's configured limits. Prevents exponential query plans and denial-of-service via deeply nested filters.

```typescript
// common/query-engine/complexity.guard.ts
import { BadRequestException } from '@nestjs/common';
import { FilterTree, FilterNode, FilterLeaf, FilterGroup } from './types';

export interface ComplexityConfig {
  maxDepth: number;       // default 3
  maxConditions: number;  // default 50
}

export const DEFAULT_COMPLEXITY: ComplexityConfig = {
  maxDepth: 3,
  maxConditions: 50,
};

export function assertComplexity(
  tree: FilterTree,
  config: ComplexityConfig = DEFAULT_COMPLEXITY,
): void {
  let conditionCount = 0;

  function walk(node: FilterNode, depth: number): void {
    if (depth > config.maxDepth) {
      throw new BadRequestException(
        `Filter exceeds maximum nesting depth of ${config.maxDepth}. ` +
        `Simplify your filter or ask your administrator to increase the limit.`,
      );
    }

    if (isLeaf(node)) {
      conditionCount++;
      if (conditionCount > config.maxConditions) {
        throw new BadRequestException(
          `Filter exceeds maximum of ${config.maxConditions} conditions per request.`,
        );
      }
      return;
    }

    const group = node as FilterGroup;
    for (const child of [...(group.AND ?? []), ...(group.OR ?? []), ...(group.NOT ?? [])]) {
      walk(child, depth + 1);
    }
  }

  walk(tree, 0);
}

export function isLeaf(node: FilterNode): node is FilterLeaf {
  return 'field' in node;
}
```

---

## 9. Per-Resource Filter Builder Base Class

Each resource extends this class and declares its schema. The base class handles all parsing, validation, coercion, complexity checking, and Prisma compilation.

```typescript
// common/query-engine/filter-builder.base.ts
import { BadRequestException } from '@nestjs/common';
import {
  FilterBuilderConfig, FilterTree, FilterNode, FilterLeaf,
  FilterGroup, FilterOperator, RelationSchema,
} from './types';
import { coerceValue, DEFAULT_OPERATORS_BY_TYPE } from './coerce';
import { assertComplexity, DEFAULT_COMPLEXITY, isLeaf } from './complexity.guard';

export abstract class FilterBuilder {
  protected abstract readonly config: FilterBuilderConfig;

  /**
   * Main entry point.
   * Parses the raw ?filter= string, validates, and compiles to Prisma WhereInput.
   */
  build(rawFilter: string | undefined): Record<string, unknown> {
    if (!rawFilter) return {};

    const tree = this.parse(rawFilter);
    assertComplexity(tree, {
      maxDepth:      this.config.complexity?.maxDepth      ?? DEFAULT_COMPLEXITY.maxDepth,
      maxConditions: this.config.complexity?.maxConditions ?? DEFAULT_COMPLEXITY.maxConditions,
    });

    return this.compileGroup(tree, 0);
  }

  // ─── Parsing ──────────────────────────────────────────────────────────────

  private parse(raw: string): FilterTree {
    let parsed: unknown;
    try {
      parsed = JSON.parse(raw);
    } catch {
      throw new BadRequestException(
        `Invalid filter: could not parse JSON. Make sure the filter parameter is a valid JSON string.`,
      );
    }

    if (typeof parsed !== 'object' || parsed === null || Array.isArray(parsed)) {
      throw new BadRequestException(
        `Invalid filter: must be a JSON object with AND, OR, or NOT keys.`,
      );
    }

    return parsed as FilterTree;
  }

  // ─── Compilation ──────────────────────────────────────────────────────────

  private compileGroup(group: FilterGroup, depth: number): Record<string, unknown> {
    const clauses: Record<string, unknown>[] = [];

    if (group.AND?.length) {
      clauses.push({ AND: group.AND.map((n) => this.compileNode(n, depth + 1)) });
    }
    if (group.OR?.length) {
      clauses.push({ OR: group.OR.map((n) => this.compileNode(n, depth + 1)) });
    }
    if (group.NOT?.length) {
      clauses.push({ NOT: group.NOT.map((n) => this.compileNode(n, depth + 1)) });
    }

    if (clauses.length === 0) return {};
    if (clauses.length === 1) return clauses[0];
    return { AND: clauses }; // implicit AND when multiple top-level keys
  }

  private compileNode(node: FilterNode, depth: number): Record<string, unknown> {
    if (isLeaf(node)) return this.compileLeaf(node);
    return this.compileGroup(node as FilterGroup, depth);
  }

  private compileLeaf(leaf: FilterLeaf): Record<string, unknown> {
    const { field, op, value } = leaf;

    // ── Relation modifier (some/every/none) ──────────────────────────────────
    if (op === 'some' || op === 'every' || op === 'none') {
      return this.compileRelationModifier(field, op, value);
    }

    // ── Dot-notation relation scalar ─────────────────────────────────────────
    if (field.includes('.')) {
      return this.compileDotNotation(field, op, value);
    }

    // ── Direct field ─────────────────────────────────────────────────────────
    return this.compileDirectField(field, op, value);
  }

  // ─── Direct field compilation ─────────────────────────────────────────────

  private compileDirectField(
    field: string,
    op: FilterOperator,
    value: unknown,
  ): Record<string, unknown> {
    const schema = this.config.allowedFields[field];
    if (!schema) {
      throw new BadRequestException(
        `Filter on unknown field '${field}'. ` +
        `Allowed fields: ${Object.keys(this.config.allowedFields).join(', ')}`,
      );
    }

    const allowedOps = schema.allowedOperators ?? DEFAULT_OPERATORS_BY_TYPE[schema.type] ?? [];
    if (!allowedOps.includes(op)) {
      throw new BadRequestException(
        `Operator '${op}' is not allowed on field '${field}' (type: ${schema.type}). ` +
        `Allowed operators: ${allowedOps.join(', ')}`,
      );
    }

    const coerced = coerceValue(value, schema, field, op);
    return { [field]: this.opToPrisma(op, coerced) };
  }

  // ─── Dot-notation compilation ─────────────────────────────────────────────

  private compileDotNotation(
    dotField: string,
    op: FilterOperator,
    value: unknown,
  ): Record<string, unknown> {
    const parts = dotField.split('.');
    const relationName = parts[0];
    const subField = parts.slice(1).join('.');

    const relationSchema = this.config.allowedRelations?.[relationName];
    if (!relationSchema) {
      throw new BadRequestException(
        `Filter on unknown relation '${relationName}'. ` +
        `Allowed relations: ${Object.keys(this.config.allowedRelations ?? {}).join(', ')}`,
      );
    }

    // Recursively build inner where — supports multi-level dot-notation
    const innerWhere = this.compileDotField(subField, op, value, relationSchema, dotField);
    return { [relationName]: innerWhere };
  }

  private compileDotField(
    field: string,
    op: FilterOperator,
    value: unknown,
    relation: RelationSchema,
    fullPath: string,
  ): Record<string, unknown> {
    if (field.includes('.')) {
      // Another level of nesting — look for the next relation
      throw new BadRequestException(
        `Filter path '${fullPath}' is too deep. Maximum one level of dot-notation per relation.`,
      );
    }

    const schema = relation.allowedFields[field];
    if (!schema) {
      throw new BadRequestException(
        `Filter on unknown relation field '${fullPath}'. ` +
        `Allowed: ${Object.keys(relation.allowedFields)
          .map((f) => `${fullPath.split('.')[0]}.${f}`)
          .join(', ')}`,
      );
    }

    const allowedOps = schema.allowedOperators ?? DEFAULT_OPERATORS_BY_TYPE[schema.type] ?? [];
    if (!allowedOps.includes(op)) {
      throw new BadRequestException(
        `Operator '${op}' is not allowed on field '${fullPath}'. Allowed: ${allowedOps.join(', ')}`,
      );
    }

    const coerced = coerceValue(value, schema, fullPath, op);
    return { [field]: this.opToPrisma(op, coerced) };
  }

  // ─── Relation modifier compilation ───────────────────────────────────────

  private compileRelationModifier(
    field: string,
    op: 'some' | 'every' | 'none',
    value: unknown,
  ): Record<string, unknown> {
    const relationSchema = this.config.allowedRelations?.[field];
    if (!relationSchema) {
      throw new BadRequestException(
        `Relation '${field}' is not filterable. ` +
        `Allowed relations: ${Object.keys(this.config.allowedRelations ?? {}).join(', ')}`,
      );
    }

    if (!relationSchema.modifiers.includes(op)) {
      throw new BadRequestException(
        `Modifier '${op}' is not allowed on relation '${field}'. ` +
        `Allowed modifiers: ${relationSchema.modifiers.join(', ')}`,
      );
    }

    if (typeof value !== 'object' || value === null || Array.isArray(value)) {
      throw new BadRequestException(
        `Filter on '${field}' with operator '${op}': value must be a filter group object ` +
        `(e.g. { "AND": [...] }), got ${JSON.stringify(value)}`,
      );
    }

    // Compile the nested group using only the relation's allowed fields
    const subGroup = value as FilterGroup;
    const subWhere = this.compileSubGroup(subGroup, field, relationSchema);
    return { [field]: { [op]: subWhere } };
  }

  private compileSubGroup(
    group: FilterGroup,
    relationName: string,
    schema: RelationSchema,
  ): Record<string, unknown> {
    const compile = (nodes: FilterNode[]): Record<string, unknown>[] =>
      nodes.map((node) => {
        if (!isLeaf(node)) {
          throw new BadRequestException(
            `Nested filter groups inside relation modifiers are not supported. ` +
            `Keep conditions flat inside '${relationName}' filters.`,
          );
        }
        const leaf = node as FilterLeaf;
        const fieldSchema = schema.allowedFields[leaf.field];
        if (!fieldSchema) {
          throw new BadRequestException(
            `Unknown field '${leaf.field}' in relation '${relationName}'. ` +
            `Allowed: ${Object.keys(schema.allowedFields).join(', ')}`,
          );
        }
        const allowedOps =
          fieldSchema.allowedOperators ?? DEFAULT_OPERATORS_BY_TYPE[fieldSchema.type] ?? [];
        if (!allowedOps.includes(leaf.op)) {
          throw new BadRequestException(
            `Operator '${leaf.op}' not allowed on '${relationName}.${leaf.field}'. ` +
            `Allowed: ${allowedOps.join(', ')}`,
          );
        }
        const coerced = coerceValue(leaf.value, fieldSchema, `${relationName}.${leaf.field}`, leaf.op);
        return { [leaf.field]: this.opToPrisma(leaf.op, coerced) };
      });

    const clauses: Record<string, unknown>[] = [];
    if (group.AND?.length) clauses.push({ AND: compile(group.AND) });
    if (group.OR?.length)  clauses.push({ OR:  compile(group.OR)  });
    if (group.NOT?.length) clauses.push({ NOT: compile(group.NOT) });

    if (clauses.length === 0) return {};
    if (clauses.length === 1) return clauses[0];
    return { AND: clauses };
  }

  // ─── Operator → Prisma mapping ────────────────────────────────────────────

  private opToPrisma(op: FilterOperator, value: unknown): unknown {
    switch (op) {
      case 'eq':         return { equals: value };
      case 'neq':        return { not: value };
      case 'gt':         return { gt: value };
      case 'gte':        return { gte: value };
      case 'lt':         return { lt: value };
      case 'lte':        return { lte: value };
      case 'in':         return { in: value };
      case 'nin':        return { notIn: value };
      // insensitive for Postgres; MySQL uses collation — Prisma handles transparently
      case 'contains':   return { contains: value, mode: 'insensitive' };
      case 'startsWith': return { startsWith: value, mode: 'insensitive' };
      case 'endsWith':   return { endsWith: value, mode: 'insensitive' };
      case 'isNull':     return value === true ? null : { not: null };
      default:
        throw new BadRequestException(`Unknown operator '${op}'`);
    }
  }
}
```

---

## 10. Compiling Filter Tree → Prisma WhereInput

The base class handles compilation. Resources only need to declare the schema. Here's the full flow:

```
?filter=<JSON string>
    │
    ▼
FilterBuilder.build(raw)
    │
    ├─ parse()           — JSON.parse, shape validation
    ├─ assertComplexity()— depth + condition count check
    └─ compileGroup()    — recursive tree walk
           │
           ├─ compileLeaf()
           │     ├─ compileDirectField()     — whitelist + coerce + opToPrisma
           │     ├─ compileDotNotation()     — relation scalar field
           │     └─ compileRelationModifier()— some/every/none + sub-group
           └─ compileGroup() (recursive)
                     │
                     ▼
            Prisma WhereInput object
```

---

## 11. Sort Parser

```typescript
// common/query-engine/sort-parser.ts
import { BadRequestException } from '@nestjs/common';

export type SortDirection = 'asc' | 'desc';
export type SortClause = Record<string, SortDirection>;

export interface SortParserConfig {
  allowedFields: string[];
  defaultSort: SortClause[];
}

/**
 * Parses "-createdAt,firstName" into [{ createdAt: 'desc' }, { firstName: 'asc' }]
 * Prefix with - for DESC. No prefix = ASC.
 * Multiple fields separated by comma.
 */
export function parseSort(
  raw: string | undefined,
  config: SortParserConfig,
): SortClause[] {
  if (!raw) return config.defaultSort;

  return raw.split(',').map((part) => {
    const trimmed  = part.trim();
    const isDesc   = trimmed.startsWith('-');
    const field    = isDesc ? trimmed.slice(1) : trimmed;
    const direction: SortDirection = isDesc ? 'desc' : 'asc';

    if (!config.allowedFields.includes(field)) {
      throw new BadRequestException(
        `Cannot sort by '${field}'. Allowed sort fields: ${config.allowedFields.join(', ')}`,
      );
    }

    return { [field]: direction };
  });
}
```

---

## 12. Pagination Utility

```typescript
// common/query-engine/paginate.ts
export interface PaginationInput {
  page?:  number;
  limit?: number;
}

export interface PaginationMeta {
  page:        number;
  limit:       number;
  total:       number;
  totalPages:  number;
  hasNextPage: boolean;
  hasPrevPage: boolean;
}

export interface PaginatedResult<T> {
  data: T[];
  meta: PaginationMeta;
}

// The minimal Prisma model interface needed for pagination
interface PaginatableModel<T> {
  findMany(args: unknown): Promise<T[]>;
  count(args: unknown): Promise<number>;
}

export async function paginate<T>(
  model: PaginatableModel<T>,
  query: {
    where?:   unknown;
    orderBy?: unknown;
    include?: unknown;
    select?:  unknown;
  },
  pagination: PaginationInput,
): Promise<PaginatedResult<T>> {
  const page  = Math.max(1, pagination.page  ?? 1);
  const limit = Math.min(100, Math.max(1, pagination.limit ?? 20));
  const skip  = (page - 1) * limit;

  const [data, total] = await Promise.all([
    model.findMany({ ...query, skip, take: limit }),
    model.count({ where: query.where }),
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

---

## 13. Query DTO — the single entry point

Every collection endpoint uses this DTO (or extends it). It carries the raw filter string, sort, and pagination — all validated at the NestJS pipe level before reaching the service.

```typescript
// common/query-engine/query.dto.ts
import { ApiPropertyOptional } from '@nestjs/swagger';
import { IsOptional, IsString, IsInt, Min, Max } from 'class-validator';
import { Type } from 'class-transformer';

export class QueryDto {
  @ApiPropertyOptional({
    description:
      'URL-encoded JSON filter. Supports AND/OR/NOT groups with leaf conditions. ' +
      'Example: {"AND":[{"field":"isActive","op":"eq","value":true}]}',
    example: '{"AND":[{"field":"role","op":"eq","value":"ADMIN"}]}',
  })
  @IsOptional()
  @IsString()
  filter?: string;

  @ApiPropertyOptional({
    description:
      'Comma-separated sort fields. Prefix with - for descending. ' +
      'Example: -createdAt,firstName',
    example: '-createdAt',
  })
  @IsOptional()
  @IsString()
  sort?: string;

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
}
```

---

## 14. Wiring into a Controller & Service

### Controller

```typescript
@Get()
@ApiOperation({ summary: 'List users with dynamic filtering, sorting, pagination' })
@ApiPaginatedResponse(UserResponseDto)
findAll(@Query() query: QueryDto): Promise<PaginatedResult<UserResponseDto>> {
  return this.usersService.findMany(query);
}
```

### Service

```typescript
@Injectable()
export class UsersService {
  private readonly filterBuilder = new UserFilterBuilder();

  constructor(private readonly prisma: PrismaService) {}

  async findMany(query: QueryDto): Promise<PaginatedResult<UserResponseDto>> {
    const where   = this.filterBuilder.build(query.filter);
    const orderBy = parseSort(query.sort, {
      allowedFields: UserFilterBuilder.SORT_FIELDS,
      defaultSort:   [{ createdAt: 'desc' }],
    });

    const result = await paginate(
      this.prisma.user,
      { where, orderBy },
      { page: query.page, limit: query.limit },
    );

    return {
      ...result,
      data: plainToInstance(UserResponseDto, result.data, {
        excludeExtraneousValues: true,
      }),
    };
  }
}
```

---

## 15. Complete Resource Example — Users

```typescript
// modules/users/users.filter-builder.ts
import { FilterBuilder } from '../../common/query-engine/filter-builder.base';
import { FilterBuilderConfig } from '../../common/query-engine/types';
import { UserRole } from '@prisma/client';

export class UserFilterBuilder extends FilterBuilder {
  // Exposed so the service can reuse the same list for sort validation
  static readonly SORT_FIELDS = [
    'createdAt', 'updatedAt', 'firstName', 'lastName', 'email',
  ];

  protected readonly config: FilterBuilderConfig = {
    allowedFields: {
      // Scalar fields
      email:       { type: 'string' },
      firstName:   { type: 'string' },
      lastName:    { type: 'string' },
      isActive:    { type: 'boolean' },
      role:        { type: 'enum', enumValues: Object.values(UserRole) },
      createdAt:   { type: 'date' },
      updatedAt:   { type: 'date' },
      // Nullable example
      deletedAt:   { type: 'date', nullable: true },
    },

    allowedRelations: {
      // Dot-notation scalars: address.city, address.countryCode
      address: {
        modifiers: [],            // no some/every/none — it's a single related record
        allowedFields: {
          city:        { type: 'string' },
          countryCode: { type: 'string', allowedOperators: ['eq', 'in', 'nin'] },
          state:       { type: 'string', nullable: true },
        },
      },

      // List relation — supports some/every/none
      posts: {
        modifiers: ['some', 'none'],
        allowedFields: {
          published: { type: 'boolean' },
          views:     { type: 'number' },
          createdAt: { type: 'date' },
        },
      },

      tags: {
        modifiers: ['some', 'every', 'none'],
        allowedFields: {
          name: { type: 'string' },
          slug: { type: 'string' },
        },
      },
    },

    complexity: {
      maxDepth:      3,
      maxConditions: 30,
    },
  };
}
```

### Example requests this handles

```bash
# Simple equality
?filter={"AND":[{"field":"isActive","op":"eq","value":true}]}

# Enum IN
?filter={"AND":[{"field":"role","op":"in","value":["ADMIN","MODERATOR"]}]}

# Date range
?filter={"AND":[{"field":"createdAt","op":"gte","value":"2024-01-01"},{"field":"createdAt","op":"lte","value":"2024-12-31"}]}

# Nested OR inside AND
?filter={"AND":[{"field":"isActive","op":"eq","value":true},{"OR":[{"field":"role","op":"eq","value":"ADMIN"},{"field":"role","op":"eq","value":"MODERATOR"}]}]}

# Dot-notation relation scalar
?filter={"AND":[{"field":"address.city","op":"eq","value":"New York"}]}

# Relation modifier — has at least one published post
?filter={"AND":[{"field":"posts","op":"some","value":{"AND":[{"field":"published","op":"eq","value":true}]}}]}

# Relation modifier — has no posts with >1000 views (none)
?filter={"AND":[{"field":"posts","op":"none","value":{"AND":[{"field":"views","op":"gt","value":1000}]}}]}

# Combined: active admins in US with at least one published post, sorted by name
?filter={"AND":[{"field":"isActive","op":"eq","value":true},{"field":"role","op":"eq","value":"ADMIN"},{"field":"address.countryCode","op":"eq","value":"US"},{"field":"posts","op":"some","value":{"AND":[{"field":"published","op":"eq","value":true}]}}]}&sort=firstName&page=1&limit=20

# isNull check
?filter={"AND":[{"field":"deletedAt","op":"isNull","value":true}]}

# NOT — exclude banned users
?filter={"NOT":[{"field":"role","op":"eq","value":"BANNED"}]}
```

---

## 16. Swagger Documentation Strategy

The dynamic filter string is complex — document it thoroughly so Swagger users can actually use it without reading source code.

```typescript
// common/query-engine/swagger.decorator.ts
import { applyDecorators } from '@nestjs/common';
import { ApiQuery, ApiExtraModels, ApiOkResponse, getSchemaPath } from '@nestjs/swagger';
import { PaginatedResponseDto, PaginationMetaDto } from '../dto/response.dto';
import { Type } from '@nestjs/common';

// Attach this to any GET list endpoint
export function ApiDynamicQuery<T extends Type<unknown>>(
  model: T,
  options: {
    filterExample?: string;
    sortFields: string[];
  },
) {
  return applyDecorators(
    ApiExtraModels(PaginatedResponseDto, PaginationMetaDto, model),

    ApiQuery({
      name: 'filter',
      required: false,
      schema: { type: 'string' },
      description:
        'URL-encoded JSON filter tree. Supports AND/OR/NOT groups.\n\n' +
        '**Leaf condition:** `{ "field": string, "op": string, "value": any }`\n\n' +
        '**Operators:** `eq` `neq` `gt` `gte` `lt` `lte` `in` `nin` `contains` `startsWith` `endsWith` `isNull` `some` `every` `none`\n\n' +
        '**Example:** `{"AND":[{"field":"isActive","op":"eq","value":true},{"OR":[{"field":"role","op":"eq","value":"ADMIN"}]}]}`',
      example: options.filterExample,
    }),

    ApiQuery({
      name: 'sort',
      required: false,
      schema: { type: 'string' },
      description:
        `Comma-separated sort fields. Prefix with \`-\` for descending.\n\n` +
        `Allowed: \`${options.sortFields.join('`, `')}\`\n\n` +
        `Example: \`-createdAt,firstName\``,
    }),

    ApiQuery({ name: 'page',  required: false, schema: { type: 'integer', default: 1,  minimum: 1 } }),
    ApiQuery({ name: 'limit', required: false, schema: { type: 'integer', default: 20, minimum: 1, maximum: 100 } }),

    ApiOkResponse({
      schema: {
        allOf: [{
          properties: {
            data: { type: 'array', items: { $ref: getSchemaPath(model) } },
            meta: { $ref: getSchemaPath(PaginationMetaDto) },
          },
        }],
      },
    }),
  );
}

// Usage in controller:
// @ApiDynamicQuery(UserResponseDto, {
//   filterExample: '{"AND":[{"field":"role","op":"eq","value":"ADMIN"}]}',
//   sortFields: UserFilterBuilder.SORT_FIELDS,
// })
// @Get()
// findAll(@Query() query: QueryDto) { ... }
```

---

## 17. Unit Tests

```typescript
// common/query-engine/__tests__/filter-builder.spec.ts
import { BadRequestException } from '@nestjs/common';
import { FilterBuilder } from '../filter-builder.base';
import { FilterBuilderConfig } from '../types';

// ── Test resource builder ──────────────────────────────────────────────────

class TestFilterBuilder extends FilterBuilder {
  protected readonly config: FilterBuilderConfig = {
    allowedFields: {
      name:      { type: 'string' },
      age:       { type: 'number' },
      isActive:  { type: 'boolean' },
      role:      { type: 'enum', enumValues: ['ADMIN', 'USER'] },
      createdAt: { type: 'date' },
      deletedAt: { type: 'date', nullable: true },
    },
    allowedRelations: {
      address: {
        modifiers: [],
        allowedFields: {
          city:    { type: 'string' },
          country: { type: 'string', allowedOperators: ['eq', 'in'] },
        },
      },
      posts: {
        modifiers: ['some', 'none'],
        allowedFields: {
          published: { type: 'boolean' },
          views:     { type: 'number' },
        },
      },
    },
    complexity: { maxDepth: 3, maxConditions: 10 },
  };
}

// ── Helpers ────────────────────────────────────────────────────────────────

const builder = new TestFilterBuilder();
const build = (filter: unknown) => builder.build(JSON.stringify(filter));

// ── Tests ──────────────────────────────────────────────────────────────────

describe('FilterBuilder', () => {

  describe('empty / no filter', () => {
    it('returns empty object for undefined', () => {
      expect(builder.build(undefined)).toEqual({});
    });

    it('returns empty object for empty group', () => {
      expect(build({})).toEqual({});
    });
  });

  // ── Basic operators ──────────────────────────────────────────────────────

  describe('direct field operators', () => {
    it('eq — string', () => {
      expect(build({ AND: [{ field: 'name', op: 'eq', value: 'Alice' }] }))
        .toEqual({ AND: [{ name: { equals: 'Alice' } }] });
    });

    it('eq — boolean coercion from string', () => {
      expect(build({ AND: [{ field: 'isActive', op: 'eq', value: 'true' }] }))
        .toEqual({ AND: [{ isActive: { equals: true } }] });
    });

    it('gte — date coercion from string', () => {
      const result = build({ AND: [{ field: 'createdAt', op: 'gte', value: '2024-01-01' }] });
      expect((result as any).AND[0].createdAt.gte).toBeInstanceOf(Date);
    });

    it('in — array of enum values', () => {
      expect(build({ AND: [{ field: 'role', op: 'in', value: ['ADMIN', 'USER'] }] }))
        .toEqual({ AND: [{ role: { in: ['ADMIN', 'USER'] } }] });
    });

    it('in — comma-separated string coerced to array', () => {
      expect(build({ AND: [{ field: 'role', op: 'in', value: 'ADMIN,USER' }] }))
        .toEqual({ AND: [{ role: { in: ['ADMIN', 'USER'] } }] });
    });

    it('contains — adds insensitive mode', () => {
      expect(build({ AND: [{ field: 'name', op: 'contains', value: 'ali' }] }))
        .toEqual({ AND: [{ name: { contains: 'ali', mode: 'insensitive' } }] });
    });

    it('isNull true', () => {
      expect(build({ AND: [{ field: 'deletedAt', op: 'isNull', value: true }] }))
        .toEqual({ AND: [{ deletedAt: null }] });
    });

    it('isNull false', () => {
      expect(build({ AND: [{ field: 'deletedAt', op: 'isNull', value: false }] }))
        .toEqual({ AND: [{ deletedAt: { not: null } }] });
    });

    it('neq', () => {
      expect(build({ AND: [{ field: 'age', op: 'neq', value: 25 }] }))
        .toEqual({ AND: [{ age: { not: 25 } }] });
    });
  });

  // ── Logic groups ─────────────────────────────────────────────────────────

  describe('AND / OR / NOT groups', () => {
    it('AND with two conditions', () => {
      const result = build({
        AND: [
          { field: 'isActive', op: 'eq',  value: true },
          { field: 'role',     op: 'eq',  value: 'ADMIN' },
        ],
      });
      expect(result).toEqual({
        AND: [
          { isActive: { equals: true } },
          { role:     { equals: 'ADMIN' } },
        ],
      });
    });

    it('OR with two conditions', () => {
      const result = build({
        OR: [
          { field: 'role', op: 'eq', value: 'ADMIN' },
          { field: 'role', op: 'eq', value: 'USER' },
        ],
      });
      expect(result).toEqual({
        OR: [
          { role: { equals: 'ADMIN' } },
          { role: { equals: 'USER' } },
        ],
      });
    });

    it('NOT group', () => {
      const result = build({ NOT: [{ field: 'role', op: 'eq', value: 'ADMIN' }] });
      expect(result).toEqual({ NOT: [{ role: { equals: 'ADMIN' } }] });
    });

    it('nested OR inside AND', () => {
      const result = build({
        AND: [
          { field: 'isActive', op: 'eq', value: true },
          { OR: [
            { field: 'role', op: 'eq', value: 'ADMIN' },
            { field: 'role', op: 'eq', value: 'USER'  },
          ]},
        ],
      });
      expect(result).toEqual({
        AND: [
          { isActive: { equals: true } },
          { OR: [{ role: { equals: 'ADMIN' } }, { role: { equals: 'USER' } }] },
        ],
      });
    });

    it('multiple top-level keys merged with AND', () => {
      const result = build({
        AND: [{ field: 'isActive', op: 'eq', value: true }],
        NOT: [{ field: 'role',     op: 'eq', value: 'ADMIN' }],
      });
      expect(result).toEqual({
        AND: [
          { AND: [{ isActive: { equals: true } }] },
          { NOT: [{ role:     { equals: 'ADMIN' } }] },
        ],
      });
    });
  });

  // ── Dot-notation relations ────────────────────────────────────────────────

  describe('dot-notation relation fields', () => {
    it('resolves address.city to nested Prisma where', () => {
      expect(build({ AND: [{ field: 'address.city', op: 'eq', value: 'NYC' }] }))
        .toEqual({ AND: [{ address: { city: { equals: 'NYC' } } }] });
    });

    it('resolves address.country with restricted operators', () => {
      expect(build({ AND: [{ field: 'address.country', op: 'in', value: ['US', 'CA'] }] }))
        .toEqual({ AND: [{ address: { country: { in: ['US', 'CA'] } } }] });
    });

    it('throws 400 for disallowed operator on restricted field', () => {
      expect(() =>
        build({ AND: [{ field: 'address.country', op: 'contains', value: 'U' }] })
      ).toThrow(BadRequestException);
    });

    it('throws 400 for unknown dot-notation field', () => {
      expect(() =>
        build({ AND: [{ field: 'address.zipCode', op: 'eq', value: '10001' }] })
      ).toThrow(BadRequestException);
    });
  });

  // ── Relation modifiers ────────────────────────────────────────────────────

  describe('relation modifiers (some / none)', () => {
    it('some — has at least one matching post', () => {
      const result = build({
        AND: [{
          field: 'posts', op: 'some',
          value: { AND: [{ field: 'published', op: 'eq', value: true }] },
        }],
      });
      expect(result).toEqual({
        AND: [{ posts: { some: { AND: [{ published: { equals: true } }] } } }],
      });
    });

    it('none — has no posts with high views', () => {
      const result = build({
        AND: [{
          field: 'posts', op: 'none',
          value: { AND: [{ field: 'views', op: 'gt', value: 1000 }] },
        }],
      });
      expect(result).toEqual({
        AND: [{ posts: { none: { AND: [{ views: { gt: 1000 } }] } } }],
      });
    });

    it('throws 400 for disallowed modifier on relation', () => {
      expect(() =>
        build({
          AND: [{
            field: 'address', op: 'some', // address has modifiers: []
            value: { AND: [{ field: 'city', op: 'eq', value: 'NYC' }] },
          }],
        })
      ).toThrow(BadRequestException);
    });
  });

  // ── Security / validation ─────────────────────────────────────────────────

  describe('security — field whitelist enforcement', () => {
    it('throws 400 for unknown field', () => {
      expect(() =>
        build({ AND: [{ field: 'password', op: 'eq', value: 'secret' }] })
      ).toThrow(BadRequestException);
    });

    it('throws 400 with message containing the unknown field name', () => {
      try {
        build({ AND: [{ field: 'internalNotes', op: 'eq', value: 'x' }] });
        fail('should have thrown');
      } catch (e: any) {
        expect(e.message).toContain('internalNotes');
      }
    });

    it('throws 400 for invalid enum value', () => {
      expect(() =>
        build({ AND: [{ field: 'role', op: 'eq', value: 'SUPERADMIN' }] })
      ).toThrow(BadRequestException);
    });

    it('throws 400 for operator not allowed on field type', () => {
      expect(() =>
        build({ AND: [{ field: 'isActive', op: 'contains', value: 'tr' }] })
      ).toThrow(BadRequestException);
    });

    it('throws 400 for malformed JSON', () => {
      expect(() => builder.build('{AND:[}')).toThrow(BadRequestException);
    });

    it('throws 400 for non-object JSON', () => {
      expect(() => builder.build('["ADMIN"]')).toThrow(BadRequestException);
    });
  });

  // ── Complexity limits ─────────────────────────────────────────────────────

  describe('complexity guard', () => {
    it('throws 400 when condition count exceeds limit', () => {
      const conditions = Array.from({ length: 11 }, (_, i) => ({
        field: 'age', op: 'gt', value: i,
      }));
      expect(() => build({ AND: conditions })).toThrow(BadRequestException);
    });

    it('throws 400 when nesting depth exceeds limit', () => {
      // Depth: root(0) → AND(1) → OR(2) → AND(3) → AND(4) = exceeds maxDepth:3
      expect(() =>
        build({
          AND: [{
            OR: [{
              AND: [{
                AND: [{ field: 'name', op: 'eq', value: 'x' }],
              }],
            }],
          }],
        })
      ).toThrow(BadRequestException);
    });
  });

  // ── Coercion edge cases ───────────────────────────────────────────────────

  describe('value coercion edge cases', () => {
    it('coerces number string to number', () => {
      expect(build({ AND: [{ field: 'age', op: 'gt', value: '25' }] }))
        .toEqual({ AND: [{ age: { gt: 25 } }] });
    });

    it('throws 400 for non-numeric value on number field', () => {
      expect(() =>
        build({ AND: [{ field: 'age', op: 'gt', value: 'not-a-number' }] })
      ).toThrow(BadRequestException);
    });

    it('throws 400 for non-boolean on isNull operator', () => {
      expect(() =>
        build({ AND: [{ field: 'deletedAt', op: 'isNull', value: 'yes' }] })
      ).toThrow(BadRequestException);
    });

    it('throws 400 for null on non-nullable field', () => {
      expect(() =>
        build({ AND: [{ field: 'name', op: 'eq', value: null }] })
      ).toThrow(BadRequestException);
    });

    it('allows null on nullable field', () => {
      expect(build({ AND: [{ field: 'deletedAt', op: 'eq', value: null }] }))
        .toEqual({ AND: [{ deletedAt: { equals: null } }] });
    });
  });
});
```

---

## 18. Error Reference

| Scenario | HTTP | Message pattern |
|---|---|---|
| Malformed JSON | 400 | `Invalid filter: could not parse JSON...` |
| Non-object JSON | 400 | `Invalid filter: must be a JSON object...` |
| Unknown field | 400 | `Filter on unknown field 'X'. Allowed fields: ...` |
| Unknown relation | 400 | `Filter on unknown relation 'X'. Allowed relations: ...` |
| Unknown relation field | 400 | `Unknown field 'X' in relation 'Y'. Allowed: ...` |
| Disallowed operator for field type | 400 | `Operator 'X' is not allowed on field 'Y'. Allowed: ...` |
| Invalid enum value | 400 | `'X' is not a valid value. Allowed: ...` |
| Non-numeric on number field | 400 | `Expected a number, got 'X'` |
| Non-boolean on boolean field | 400 | `Expected a boolean (true/false), got 'X'` |
| Invalid date string | 400 | `Expected an ISO 8601 date string, got 'X'` |
| Null on non-nullable field | 400 | `Field 'X' is not nullable` |
| Depth exceeded | 400 | `Filter exceeds maximum nesting depth of N...` |
| Condition count exceeded | 400 | `Filter exceeds maximum of N conditions...` |
| `some/every/none` value not an object | 400 | `Value must be a filter group object...` |
| Disallowed modifier on relation | 400 | `Modifier 'X' is not allowed on relation 'Y'. Allowed: ...` |
