# Pulse-DRS Comprehensive Project Documentation

## Table of Contents

1. [Project Architecture & Tech Stack](#1-project-architecture--tech-stack)
2. [File Structure & Organization](#2-file-structure--organization)
3. [Backend Patterns](#3-backend-patterns)
4. [Frontend Patterns](#4-frontend-patterns)
5. [Schema & Type System](#5-schema--type-system)
6. [Query Patterns](#6-query-patterns)
7. [Function Building Patterns](#7-function-building-patterns)
8. [Dependencies Deep Dive](#8-dependencies-deep-dive)
9. [API Contract Pattern](#9-api-contract-pattern)
10. [Database Schema (CDS)](#10-database-schema-cds)
11. [Styling System (Panda CSS)](#11-styling-system-panda-css)
12. [Authentication & Authorization](#12-authentication--authorization)
13. [Error Handling](#13-error-handling)
14. [Offline Support](#14-offline-support)
15. [Common Patterns & Best Practices](#15-common-patterns--best-practices)

---

## 1. Project Architecture & Tech Stack

### 1.1 Monorepo Structure

Pulse-DRS uses **pnpm workspaces** for monorepo management:

```
Pulse-DRS/
├── backend/          # NestJS backend application
├── frontend/         # React 19 frontend application
├── packages/         # Shared packages
│   ├── core/         # Zod schemas and types
│   ├── contract/     # API contracts (ts-rest)
│   ├── eslint/       # ESLint configuration
│   └── prettier/     # Prettier configuration
└── pnpm-workspace.yaml
```

**Key Scripts:**
- `pnpm startapp` - Start both backend and frontend
- `pnpm backend:setup` - Setup SAP CAP database
- `pnpm packages build` - Build all shared packages

### 1.2 Tech Stack Overview

**Backend:**
- **NestJS** - Node.js framework
- **SAP CAP (CDS)** - Database abstraction layer
- **CQRS Pattern** - Command Query Responsibility Segregation
- **ts-rest** - Type-safe REST API contracts
- **Zod** - Schema validation

**Frontend:**
- **React 19** - UI framework
- **TanStack Router** - Type-safe routing
- **TanStack React Query** - Data fetching and caching
- **Dexie** - IndexedDB wrapper for offline-first
- **Panda CSS** - CSS-in-JS styling
- **Ark UI** - Headless UI components

**Shared:**
- **@cctech/core** - Zod schemas and TypeScript types
- **@cctech/contract** - API contract definitions

---

## 2. File Structure & Organization

### 2.1 Backend Structure

Each feature follows a consistent module pattern:

```
backend/src/
├── {feature}/
│   ├── {feature}.module.ts      # NestJS module
│   ├── {feature}.controller.ts   # API controller (ts-rest)
│   ├── commands/                 # Command definitions
│   │   └── {feature}.commands.ts
│   └── handlers/                # Command handlers
│       └── {action}-{feature}.handler.ts
├── repository/                   # Database service
├── shared/                       # Shared utilities
│   └── command-schema.ts        # Command base class
└── app/
    └── app.module.ts             # Root module
```

**Example: RFI Feature**
```
backend/src/rfi/
├── rfi.module.ts
├── rfi.controller.ts
├── commands/
│   └── rfi.commands.ts
└── handlers/
    ├── upsert-rfi.handler.ts
    ├── update-rfi-status.handler.ts
    ├── validate-rfi.handler.ts
    └── reassign-rfi-handler.ts
```

**Example: NC Feature**
```
backend/src/ncs/
├── ncs.module.ts
├── ncs.controller.ts
├── command/
│   └── ncs.command.ts
└── handler/
    ├── create-ncs.handler.ts
    ├── submit-ncs.handler.ts
    ├── open-ncs.handler.ts
    └── update-nc-status.handler.ts
```

**Example: WMC Feature**
```
backend/src/wmc/
├── wmc.module.ts
├── wmc.controller.ts
├── commands/
│   └── wmc.commands.ts
└── handlers/
    ├── create-partial-wmc.handler.ts
    ├── approve-wmc.handler.ts
    ├── reject-wmc.handler.ts
    └── generate-full-wmc.handler.ts
```

### 2.2 Frontend Structure

Frontend follows a feature-based organization:

```
frontend/src/
├── features/                     # Feature components
│   ├── {feature}/
│   │   ├── {feature}-detail.tsx
│   │   ├── {feature}-list.tsx
│   │   └── queries.ts           # Dexie queries
│   └── shared/                  # Shared components
├── router/                      # Route definitions
│   ├── {feature}/
│   │   ├── layout.ts
│   │   └── {feature}-route.ts
│   └── root-route.tsx
├── components/                  # Reusable UI components
│   └── ui/                      # Ark UI components
├── db/                          # Dexie database
│   ├── models.ts                # Database models
│   └── db.tsx                   # Database initialization
├── api/                         # API client setup
└── hooks/                       # Custom hooks
```

**Example: RFI Routes**
```
frontend/src/router/rfi/
├── layout.ts                    # RFI layout route
└── rfi-details-route.tsx       # RFI detail route
```

**Example: NC Routes**
```
frontend/src/router/nc/
├── layout.ts
└── create-nc.ts
```

**Example: WMC Routes**
```
frontend/src/router/wmc/
├── layout.ts
├── home.ts
└── partial-wmc-approved.ts
```

### 2.3 Shared Packages Structure

**@cctech/core** - Schema definitions:
```
packages/core/src/
├── {entity}.ts                  # Zod schemas per entity
├── common.ts                   # Common schemas
└── index.ts                    # Exports
```

**@cctech/contract** - API contracts:
```
packages/contract/src/
├── {feature}.ts                # Contract per feature
├── contract.ts                 # Combined contract
└── index.ts                    # Exports
```

---

## 3. Backend Patterns

### 3.1 CQRS Command Pattern

All commands extend `CommandSchema` from `shared/command-schema.ts`:

```typescript
// backend/src/shared/command-schema.ts
import { z } from "zod/v4"

export const CommandMetaSchema = z.object({
  commandId: z.string().nonempty(),
  user: PersonnelSchema,
})

export type CommandMetaType = z.infer<typeof CommandMetaSchema>

export const CommandSchema = <TSchema extends z.ZodType<any>>(
  _schema: TSchema
) => {
  return class {
    constructor(
      public data: z.infer<TSchema>,
      public meta: CommandMetaType,
    ) {}
  }
}
```

**Example: RFI Commands**
```typescript
// backend/src/rfi/commands/rfi.commands.ts
import {
  UpsertRfiSchema,
  UpdateRfiStatusSchema,
  ReassignRfiSchema,
  ValidateRfiSchema,
} from "@cctech/core"
import { CommandSchema } from "src/shared/command-schema"

export class UpsertRfiCommand extends CommandSchema(UpsertRfiSchema) {}
export class UpdateRfiStatusCommand extends CommandSchema(UpdateRfiStatusSchema) {}
export class ReassignRfiCommand extends CommandSchema(ReassignRfiSchema) {}
export class ValidateRfiCommand extends CommandSchema(ValidateRfiSchema) {}
```

**Example: NC Commands**
```typescript
// backend/src/ncs/command/ncs.command.ts
import { InsertNcsSchema, UpdateNcSchema, SubmitNcSchema } from "@cctech/core"
import { CommandSchema } from "src/shared/command-schema"

export class CreateNcsCommand extends CommandSchema(InsertNcsSchema) {}
export class UpdateNcStatusCommand extends CommandSchema(UpdateNcSchema) {}
export class SubmitNCCommand extends CommandSchema(SubmitNcSchema) {}
export class OpenNCCommand extends CommandSchema(OpenNCSchema) {}
```

**Example: WMC Commands**
```typescript
// backend/src/wmc/commands/wmc.commands.ts
import {
  CreatePartialWmcSchema,
  ApproveWmcSchema,
  RejectWmcSchema,
  GenerateFullWmcSchema,
} from "@cctech/core"
import { CommandSchema } from "src/shared/command-schema"

export class CreatePartialWmcCommand extends CommandSchema(CreatePartialWmcSchema) {}
export class ApproveWmcCommand extends CommandSchema(ApproveWmcSchema) {}
export class RejectWmcCommand extends CommandSchema(RejectWmcSchema) {}
export class GenerateFullWmcCommand extends CommandSchema(GenerateFullWmcSchema) {}
```

### 3.2 Command Handlers

Handlers implement `ICommandHandler<CommandType>` and use `@CommandHandler` decorator:

**Example: RFI Handler**
```typescript
// backend/src/rfi/handlers/upsert-rfi.handler.ts
import { Injectable } from "@nestjs/common"
import { CommandHandler, ICommandHandler } from "@nestjs/cqrs"
import { RepositoryService } from "src/repository"
import { UpsertRfiCommand } from "../commands/rfi.commands"

@Injectable()
@CommandHandler(UpsertRfiCommand)
export class UpsertRfiHandler implements ICommandHandler<UpsertRfiCommand> {
  constructor(
    private readonly repoSvc: RepositoryService,
    private readonly commandBus: CommandBus,
  ) {}

  async execute(command: UpsertRfiCommand) {
    // Validation
    if (!command.meta.user.VENDOR_ID) {
      throw new ForbiddenException("RFI can be created by Contractors only")
    }

    // Business logic
    const { serviceOrder, block } = await this.getServiceOrder(
      command.meta.user.VENDOR_ID,
      command.data.INSPECTION_POINT_ID,
      command.data.UNIT_SCOPES.map((us) => us.DESIGN_ELEMENT_ID),
    )

    // Database operations
    const rfiRes = await this.repoSvc.insert<RfiType>({
      entity: "RFIS",
      values: [
        {
          ID: command.data.ID,
          CONTRACTOR_ID: command.meta.user.ID,
          // ... other fields
        },
      ],
      commandId: command.meta.commandId,
      event: "RFIS.created",
      actorId: command.meta.user.ID,
    })

    return rfiRes[0]
  }
}
```

**Example: NC Handler**
```typescript
// backend/src/ncs/handler/create-ncs.handler.ts
import { Injectable } from "@nestjs/common"
import { CommandHandler, ICommandHandler } from "@nestjs/cqrs"
import { RepositoryService } from "src/repository"
import { CreateNcsCommand } from "../command/ncs.command"

@Injectable()
@CommandHandler(CreateNcsCommand)
export class CreateNcsHandler implements ICommandHandler<CreateNcsCommand> {
  constructor(
    private readonly repoSvc: RepositoryService,
    private readonly emailService: EmailService,
  ) {}

  async execute(command: CreateNcsCommand) {
    // Validation and business logic
    const nc = await this.repoSvc.insert<NcsType>({
      entity: "NCS",
      values: [
        {
          ID: uuidv4(),
          QUALITY_ID: command.meta.user.ID,
          // ... other fields
        },
      ],
      commandId: command.meta.commandId,
      event: "NCS.created",
      actorId: command.meta.user.ID,
    })

    return { nc: nc[0] }
  }
}
```

**Example: Project Handler**
```typescript
// backend/src/projects/handlers/create-project.handler.ts
import { Injectable } from "@nestjs/common"
import { CommandHandler, ICommandHandler } from "@nestjs/cqrs"
import { RepositoryService } from "src/repository"
import { v4 as uuidv4 } from "uuid"
import { CreateProjectCommand } from "../commands/projects.commands"

@Injectable()
@CommandHandler(CreateProjectCommand)
export class CreateProjectHandler implements ICommandHandler<CreateProjectCommand> {
  constructor(private readonly repoSvc: RepositoryService) {}

  async execute(command: CreateProjectCommand) {
    const results = await this.repoSvc.insert<ProjectType>({
      entity: "PROJECTS",
      values: [
        {
          ...command.data,
          ID: uuidv4(),
        },
      ],
      commandId: command.meta.commandId,
      event: "Projects.created",
      actorId: command.meta.user.ID,
    })

    return results[0]
  }
}
```

### 3.3 Repository Service Pattern

`RepositoryService` provides unified database access with automatic audit logging:

```typescript
// backend/src/repository/repository.service.ts
@Injectable()
export class RepositoryService implements OnModuleInit {
  constructor(@InjectDb() private readonly db: CdsService) {}

  // Find with where clause
  async find<T = any>(options: FindOptions): Promise<QueryResult<T>> {
    const entityName = this.getEntityName(options.entity)
    let query = this.db.read(entityName)

    if (options.where) {
      const [key, value] = Object.entries(options.where)[0]
      if (Array.isArray(value)) {
        query = query.where({ [key]: value })
      } else {
        query = query.where(options.where)
      }
    }

    if (options.select) {
      query = query.columns(options.select)
    }

    if (options.orderBy) {
      query = query.orderBy(options.orderBy)
    }

    if (options.limit) {
      query = query.limit(options.limit, options.offset || 0)
    }

    const data = await query
    const mapped = Array.isArray(data) ? data : [data].filter(Boolean)

    return { data: mapped }
  }

  // Find by ID
  async findById<T = any>(entity: EntityType, ID: string): Promise<T | null> {
    const entityName = this.getEntityName(entity)
    const result = await this.db.read(entityName, ID)
    return result || null
  }

  // Insert with audit logging
  async insert<T = any>(options: InsertOptions<T>): Promise<T[]> {
    const entityName = this.getEntityName(options.entity)
    // ... implementation with audit logging
  }

  // Update with audit logging
  async update<T = any>(options: UpdateOptions<T>): Promise<T | null> {
    // ... implementation with audit logging
  }

  // Delete with audit logging
  async delete<T = any>(options: DeleteOptions): Promise<T | null> {
    // ... implementation with audit logging
  }

  private getEntityName(entity: EntityType): string {
    return `CCTECH.DRS.ENTITIES_${entity.toUpperCase()}`
  }
}
```

**Usage Examples:**

```typescript
// Find with simple where
const wmcs = await this.repoSvc.find<WmcType>({
  entity: "WMCS",
  where: { WMC_STATUS: "pending_approval" },
  orderBy: "CREATED_AT desc",
  limit: 10,
})

// Find with array where (IN clause)
const wmcs = await this.repoSvc.find<WmcType>({
  entity: "WMCS",
  where: { ID: { in: wmcIds } },
})

// Find by ID
const rfi = await this.repoSvc.findById<RfiType>("RFIS", rfiId)

// Insert
const result = await this.repoSvc.insert<RfiType>({
  entity: "RFIS",
  values: [
    {
      ID: uuidv4(),
      CONTRACTOR_ID: userId,
      // ... other fields
    },
  ],
  commandId: command.meta.commandId,
  event: "RFIS.created",
  actorId: command.meta.user.ID,
})

// Update
const updated = await this.repoSvc.update<NcsType>({
  entity: "NCS",
  id: ncId,
  values: {
    STATUS: "approved",
    UPDATED_AT: new Date().toISOString(),
  },
  commandId: command.meta.commandId,
  event: "NCS.updated",
  actorId: command.meta.user.ID,
})
```

### 3.4 Controller Pattern (ts-rest)

Controllers use `@TsRestHandler` decorator and return typed responses:

**Example: RFI Controller**
```typescript
// backend/src/rfi/rfi.controller.ts
import { contract } from "@cctech/contract"
import { Controller } from "@nestjs/common"
import { CommandBus } from "@nestjs/cqrs"
import { tsRestHandler, TsRestHandler } from "@ts-rest/nest"
import { RepositoryService } from "src/repository"
import { CommandMeta } from "../auth/auth.decorators"
import { UpsertRfiCommand, UpdateRfiStatusCommand } from "./commands/rfi.commands"

@Controller()
export class RfiController {
  constructor(
    private readonly commandBus: CommandBus,
    private readonly repoSvc: RepositoryService,
  ) {}

  @TsRestHandler(contract.rfis)
  async rfi(@CommandMeta() meta: CommandMetaType) {
    return tsRestHandler(contract.rfis, {
      upsert: async ({ body }) => {
        const validation = await this.commandBus.execute(
          new ValidateRfiCommand(body, meta)
        )

        if (validation.errors?.length > 0) {
          return {
            status: 400,
            body: validation,
          }
        }

        const res = await this.commandBus.execute(
          new UpsertRfiCommand(body, meta)
        )

        return {
          status: 201,
          body: res,
        }
      },
      getAll: async () => {
        const res = await this.repoSvc.find<RfiType>({
          entity: "RFIS",
        })

        return {
          status: 200,
          body: res,
        }
      },
      updateRFIStatus: async ({ body }) => {
        const res = await this.commandBus.execute(
          new UpdateRfiStatusCommand(body, meta)
        )
        return {
          status: 200,
          body: res,
        }
      },
    })
  }
}
```

**Example: NC Controller**
```typescript
// backend/src/ncs/ncs.controller.ts
@Controller()
export class NcsController {
  constructor(
    private readonly commandBus: CommandBus,
    private readonly repoSvc: RepositoryService,
  ) {}

  @TsRestHandler(contract.ncs)
  async ncs(@CommandMeta() commandMeta: CommandMetaType) {
    return tsRestHandler(contract.ncs, {
      create: async ({ body }) => {
        const result = await this.commandBus.execute(
          new CreateNcsCommand({ ...body }, commandMeta)
        )
        return {
          status: 200,
          body: result,
        }
      },
      getAll: async () => {
        const result = await this.repoSvc.find<NcsType>({
          entity: "NCS",
        })
        return {
          status: 200,
          body: {
            data: result.data,
            events: [],
          },
        }
      },
      updateNcStatus: async ({ body }) => {
        const result = await this.commandBus.execute(
          new UpdateNcStatusCommand(body, commandMeta)
        )
        return {
          status: 200,
          body: result,
        }
      },
    })
  }
}
```

**Example: WMC Controller**
```typescript
// backend/src/wmc/wmc.controller.ts
@Controller()
export class WmcController {
  constructor(
    private readonly commandBus: CommandBus,
    private readonly repoSvc: RepositoryService,
  ) {}

  @TsRestHandler(contract.wmc)
  async wmc(@CommandMeta() meta: CommandMetaType) {
    return tsRestHandler(contract.wmc, {
      getAll: async () => {
        const createdWmcs = await this.repoSvc.find<WmcType>({
          entity: "WMCS",
          where: {
            GENERATED_BY_ID: meta.user.ID,
          },
        })

        return {
          status: 200,
          body: {
            WMCS: createdWmcs.data,
            ACTIVITYSCOPES: [],
          },
        }
      },
      createPartial: async ({ body }) => {
        const res = await this.commandBus.execute(
          new CreatePartialWmcCommand(body, meta)
        )
        return {
          status: 200,
          body: res,
        }
      },
      approve: async ({ body }) => {
        const res = await this.commandBus.execute(
          new ApproveWmcCommand(body, meta)
        )
        return {
          status: 200,
          body: res,
        }
      },
    })
  }
}
```

### 3.5 Module Structure

Each feature module imports CqrsModule and registers handlers:

**Example: RFI Module**
```typescript
// backend/src/rfi/rfi.module.ts
import { Module } from "@nestjs/common"
import { CqrsModule } from "@nestjs/cqrs"
import { RfiController } from "./rfi.controller"
import { UpsertRfiHandler } from "./handlers/upsert-rfi.handler"
import { UpdateRFIStatusHandler } from "./handlers/update-rfi-status.handler"
import { ValidateRfiHandler } from "./handlers/validate-rfi.handler"

@Module({
  imports: [CqrsModule, EmailModule],
  controllers: [RfiController],
  providers: [
    ValidateRfiHandler,
    UpsertRfiHandler,
    UpdateRFIStatusHandler,
    ReassignRfiHandler,
  ],
})
export class RfiModule {}
```

**Example: NC Module**
```typescript
// backend/src/ncs/ncs.module.ts
import { Module } from "@nestjs/common"
import { CqrsModule } from "@nestjs/cqrs"
import { NcsController } from "./ncs.controller"
import { CreateNcsHandler } from "./handler/create-ncs.handler"
import { SubmitNCHanlder } from "./handler/submit-ncs.handler"

@Module({
  imports: [CqrsModule],
  controllers: [NcsController],
  providers: [
    CreateNcsHandler,
    SubmitNCHanlder,
    UpdateNcStatusHandler,
    OpenNCHandler,
  ],
})
export class NcsModule {}
```

Modules are imported in `app.module.ts`:
```typescript
// backend/src/app/app.module.ts
@Module({
  imports: [
    RepositoryModule, // Global module
    RfiModule,
    NcsModule,
    WmcModule,
    ProjectsModule,
    // ... other modules
  ],
})
export class AppModule {}
```

---

## 4. Frontend Patterns

### 4.1 Routing (TanStack Router)

Routes are defined using `createRoute()` with lazy loading:

**Example: RFI Route**
```typescript
// frontend/src/router/rfi/rfi-details-route.tsx
import { createRoute, lazyRouteComponent } from "@tanstack/react-router"
import { rfiLayoutRoute } from "./layout"

export const editRfiRoute = createRoute({
  getParentRoute: () => rfiLayoutRoute,
  path: "/$rfiId",
  component: lazyRouteComponent(
    () => import("~/features/rfi/detail/rfi-detail"),
    "RfiDetail"
  ),
  wrapInSuspense: true,
  beforeLoad: async ({ context: { db }, params }) => {
    const rfi = await db.rfis.get(params.rfiId)
    const blockElement = await db.designElements
      .get(rfi?.BLOCK_ID ?? "")
      .then((de) => {
        if (de?.TYPE !== "BLOCK") {
          return db.designElements.get(de?.PARENT_ID ?? "")
        }
        return de
      })
    const plotElement = await db.plots.get(rfi?.PLOT_ID ?? "")

    return {
      breadcrumb: {
        title: rfi
          ? `RFI-${plotElement?.NAME}-${blockElement?.NAME}-${rfi?.RFI_LABEL}`
          : "Create RFI",
      },
      colorPalette: "info",
      previousLink: "/welcome",
    }
  },
  loader: async ({ params, context }) => {
    return {
      rfi: await context.db.rfis.get(params.rfiId),
    }
  },
})
```

**Example: NC Route**
```typescript
// frontend/src/router/nc/create-nc.ts
import { createRoute, lazyRouteComponent } from "@tanstack/react-router"
import { ncLayoutRoute } from "~/router/nc/layout"

export const createNCRoute = createRoute({
  getParentRoute: () => ncLayoutRoute,
  path: "/create/$ncId",
  component: lazyRouteComponent(
    () => import("~/features/nc/create/create"),
    "NcCreate"
  ),
  wrapInSuspense: true,
  beforeLoad: async () => {
    return {
      breadcrumb: {
        title: "Create NC",
        previousLink: "/welcome?type=nc",
      },
      colorPalette: "orange",
    }
  },
  loader: async ({ context }) => {
    const { db } = context
    const plots = await db.plots.toArray()
    return {
      plots,
    }
  },
})
```

**Example: WMC Route**
```typescript
// frontend/src/router/wmc/partial-wmc-approved.ts
import { createRoute, lazyRouteComponent } from "@tanstack/react-router"
import { wmcLayoutRoute } from "./layout"

export const partialWmcApprovedRoute = createRoute({
  getParentRoute: () => wmcLayoutRoute,
  path: "/partial/approved",
  component: lazyRouteComponent(
    () => import("~/features/wmc/partial-wmc-approved"),
    "PartialWmcApproved"
  ),
  wrapInSuspense: true,
  beforeLoad: async () => {
    return {
      breadcrumb: {
        title: "Approved Partial WMCs",
      },
    }
  },
})
```

**Route Context:**
Routes have access to `api`, `queryClient`, and `db` through context:

```typescript
// frontend/src/app.tsx
const context = React.useMemo(
  () => ({
    api,
    queryClient: apiQueryClient,
    db,
  }),
  [api, queryClient]
)

<RouterProvider router={router} context={context} />
```

### 4.2 API Client Pattern

API client is initialized using `initTsrReactQuery`:

```typescript
// frontend/src/app.tsx
import { initTsrReactQuery } from "@ts-rest/react-query/v5"
import { contract } from "@cctech/contract"
import { v4 as uuid } from "uuid"

const api = React.useMemo(() => {
  return initTsrReactQuery(contract, {
    baseUrl: "",
    api: async (args) => {
      args.headers["x-command-id"] = uuid()
      const result = await tsRestFetchApi(args)
      return result
    },
  })
}, [])
```

**Usage in Components:**

```typescript
// Query example
const { data } = api.wmc.getAll.useQuery({
  queryKey: ["wmcs"],
})

// Mutation example
const mutation = api.rfi.upsert.useMutation({
  onSuccess: () => {
    queryClient.invalidateQueries({ queryKey: ["rfis"] })
  },
})

// Usage
mutation.mutate({
  body: {
    ID: rfiId,
    // ... other fields
  },
})
```

**Example: WMC Home Component**
```typescript
// frontend/src/features/wmc/home/route.tsx
import { useRouteContext } from "@tanstack/react-router"
import { wmcHomeRoute } from "../../../router/wmc/home"

export const WmcHome = () => {
  const { api } = useRouteContext({ from: wmcHomeRoute.id })

  const { data } = api.wmc.getAll.useQuery({
    queryKey: ["wmcs"],
  })

  const { data: pendingApprovalsData } = api.wmc.getPendingApproval.useQuery({
    queryKey: ["pending-approvals"],
  })

  const { data: partialWmcsData } = api.wmc.getPartialWmcs.useQuery({
    queryKey: ["partial-wmcs-home"],
  })

  // ... render component
}
```

### 4.3 Database Pattern (Dexie)

Dexie provides IndexedDB wrapper for offline-first functionality:

**Database Initialization:**
```typescript
// frontend/src/db/db.tsx
import Dexie from "dexie"
import { Db } from "./models"

export const useInitDb = () =>
  React.useMemo(() => {
    const instance = new Dexie("PULSE") as Db

    instance.version(1).stores({
      projects: "ID",
      rfis: "ID, ARCHIVED, [PLOT_ID+ARCHIVED], [STATUS+ARCHIVED]",
      nc: "ID, ARCHIVED, [ROOT_ID+ARCHIVED], [STATUS+ARCHIVED]",
      wmc: "ID, WMC_TYPE, WMC_STATUS",
      // ... other stores
    })

    return instance
  }, [])
```

**Live Queries with useLiveQuery:**

```typescript
// frontend/src/features/nc/queries.ts
import { useLiveQuery } from "dexie-react-hooks"

export const useGetNcData = (db: Db, ncId: string) => {
  const nc = useLiveQuery(() => db.nc.get(ncId), [ncId])

  const rootNc = useLiveQuery(
    () => db.nc.get(nc?.ROOT_ID ?? ""),
    [nc?.ROOT_ID]
  )

  const unitScopes = useLiveQuery(
    () =>
      db.unitScopes
        .where("NC_ID")
        .equals(rootNc?.ID ?? "")
        .toArray(),
    [rootNc?.ID],
    []
  )

  const plot = useLiveQuery(
    () =>
      db.plots
        .where("DESIGN_ELEMENT_ID")
        .equals(plotElement?.ID ?? "")
        .toArray()
        .then((ps) => ps[0]),
    [plotElement?.ID]
  )

  return {
    nc,
    rootNc,
    unitScopes,
    plot,
  }
}
```

**Example: RFI Queries**
```typescript
// frontend/src/features/rfi/queries.ts
export const useGetPersonnelFilters = () => {
  const { db } = useRouteContext({ from: layoutRoute.id })
  const values = useStore(form.store, (state) => state.values)

  const activities = useLiveQuery(
    () => listActivitiesByPackageId(values.PACKAGE_ID, db),
    [values.PACKAGE_ID],
    []
  )

  const subActivities = useLiveQuery(
    () => listSubActivitiesByActivityId(values.ACTIVITY_ID, db),
    [values.ACTIVITY_ID],
    []
  )

  const inspectionPoints = useLiveQuery(
    () =>
      db.inspectionPoints
        .where("SUB_ACTIVITY_ID")
        .equals(values.SUB_ACTIVITY_ID)
        .sortBy("SERIAL"),
    [values.SUB_ACTIVITY_ID],
    []
  )

  return {
    activities,
    subActivities,
    inspectionPoints,
  }
}
```

**Example: WAM Design Elements**
```typescript
// frontend/src/features/wam/design-elements.tsx
const plots = useLiveQuery(
  async () => {
    if (canReadAllPlots) {
      return db.designElements.where("TYPE").equals("PLOT").toArray()
    } else {
      const assignments = await db.personnelRegionAssignments
        .where("PERSONNEL_ID")
        .equals(currentUserId)
        .filter((a) => a.STATUS === "ASSIGNED")
        .toArray()

      const plotIds = new Set<string>()
      for (const assignment of assignments) {
        const element = await db.designElements.get(
          assignment.DESIGN_ELEMENT_ID
        )
        if (element?.TYPE === "PLOT") {
          plotIds.add(element.ID)
        }
      }

      return Promise.all(
        Array.from(plotIds).map((id) => db.designElements.get(id))
      )
    }
  },
  [canReadAllPlots, currentUserId],
  []
)
```

### 4.4 Component Patterns

Components use Panda CSS for styling and Ark UI for headless components:

```typescript
// Example component structure
import { Box, HStack, VStack } from "styled-system/jsx"
import { Button } from "~/components/ui/button"
import { useRouteContext } from "@tanstack/react-router"

export const MyComponent = () => {
  const { api, db } = useRouteContext({ from: layoutRoute.id })

  return (
    <VStack gap={4}>
      <Box>Content</Box>
    </VStack>
  )
}
```

---

## 5. Schema & Type System

### 5.1 Zod Schemas (packages/core)

All entities are defined as Zod schemas:

**Example: RFI Schema**
```typescript
// packages/core/src/rfis.ts
import { z } from "zod/v4"
import { DateSchema, IdSchema, TextSchema } from "./common"

export const RfiStatusSchema = z.enum([
  "draft",
  "submitted",
  "rejected",
  "resubmitted",
  "approved",
  "completed",
])

export const RfiSchema = z.object({
  ID: IdSchema,
  CONTRACTOR_ID: IdSchema,
  ENGINEER_ID: IdSchema.optional().nullable(),
  QUALITY_ID: IdSchema.optional().nullable(),
  SERVICE_ORDER_ID: IdSchema,
  INSPECTION_POINT_ID: IdSchema,
  CHECKLIST_ID: IdSchema.optional().nullable(),
  PROJECT_ID: IdSchema,
  VERSION: IntegerSchema,
  PARENT_ID: IdSchema.nullable(),
  ROOT_ID: IdSchema,
  RFI_LABEL: TextSchema.optional(),
  STATUS: RfiStatusSchema,
  CREATED_AT: DateSchema,
  UPDATED_AT: DateSchema,
  CURRENT_HANDLER: RfiHandlerSchema,
  PLOT_ID: IdSchema,
  BLOCK_ID: IdSchema,
  PACKAGE_ID: IdSchema,
})

export type RfiType = z.infer<typeof RfiSchema>
```

**Example: NC Schema**
```typescript
// packages/core/src/ncs.ts
export const NCStatusSchema = z.enum([
  "draft",
  "raised",
  "submitted",
  "rejected",
  "resubmitted",
  "approved",
  "completed",
])

export const NcsSchema = z.object({
  ID: IdSchema,
  QUALITY_ID: IdSchema,
  ENGINEER_ID: IdSchema.optional().nullable(),
  CONTRACTOR_ID: IdSchema.optional().nullable(),
  SERVICE_ORDER_ID: IdSchema.optional().nullable(),
  RFI_CHECKLIST_RESPONSE_ID: IdSchema.optional().nullable(),
  SUBACTIVITY_ID: IdSchema,
  PARENT_ID: IdSchema.optional().nullable(),
  ROOT_ID: IdSchema,
  NC_LABEL: TextSchema,
  VERSION: z.number(),
  STATUS: NCStatusSchema,
  CURRENT_HANDLER: RfiHandlerSchema,
  PLOT_ID: IdSchema,
  BLOCK_ID: IdSchema,
  PACKAGE_ID: IdSchema,
})

export type NcsType = z.infer<typeof NcsSchema>
```

**Example: WMC Schema**
```typescript
// packages/core/src/wmc.ts
export const WmcTypeSchema = z.enum(["partial", "full"])
export const WmcStatusSchema = z.enum([
  "draft",
  "pending_approval",
  "approved",
  "rejected",
  "submitted",
  "completed",
])

export const WMCSchema = z.object({
  ID: IdSchema,
  WMC_TYPE: WmcTypeSchema,
  WMC_STATUS: WmcStatusSchema,
  WMC_LABEL: TextSchema,
  PLOT_ID: IdSchema,
  BLOCK_ID: IdSchema,
  SERVICE_ORDER_ID: IdSchema.optional().nullable(),
  GENERATED_BY_ID: IdSchema,
  CREATED_AT: DateSchema,
  UPDATED_AT: DateSchema,
})

export type WmcType = z.infer<typeof WMCSchema>
```

**Common Schemas:**
```typescript
// packages/core/src/common.ts
export const IdSchema = z.string().uuid()
export const TextSchema = z.string()
export const DateSchema = z.string().datetime()
export const IntegerSchema = z.number().int()
export const NumericSchema = z.number()
```

### 5.2 Contract Schemas (packages/contract)

API contracts use ts-rest with schemas from @cctech/core:

**Example: RFI Contract**
```typescript
// packages/contract/src/rfis.ts
import { initContract } from "@ts-rest/core"
import {
  UpsertRfiSchema,
  RfiSchema,
  UpdateRfiStatusSchema,
  ValidateRfiSchema,
} from "@cctech/core"
import { z } from "zod/v4"

const c = initContract()

export const rfi = c.router(
  {
    upsert: {
      method: "POST",
      path: "",
      body: UpsertRfiSchema,
      responses: {
        200: EventsResponse,
      },
      summary: "Create RFI",
    },
    getAll: {
      method: "GET",
      path: "",
      responses: {
        200: successResponseOf(z.array(RfiSchema)),
      },
    },
    updateRFIStatus: {
      method: "PATCH",
      path: "",
      body: UpdateRfiStatusSchema,
      responses: {
        200: UpdateRfiStatusSchema,
      },
    },
  },
  {
    pathPrefix: "/rfi",
  }
)
```

**Example: NC Contract**
```typescript
// packages/contract/src/ncs.ts
export const ncs = c.router(
  {
    getAll: {
      method: "GET",
      path: "",
      responses: {
        200: successResponseOf(z.array(NcsSchema)),
      },
    },
    create: {
      method: "POST",
      path: "",
      body: InsertNcsSchema,
      responses: {
        200: InsertNcsSchema,
      },
    },
    updateNcStatus: {
      method: "PATCH",
      path: "/status",
      body: UpdateNcSchema,
      responses: {
        200: UpdateNcSchema,
      },
    },
  },
  {
    pathPrefix: "/ncs",
  }
)
```

**Example: WMC Contract**
```typescript
// packages/contract/src/wmc.ts
export const wmcContract = c.router({
  getAll: {
    method: "GET",
    path: "/wmc",
    responses: {
      200: WmcGetSchema,
    },
  },
  createPartial: {
    method: "POST",
    path: "/wmc/partial",
    body: CreatePartialWmcSchema,
    responses: {
      200: WMCSchema,
    },
  },
  approve: {
    method: "POST",
    path: "/wmc/:wmcId/approve",
    body: ApproveWmcSchema,
    responses: {
      200: WMCSchema,
    },
  },
})
```

---

## 6. Query Patterns

### 6.1 Backend Queries (RepositoryService)

**Simple Find:**
```typescript
const wmcs = await this.repoSvc.find<WmcType>({
  entity: "WMCS",
  where: { WMC_STATUS: "pending_approval" },
})
```

**Find with Ordering and Limit:**
```typescript
const rfis = await this.repoSvc.find<RfiType>({
  entity: "RFIS",
  where: { STATUS: "submitted" },
  orderBy: "CREATED_AT desc",
  limit: 10,
})
```

**Find with Array (IN clause):**
```typescript
const wmcs = await this.repoSvc.find<WmcType>({
  entity: "WMCS",
  where: { ID: { in: wmcIds } },
})
```

**Find by ID:**
```typescript
const rfi = await this.repoSvc.findById<RfiType>("RFIS", rfiId)
```

**Complex Queries:**
```typescript
// Get WMCs for multiple conditions
const assigneeWmcs = await this.repoSvc.find<WmcAssigneeType>({
  entity: "WMCASSIGNEES",
  where: {
    PERSONNEL_ID: userId,
    UNASSIGNED_AT: null,
  },
})

// Then get related WMCs
const assigneeWmcIds = assigneeWmcs.data.map((assignee) => assignee.WMC_ID)
const wmcs = await this.repoSvc.find<WmcType>({
  entity: "WMCS",
  where: {
    ID: { in: assigneeWmcIds },
    WMC_TYPE: "partial",
  },
})
```

### 6.2 Frontend Queries (Dexie)

**Simple Get:**
```typescript
const rfi = useLiveQuery(() => db.rfis.get(rfiId), [rfiId])
```

**Where Query:**
```typescript
const plots = useLiveQuery(
  () =>
    db.plots
      .where("DESIGN_ELEMENT_ID")
      .equals(plotId)
      .toArray(),
  [plotId],
  []
)
```

**Complex Where with Filter:**
```typescript
const assignments = useLiveQuery(
  async () => {
    return db.personnelRegionAssignments
      .where("PERSONNEL_ID")
      .equals(userId)
      .filter((a) => a.STATUS === "ASSIGNED")
      .toArray()
  },
  [userId],
  []
)
```

**Bulk Get:**
```typescript
const elements = useLiveQuery(
  () =>
    db.designElements
      .bulkGet(elementIds)
      .then((els) => els.filter(Boolean).map((e) => e!)),
  [elementIds],
  []
)
```

**Compound Index Query:**
```typescript
const rfis = useLiveQuery(
  () =>
    db.rfis
      .where("[PLOT_ID+ARCHIVED]")
      .equals([plotId, false])
      .toArray(),
  [plotId],
  []
)
```

### 6.3 API Queries (React Query)

**Query:**
```typescript
const { data, isLoading } = api.wmc.getAll.useQuery({
  queryKey: ["wmcs"],
})
```

**Mutation:**
```typescript
const mutation = api.rfi.upsert.useMutation({
  onSuccess: () => {
    queryClient.invalidateQueries({ queryKey: ["rfis"] })
  },
})

mutation.mutate({
  body: {
    ID: rfiId,
    // ... data
  },
})
```

**Query with Parameters:**
```typescript
const { data } = api.ncs.getNCFiles.useQuery({
  params: { id: ncId },
  queryKey: ["nc-files", ncId],
})
```

---

## 7. Function Building Patterns

### 7.1 Backend Handlers

**Template:**
```typescript
import { Injectable } from "@nestjs/common"
import { CommandHandler, ICommandHandler } from "@nestjs/cqrs"
import { RepositoryService } from "src/repository"
import { CreateEntityCommand } from "../commands/entity.commands"

@Injectable()
@CommandHandler(CreateEntityCommand)
export class CreateEntityHandler implements ICommandHandler<CreateEntityCommand> {
  constructor(private readonly repoSvc: RepositoryService) {}

  async execute(command: CreateEntityCommand) {
    // 1. Validation
    if (!command.data.requiredField) {
      throw new BadRequestException("Required field missing")
    }

    // 2. Business logic
    const relatedEntity = await this.repoSvc.findById(
      "RELATEDENTITY",
      command.data.relatedId
    )

    if (!relatedEntity) {
      throw new NotFoundException("Related entity not found")
    }

    // 3. Database operation
    const result = await this.repoSvc.insert<EntityType>({
      entity: "ENTITIES",
      values: [
        {
          ID: uuidv4(),
          ...command.data,
        },
      ],
      commandId: command.meta.commandId,
      event: "Entities.created",
      actorId: command.meta.user.ID,
    })

    // 4. Return result
    return result[0]
  }
}
```

### 7.2 Frontend Hooks

**Custom Hook Pattern:**
```typescript
// frontend/src/features/nc/queries.ts
export const useGetNcData = (db: Db, ncId: string) => {
  const nc = useLiveQuery(() => db.nc.get(ncId), [ncId])
  const rootNc = useLiveQuery(
    () => db.nc.get(nc?.ROOT_ID ?? ""),
    [nc?.ROOT_ID]
  )

  // ... more queries

  return {
    nc,
    rootNc,
    // ... other data
  }
}
```

**Usage:**
```typescript
const { nc, rootNc } = useGetNcData(db, ncId)
```

---

## 8. Dependencies Deep Dive

### 8.1 Backend Key Dependencies

**Core Framework:**
- `@nestjs/common`, `@nestjs/core` - NestJS framework
- `@nestjs/cqrs` - CQRS pattern implementation
- `@nestjs/platform-express` - Express adapter

**Database:**
- `@sap/cds` - SAP CAP framework
- `@cap-js/sqlite` - SQLite adapter (development)
- `@cap-js/hana` - HANA adapter (production)

**API:**
- `@ts-rest/core` - Type-safe REST contracts
- `@ts-rest/nest` - NestJS integration

**Validation:**
- `zod` - Schema validation

**Utilities:**
- `uuid` - ID generation
- `lodash` - Utility functions

### 8.2 Frontend Key Dependencies

**Core:**
- `react`, `react-dom` - React 19
- `@tanstack/react-router` - Routing
- `@tanstack/react-query` - Data fetching
- `@tanstack/react-form` - Form management

**Database:**
- `dexie` - IndexedDB wrapper
- `dexie-react-hooks` - React hooks for Dexie

**API:**
- `@ts-rest/react-query` - React Query integration
- `@ts-rest/core` - Contract types

**UI:**
- `@ark-ui/react` - Headless UI components
- `@pandacss/dev` - Panda CSS
- `lucide-react` - Icons

**Utilities:**
- `date-fns` - Date manipulation
- `uuid` - ID generation

---

## 9. API Contract Pattern

### 9.1 Contract Definition

Contracts are defined in `packages/contract/src/`:

```typescript
// packages/contract/src/rfis.ts
import { initContract } from "@ts-rest/core"
import { UpsertRfiSchema, RfiSchema } from "@cctech/core"

const c = initContract()

export const rfi = c.router(
  {
    endpointName: {
      method: "POST" | "GET" | "PATCH" | "DELETE",
      path: "/path/:param",
      body: Schema, // For POST/PATCH
      pathParams: z.object({ param: IdSchema }), // For path params
      query: z.object({ queryParam: z.string() }), // For query params
      responses: {
        200: ResponseSchema,
        400: ErrorSchema,
      },
      summary: "Endpoint description",
    },
  },
  {
    pathPrefix: "/rfi",
  }
)
```

### 9.2 Contract Combination

All contracts are combined in `contract.ts`:

```typescript
// packages/contract/src/contract.ts
import { initContract } from "@ts-rest/core"
import { rfi } from "./rfis"
import { ncs } from "./ncs"
import { wmc } from "./wmc"

const c = initContract()

export const contract = c.router({
  rfis: rfi,
  ncs: ncs,
  wmc: wmc,
  // ... other features
})
```

---

## 10. Database Schema (CDS)

SAP CAP uses CDS (Core Data Services) for schema definition:

```cds
// backend/db/schema.cds
namespace CCTECH.DRS.ENTITIES;

type rfi_status : String enum {
  draft;
  submitted;
  rejected;
  approved;
  completed;
}

entity RFIS {
  key ID          : UUID;
      CONTRACTOR_ID: UUID;
      ENGINEER_ID  : UUID;
      STATUS       : rfi_status;
      CREATED_AT   : DateTime;
      UPDATED_AT   : DateTime;
      // ... other fields
}
```

Entity names in code use format: `CCTECH.DRS.ENTITIES_{ENTITY}`

---

## 11. Styling System (Panda CSS)

Panda CSS provides CSS-in-JS with design tokens:

```typescript
// frontend/panda.config.ts
import { defineConfig } from "@pandacss/dev"

export default defineConfig({
  // ... config
})
```

**Usage:**
```typescript
import { Box, HStack, VStack } from "styled-system/jsx"

<VStack gap={4} padding={6}>
  <Box>Content</Box>
</VStack>
```

---

## 12. Authentication & Authorization

Authentication uses Passport.js with JWT:

```typescript
// backend/src/auth/auth.decorators.ts
export const CommandMeta = createParamDecorator(
  (data: unknown, ctx: ExecutionContext): CommandMetaType => {
    const request = ctx.switchToHttp().getRequest()
    return {
      commandId: request.headers["x-command-id"],
      user: request.user,
    }
  }
)
```

**Usage in Controllers:**
```typescript
@TsRestHandler(contract.rfis)
async rfi(@CommandMeta() meta: CommandMetaType) {
  // meta.user contains authenticated user
  // meta.commandId contains command ID
}
```

---

## 13. Error Handling

**Backend:**
- NestJS exception filters
- Custom error responses in controllers

**Frontend:**
- React Error Boundaries
- Toast notifications for user feedback

```typescript
// Error handling in mutations
const mutation = api.rfi.upsert.useMutation({
  onError: (error) => {
    toaster.create({
      title: "Error",
      description: error.message,
      meta: { variant: "destructive" },
    })
  },
})
```

---

## 14. Offline Support

**Outbox Pattern:**
Offline mutations are stored in an outbox and synced when online:

```typescript
// frontend/src/db/outbox-manager.ts
export const OutboxManager = () => {
  const { db, api } = useRouteContext({ from: layoutRoute.id })
  const isOnline = useOnline()

  React.useEffect(() => {
    return liveQuery(() => {
      return db.outbox.where("IS_ERROR").equals(0).toArray()
    }).subscribe(async (outbox) => {
      if (isOnline) {
        for (const command of outbox) {
          // Sync command
          await api[command.MODULE][command.COMMAND].mutate(command.DATA)
          await db.outbox.delete(command.ID)
        }
      }
    }).unsubscribe
  }, [isOnline])
}
```

---

## 15. Common Patterns & Best Practices

### 15.1 Command Pattern Best Practices

1. **Always validate in handlers:**
```typescript
async execute(command: CreateEntityCommand) {
  if (!command.data.requiredField) {
    throw new BadRequestException("Required field missing")
  }
  // ... rest of logic
}
```

2. **Use transactions for multiple operations:**
```typescript
await this.repoSvc.transaction(async (tx) => {
  await this.repoSvc.insert({ ... }, { tx })
  await this.repoSvc.update({ ... }, { tx })
})
```

3. **Return typed entities:**
```typescript
const result = await this.repoSvc.insert<EntityType>({ ... })
return result[0]
```

### 15.2 Frontend Best Practices

1. **Use live queries for reactive data:**
```typescript
const data = useLiveQuery(() => db.entities.get(id), [id])
```

2. **Invalidate queries after mutations:**
```typescript
const mutation = api.entity.create.useMutation({
  onSuccess: () => {
    queryClient.invalidateQueries({ queryKey: ["entities"] })
  },
})
```

3. **Handle loading and error states:**
```typescript
const { data, isLoading, error } = api.entity.getAll.useQuery({
  queryKey: ["entities"],
})

if (isLoading) return <Loader />
if (error) return <Error message={error.message} />
```

### 15.3 Common Pitfalls

1. **Entity name casing:** Always use uppercase in entity names
   - ✅ `entity: "RFIS"`
   - ❌ `entity: "rfis"`

2. **Command ID:** Always include command ID in headers
   ```typescript
   args.headers["x-command-id"] = uuid()
   ```

3. **Dexie indexes:** Define indexes for query performance
   ```typescript
   rfis: "ID, [PLOT_ID+ARCHIVED], [STATUS+ARCHIVED]"
   ```

4. **Type inference:** Use `z.infer<typeof Schema>` for types
   ```typescript
   export type EntityType = z.infer<typeof EntitySchema>
   ```

---

## Conclusion

This documentation covers the comprehensive patterns and practices used in the Pulse-DRS project. Key takeaways:

- **Backend:** CQRS pattern with NestJS, SAP CAP, and ts-rest
- **Frontend:** React 19 with TanStack Router, React Query, and Dexie
- **Shared:** Zod schemas and ts-rest contracts
- **Patterns:** Command handlers, repository service, live queries, offline-first

For specific implementation details, refer to the examples in each section or explore the codebase directly.

