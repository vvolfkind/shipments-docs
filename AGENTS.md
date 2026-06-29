# CODING AGENT INSTRUCTIONS (Workspace)

This file is the workspace-level guide for agents working across `shipments-history-api/` and `shipments-history-projector/`.

Per-project copies live at:
- `shipments-history-api/AGENTS.md`
- `shipments-history-projector/AGENTS.md`

Do not modify the per-project files. Use this workspace file as the source of truth when guidance differs or when additional detail is needed.

---

## DO NOT MODIFY

- `<projectRoot>/framework` — never modify in either project
- `shipments-history-api/AGENTS.md` and `shipments-history-projector/AGENTS.md` — leave unchanged

**Runtime (agents):** do not start services or run build/test/lint/orchestrator — see **`prompt.md` § Runtime and MCP**. Status: **`plan.md` § Pending**. Evidence rules: **`plan.md` § Validation evidence**.

---

## DEVELOPMENT STANDARDS

ALWAYS FOLLOW EXISTING PATTERNS. When writing code, adhere to these principles:

1. Prioritize simplicity and readability over clever solutions
2. Start with minimal functionality and verify it works before adding complexity
3. Keep core logic clean and push implementation details to the edges
4. Maintain consistent style (indentation, naming, patterns) throughout the codebase

#### [TYPESCRIPT EXPERTISE]

Use these guidelines before writing, refactoring, or reviewing Typescript code.

1. **Context Awareness**
    - Note that you are a TypeScript expert focusing on declarative and type-safe implementations.
    - Never stop evaluating the rules found in linter configurations. If you have internal mechanics for evaluating context dispersion or degradation, always prioritize linter rules before writing ANY code.
2. **Clarity of Objective**
    - Reiterate the user’s coding task or feature requirements. If a `software_requirements.md` file is provided, that's your guide to accomplish the objective.
    - Type safety and performance are always must-haves.
3. **Assumption Check**
    - Surface any assumptions about the runtime environment, frameworks, or design patterns.
    - Revisit them to see if they hold universally or need verification.
    - ALWAYS prompt the user for severe inconsistencies, if any.
4. **Evidence & Reasoning**
    - Justify your coding approach with references to TypeScript best practices when proposing code that exceeds the current patterns.
    - Mention any library or architectural constraints that might shape or condition your solution.
5. **Alternative Viewpoints**
    - Weigh trade-offs in terms of complexity, performance, or maintainability, but do not act as a biased fundamentalist.
6. **Conclusion or Next Steps**
    - Summarize the final TypeScript solution or snippet, with clear instructions.
    - Suggest tests or reviews to validate correctness and performance.
    - Testing will be implemented in a second phase unless the user declares the opposite (ie: TDD).
    - ALWAYS warn/suggest the user for possible maintainability issues, providing a single paragraph recommendation at, and ONLY AT the end of the successful implementation
7. **Bias Awareness**
    - Identify personal or industry biases.
    - Show openness to changing your approach if new facts or user requirements emerge.

---

## PROJECT

### RULES

`<projectRoot>/framework` MUST NEVER be modified.

These are not standard NestJS apps. Domain code follows a hexagonal modular monolith pattern with a shared frozen `framework/` layer and feature modules under `src/modules/`.

---

## Repository Pattern Structure and Implementation

### Architectural Foundation

Inside each module you'll find a `repositories` directory. All repositories implement the hexagonal architecture pattern with strict layer separation enforced by ESLint boundaries plugin. Each repository extends the base `IRepository<T>` interface from `@framework/domain/repository`, providing mandatory operations: `findById`, `save`, and `update`.

### Directory Structure Requirements

Each repository resides in `src/modules/{moduleName}/repositories/{entityName}/` containing exactly these files:

| File | Purpose |
|------|---------|
| `{entityName}.repository.interface.ts` | Repository contract definition |
| `{entityName}.repository.ts` | Concrete implementation class |
| `{entityName}.repository.mapper.ts` | Data transformation layer |
| `{entityName}.repository.model.ts` | Prisma type definitions |
| `{entityName}.repository.provider.ts` | NestJS dependency injection configuration |

### Repository Interface

- Extends `IRepository<DomainEntity>` from framework
- Uses complete JSDoc documentation with full tag syntax: `@param {Type} name - description` and `@returns {Promise<ReturnType>} description`
- Method signatures return `Promise<DomainEntity | null>` for single entities, `Promise<DomainEntity[]>` for collections
- No use of `any` type — strict TypeScript typing enforced

### Repository Implementation

- Uses constructor injection with `@Inject(ModuleTypes.INFRASTRUCTURE.SERVICE_TOKEN)`
- Injects `PrismaPostgresService` and the corresponding mapper
- All methods marked with explicit return types
- Error handling throws typed Error objects, never generic strings
- Implements business-specific query methods beyond base interface
- **Never maps Prisma rows or builds Prisma payloads directly** — delegate all transformations to the mapper (see Mapper Boundary below)

### Provider Configuration

- Exports array of NestJS providers using symbol tokens from `{module}.types.ts`
- Maps interface tokens to concrete implementation classes
- Includes both repository and mapper in provider array

---

## Mapper Boundary (Critical)

The mapper is the **only** allowed boundary between Prisma persistence and domain/application code. This applies to every entity, on every read and every write.

### Core Rule

```
Prisma  ←→  Mapper  ←→  Domain Entity / Repository Read Model
```

- Use cases, controllers, and notification handlers must never import Prisma types or construct Prisma payloads.
- Repositories must never inline field mapping or build Prisma `data` objects by hand.
- If data crosses the persistence boundary, it goes through the mapper.

### What the Mapper Owns

The mapper class (`@Injectable()`) is responsible for **all** transformations involving Prisma:

| Direction | Typical methods | Used by repository in |
|-----------|-----------------|------------------------|
| Prisma row → domain | `toDomain(model)` | `findById`, `findByExternalId`, any `find*` returning an entity |
| Domain → Prisma create | `toPrismaCreate(entity)` | `save`, inserts |
| Domain → Prisma update | `toPrismaUpdate(entity)` | `update` — do not reuse create mapper for updates |
| Domain → Prisma upsert | `toPrismaCreateOrUpdate(entity)` | upsert flows when applicable |
| Prisma aggregate → read model | `toDetail(model)`, `toListItem(model)`, etc. | complex queries with `include` / joins |

Additional mapper responsibilities:

- Cast JSON fields safely (`metadata as Prisma.InputJsonValue`, `rawPayload as Record<string, unknown>`)
- Normalize nullability with `?? null` or domain defaults
- Apply read-side derivations (example: fallback `courier` from related `primaryShipment` or `primaryPackage`)
- Flatten nested Prisma aggregates (packages, events, milestones, shipments) into repository read models

### What the Repository Owns

The repository orchestrates queries and transactions only:

```typescript
// READ — simple
const record = await this.prisma.package.findUnique({ where: { id } });
return record ? this.mapper.toDomain(record) : null;

// WRITE — create
const record = await this.prisma.package.create({
    data: this.mapper.toPrismaCreate(entity),
});
return this.mapper.toDomain(record);

// WRITE — update
const record = await this.prisma.package.update({
    where: { id: entity.id },
    data: this.mapper.toPrismaUpdate(entity),
});
return this.mapper.toDomain(record);

// READ — composite query
const record = await this.prisma.deliveryHistoryView.findUnique({
    where: { id },
    include: { packages: { include: { package: { include: { packageEvents: true } } } } },
});
return record ? this.mapper.toDetail(record) : null;
```

The repository may define `where`, `include`, `orderBy`, pagination, and transactions — but must pass raw Prisma results to the mapper before returning to callers.

### `.repository.model.ts` Role

This file defines Prisma type aliases used by the mapper and repository. Do not invent ad-hoc shapes in the repository.

```typescript
import { Prisma } from '@prisma/client';

export type PackagePrismaModel = Prisma.PackageGetPayload<Prisma.PackageDefaultArgs>;
export type PackagePrismaCreateInput = Prisma.PackageCreateInput;
export type PackagePrismaUpdateInput = Prisma.PackageUpdateInput;

// For queries with includes:
export type PackageWithEventsPrismaModel = Prisma.PackageGetPayload<{
    include: { packageEvents: true };
}>;
```

When a query uses `include` or `select`, define a matching payload type here and use it in the mapper method signature.

### Anti-Patterns (Do Not Do This)

```typescript
// BAD — inline Prisma payload in repository
await this.prisma.package.create({
    data: {
        externalId: entity.externalId,
        sourceSystem: entity.sourceSystem,
    },
});

// BAD — inline update mapping in repository
await this.prisma.inboxEvent.update({
    where: { id: model.id },
    data: { status: model.status, version: model.version },
});

// BAD — returning Prisma row to use case
return this.prisma.package.findUnique({ where: { id } });

// BAD — use case importing Prisma types
import { Prisma } from '@prisma/client';
```

Correct approach: add or extend mapper methods (`toPrismaCreate`, `toPrismaUpdate`, `toDomain`, `toDetail`) and keep the repository thin.

### Checklist for New Entities

Before a repository is considered complete:

1. `{entity}.repository.model.ts` defines Prisma payload/create/update types
2. `{entity}.repository.mapper.ts` implements at least `toDomain` + `toPrismaCreate`
3. `{entity}.repository.mapper.ts` implements `toPrismaUpdate` if updates exist
4. `{entity}.repository.mapper.ts` implements query-specific mappers (`toDetail`, etc.) for non-trivial reads
5. `{entity}.repository.ts` uses mapper on every code path — no exceptions
6. Mapper and repository are both registered in `{entity}.repository.provider.ts`
7. Mapper token exists in `{module}.types.ts` under `INFRASTRUCTURE`

---

## Read-Side HTTP Mapping (`shipments-history-projector`)

The projector exposes read APIs (list, detail, catalogs). Do **not** add standalone `*.response.mapper.ts` files under `useCases/`. Distribute mapping as follows — same convention as other Nest hexagonal services in this monorepo:

```
Prisma  ←→  repository.mapper  ←→  repository read model
read model  ←→  use case private methods  ←→  OutputDto (Zod contract in dtos/)
```

| Concern | Layer | Examples |
|---------|-------|----------|
| Prisma aggregate → read model | `{entity}.repository.mapper.ts` | `toDetail`, flatten packages/timeline, `volumeCubicCm` from `dimensions`, extract JSON fields |
| Read model → HTTP output | Use case **private methods** | ISO datetimes, `summary` / `statusItems`, contract placeholders (`EMPTY_*`), future catalog labels |
| HTTP contract shape only | `dtos/*.dto.ts` | Zod schemas and types — no transformation logic |

**Rules:**

- Repository mappers **must not** import `dtos/*.dto.ts` or build HTTP response objects.
- Use cases may import Output DTO types and map repository read models in private methods (see `GetShipmentDetailUseCase`, `ListShipmentsUseCase`).
- Catalog label resolution (future i18n) belongs in use case private methods + `ICatalogRepository`, not in view/search repository mappers.
- Shared pure helpers used by multiple use cases may live in `utils/` within the module; prefer private methods until duplication appears.

**Catalog endpoints** (`GetAllCatalogUseCase`, etc.) follow the same split: repository returns `CatalogEntry[]`; use case maps to `CatalogDimensionOutputDto`.

**Projector DB seeds** (`shipments-history-projector`):

| Script | Purpose |
|--------|---------|
| `npm run db:seed:bootstrap` | `app_config` (MOCKS_ENABLED=false) + logistics/operation catalogs |
| `npm run db:seed:delivery-history` | Listable `delivery_history_views` + packages/events (separate; `--count=N`, `--batch-size=N`) |
| `npm run mocks:generate:delivery-history` | Regenerate static HTTP mocks from shared `deliveryHistorySeed.data.ts` |

Shared lifecycle/id formats live in `src/infrastructure/database/seeders/deliveryHistorySeed.data.ts`.

Large seeds: EP body fixed at 9 digits (`EP012933329N` production shape, max `999_999_999` views). Tracking/order/SKU widths scale with `--count`. Default `--batch-size=25`; Prisma tx `timeout` scales with batch (min 60s, max 300s). Cleanup uses `metadata.seeded=true` (no giant `IN` lists).

---

## TypeScript Compliance

`<projectRoot>/eslint.config.mjs` has all the rules; it must be contextualized at all times.

- 4-space indentation, single quotes, 140-character line width
- PascalCase for classes/interfaces, camelCase for methods/variables
- Explicit function return types mandatory
- NO `any` USAGE — use proper type composition and Prisma generated types
- Import ordering: builtin, external, framework, internal with alphabetical sorting

---

## Dependency Injection

- All tokens defined as symbols in `{module}.types.ts` under `APPLICATION`, `INFRASTRUCTURE`, and `NOTIFICATIONS` namespaces
- Repositories injected into use cases via interface tokens
- Mappers injected into repositories via mapper symbol tokens
- Use cases injected into controllers and notification handlers via application tokens

---

## Event Ingestion Pattern (Fenix)

When adding queue consumers:

1. DTO + Zod schema for notification payload
2. Use case with repository/mapper stack
3. `BaseNotificationHandler` subclass: `validate()` → `map()` → `useCase.execute()`
4. Provider factory that registers handler on `NotificationHandlerRegistry` at module bootstrap
5. Queue name from env var maps to handler key via `registry.registerHandler(eventName, handler)`

HTTP entry point: `POST /ingestor/fenix-queue/consume` → `FenixQueueConsumer` → registered handler.
