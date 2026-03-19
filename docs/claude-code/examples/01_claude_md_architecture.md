---
title: Designing a Hierarchical CLAUDE.md System for a Monorepo
---

# Designing a Hierarchical CLAUDE.md System for a Monorepo

## Overview

When working with Claude Code in a monorepo, you'll quickly discover that a single `CLAUDE.md` file at the root becomes either too generic to be useful or too bloated with package-specific details. The solution is a hierarchical system where Claude automatically inherits context as it navigates your codebase.

This walkthrough demonstrates how to structure `CLAUDE.md` files across a TypeScript monorepo containing a React frontend, a Node.js API server, and shared packages. You'll learn how to encode coding standards, API patterns, and test conventions so Claude produces consistent, idiomatic code regardless of which package it's working in.

The key insight: Claude Code reads `CLAUDE.md` files from the current working directory and its ancestors. By placing progressively more specific guidance at each level, you create a context system that scales with your codebase.

## Prerequisites

- A monorepo using npm/yarn/pnpm workspaces or Turborepo
- Basic familiarity with Claude Code and the `CLAUDE.md` convention
- Node.js 18+ and TypeScript 5+
- Understanding of your team's existing coding conventions (we'll encode them)

The example assumes this directory structure (we'll create the `CLAUDE.md` files):

```
meridian-platform/
├── package.json
├── turbo.json
├── packages/
│   ├── api-server/
│   ├── web-client/
│   ├── shared-types/
│   └── database/
└── tools/
    └── scripts/
```

## Implementation

### Step 1: Establish the Root CLAUDE.md

The root file defines organization-wide standards and provides Claude with the mental model of your system. This file should answer: "What kind of codebase is this, and what patterns apply everywhere?"

Create `meridian-platform/CLAUDE.md`:

```markdown
# Meridian Platform

## System Overview

Meridian is a B2B logistics platform. The monorepo contains:
- `packages/api-server` - Express + tRPC API serving mobile and web clients
- `packages/web-client` - React 18 SPA with TanStack Query
- `packages/shared-types` - Zod schemas and TypeScript types shared across packages
- `packages/database` - Drizzle ORM schema and migrations

## Universal Coding Standards

### TypeScript Configuration
- Strict mode is enabled everywhere. Never use `any` - prefer `unknown` with type guards.
- Use `satisfies` for type-safe object literals: `const config = { ... } satisfies Config`
- Prefer `interface` for object shapes, `type` for unions and intersections.

### Import Organization
All files use this import order (enforced by ESLint):
1. Node built-ins (`node:fs`, `node:path`)
2. External packages
3. Internal packages (`@meridian/shared-types`, `@meridian/database`)
4. Relative imports

### Error Handling
- Domain errors extend `MeridianError` from `@meridian/shared-types`
- Never throw raw strings or generic `Error` instances
- All async functions that can fail return `Result<T, E>` from `neverthrow`

### Naming Conventions
- Files: `kebab-case.ts` for modules, `PascalCase.tsx` for React components
- Database tables: `snake_case` (e.g., `shipment_tracking_events`)
- API routes: `kebab-case` (e.g., `/api/shipments/tracking-events`)
- Environment variables: `SCREAMING_SNAKE_CASE` with `MERIDIAN_` prefix

## Package Dependencies

When adding dependencies:
- Shared runtime deps go in `packages/shared-types`
- Dev tools (testing, linting) are hoisted to root
- Never install different versions of the same package across workspaces

## Git Conventions

Commit messages follow Conventional Commits:
- `feat(api-server): add shipment consolidation endpoint`
- `fix(web-client): handle null carrier in tracking display`
- `chore(database): add index for shipment lookup by customer`

## Testing Philosophy

- Unit tests live next to source files: `shipment.service.ts` → `shipment.service.test.ts`
- Integration tests go in `__tests__/integration/` within each package
- Use `vitest` everywhere, not Jest
```

**Why this structure matters**: Claude will read this file when working anywhere in the repo. By establishing the mental model upfront ("B2B logistics platform"), Claude makes better naming choices and understands domain concepts. The universal standards prevent inconsistencies that creep in when Claude works across packages.

### Step 2: Create the API Server CLAUDE.md

This file inherits the root context and adds backend-specific patterns. It should answer: "How do we build APIs in this codebase?"

Create `packages/api-server/CLAUDE.md`:

```markdown
# API Server

Inherits from root CLAUDE.md. This document covers backend-specific patterns.

## Architecture

```
src/
├── routes/           # tRPC routers, one per domain
├── services/         # Business logic, no HTTP awareness
├── repositories/     # Database access via Drizzle
├── middleware/       # Express and tRPC middleware
├── jobs/             # Background job processors (BullMQ)
└── utils/            # Server-specific utilities
```

## tRPC Patterns

### Router Definition
Every router follows this structure:

```typescript
// src/routes/shipments.router.ts
import { router, protectedProcedure } from '../trpc';
import { shipmentService } from '../services/shipment.service';
import { 
  CreateShipmentInput, 
  ShipmentId 
} from '@meridian/shared-types';

export const shipmentsRouter = router({
  // Query procedures are for reads
  getById: protectedProcedure
    .input(ShipmentId)
    .query(async ({ input, ctx }) => {
      // Services receive the authenticated user from context
      return shipmentService.getById(input, ctx.user);
    }),

  // Mutation procedures are for writes
  create: protectedProcedure
    .input(CreateShipmentInput)
    .mutation(async ({ input, ctx }) => {
      return shipmentService.create(input, ctx.user);
    }),
});
```

### Input Validation
- All inputs use Zod schemas from `@meridian/shared-types`
- Never define schemas inline in routers - import them
- Add `.describe()` to schema fields for OpenAPI generation

## Service Layer Patterns

Services contain business logic and orchestrate repositories:

```typescript
// src/services/shipment.service.ts
import { Result, ok, err } from 'neverthrow';
import { ShipmentNotFoundError, UnauthorizedError } from '@meridian/shared-types';
import { shipmentRepo } from '../repositories/shipment.repo';
import { trackingService } from './tracking.service';

class ShipmentService {
  async getById(
    id: string, 
    user: AuthenticatedUser
  ): Promise<Result<Shipment, ShipmentNotFoundError | UnauthorizedError>> {
    const shipment = await shipmentRepo.findById(id);
    
    if (!shipment) {
      return err(new ShipmentNotFoundError(id));
    }
    
    // Business rule: users can only view shipments from their organization
    if (shipment.organizationId !== user.organizationId) {
      return err(new UnauthorizedError('shipment', id));
    }
    
    return ok(shipment);
  }
}

// Export singleton instance, not class
export const shipmentService = new ShipmentService();
```

### Service Rules
- Services never import from `routes/` or access `Request`/`Response`
- Services receive already-validated inputs (trust the router layer)
- Business rule violations return domain errors, not HTTP errors
- Log at service level using `ctx.logger`, not `console.log`

## Repository Patterns

Repositories abstract Drizzle queries:

```typescript
// src/repositories/shipment.repo.ts
import { db } from '@meridian/database';
import { shipments, shipmentItems } from '@meridian/database/schema';
import { eq, and, gte } from 'drizzle-orm';

class ShipmentRepository {
  async findById(id: string) {
    // Use query builder for complex queries with relations
    return db.query.shipments.findFirst({
      where: eq(shipments.id, id),
      with: {
        items: true,
        trackingEvents: {
          orderBy: (events, { desc }) => [desc(events.occurredAt)],
          limit: 10,
        },
      },
    });
  }

  async findByOrganization(orgId: string, since: Date) {
    // Use select builder for simpler queries
    return db
      .select()
      .from(shipments)
      .where(
        and(
          eq(shipments.organizationId, orgId),
          gte(shipments.createdAt, since)
        )
      );
  }
}

export const shipmentRepo = new ShipmentRepository();
```

## Error Handling in Routes

Transform domain errors to tRPC errors at the router boundary:

```typescript
// src/middleware/error-handler.ts
import { TRPCError } from '@trpc/server';
import { MeridianError, NotFoundError, UnauthorizedError } from '@meridian/shared-types';

export function toTRPCError(error: MeridianError): TRPCError {
  if (error instanceof NotFoundError) {
    return new TRPCError({ code: 'NOT_FOUND', message: error.message });
  }
  if (error instanceof UnauthorizedError) {
    return new TRPCError({ code: 'FORBIDDEN', message: error.message });
  }
  // Log unexpected errors, return generic message to client
  logger.error('Unexpected error', { error });
  return new TRPCError({ code: 'INTERNAL_SERVER_ERROR' });
}
```

## Background Jobs

Use BullMQ for async processing:

```typescript
// src/jobs/tracking-sync.job.ts
import { Job } from 'bullmq';
import { trackingSyncQueue } from '../queues';
import { trackingService } from '../services/tracking.service';

// Job processor - handles individual jobs
export async function processTrackingSync(job: Job<TrackingSyncPayload>) {
  const { shipmentId, carrierId } = job.data;
  
  await trackingService.syncFromCarrier(shipmentId, carrierId);
  
  // Return value is stored as job result
  return { syncedAt: new Date().toISOString() };
}

// Job scheduler - called from services
export function scheduleTrackingSync(shipmentId: string, carrierId: string) {
  return trackingSyncQueue.add(
    'sync-tracking',  // Job name for debugging
    { shipmentId, carrierId },
    {
      delay: 5000,           // Start after 5 seconds
      attempts: 3,           // Retry up to 3 times
      backoff: {
        type: 'exponential',
        delay: 10000,        // 10s, 20s, 40s between retries
      },
    }
  );
}
```

## Testing

```bash
# Run all API tests
pnpm --filter api-server test

# Run specific test file
pnpm --filter api-server test shipment.service.test.ts

# Run with coverage
pnpm --filter api-server test:coverage
```

### Test Utilities
- Use `createTestContext()` from `src/test-utils/context.ts` for router tests
- Use `createMockUser()` from `src/test-utils/fixtures.ts` for auth mocking
- Database tests use transactions that rollback: `withTestTransaction(async (tx) => { ... })`
```

**Why this depth of detail**: Claude struggles most with "glue code" - the patterns connecting layers. By showing exactly how routers call services, services call repositories, and errors transform between layers, Claude can implement new features that fit the existing architecture.

### Step 3: Create the Web Client CLAUDE.md

Create `packages/web-client/CLAUDE.md`:

```markdown
# Web Client

Inherits from root CLAUDE.md. This document covers React frontend patterns.

## Architecture

```
src/
├── components/        # Shared UI components
│   ├── ui/           # Primitives (Button, Input, Modal)
│   └── domain/       # Business components (ShipmentCard, TrackingTimeline)
├── features/         # Feature modules with co-located code
│   ├── shipments/
│   ├── tracking/
│   └── settings/
├── hooks/            # Shared React hooks
├── lib/              # Utilities (api client, date formatting)
└── routes/           # TanStack Router route definitions
```

## Component Patterns

### UI Components
Primitives use CVA (class-variance-authority) for variants:

```tsx
// src/components/ui/Button.tsx
import { cva, type VariantProps } from 'class-variance-authority';
import { forwardRef } from 'react';
import { cn } from '@/lib/utils';

const buttonVariants = cva(
  // Base styles applied to all variants
  'inline-flex items-center justify-center rounded-md font-medium transition-colors focus-visible:outline-none focus-visible:ring-2 disabled:pointer-events-none disabled:opacity-50',
  {
    variants: {
      variant: {
        primary: 'bg-blue-600 text-white hover:bg-blue-700',
        secondary: 'bg-gray-100 text-gray-900 hover:bg-gray-200',
        destructive: 'bg-red-600 text-white hover:bg-red-700',
        ghost: 'hover:bg-gray-100',
      },
      size: {
        sm: 'h-8 px-3 text-sm',
        md: 'h-10 px