---
title: Parallel Subagent Pattern for Test and Implementation
---

# Parallel Subagent Pattern for Test and Implementation

## Overview

This walkthrough demonstrates how to orchestrate two Claude Code instances running in parallel—one generating integration tests from an OpenAPI specification, while the other implements the corresponding API handlers. Both agents work from the same contract (the API spec), then their outputs merge into a cohesive, tested implementation.

This pattern matters because it mirrors how high-performing teams actually work: test authors and implementers can proceed concurrently when they share a well-defined contract. By encoding this workflow into automated subagents, you achieve:

1. **Faster iteration cycles** — tests and implementation develop simultaneously rather than sequentially
2. **Contract-first discipline** — both agents derive their work from the same source of truth
3. **Reduced integration friction** — conflicts surface early at merge time rather than late in code review

The "clone pattern" here refers to spawning multiple Claude Code processes from a shared repository state, letting each diverge independently, then reconciling their changes.

## Prerequisites

**Tooling:**
- Claude Code CLI installed and authenticated (`claude` command available)
- Git 2.30+ (for worktree support)
- Node.js 20+ with npm
- A Unix-like shell (bash/zsh)

**Repository structure we'll create:**
```
order-service/
├── api/
│   └── openapi.yaml          # Our contract
├── src/
│   └── handlers/             # Implementation target
├── tests/
│   └── integration/          # Test target
├── package.json
└── tsconfig.json
```

**Conceptual familiarity:**
- Git worktrees (isolated working directories sharing the same repo)
- OpenAPI 3.0 specification format
- Express.js route handlers
- Jest integration testing patterns

## Implementation

### Step 1: Establish the Shared Contract

Before spawning parallel agents, we need the API specification both will reference. This contract defines the `POST /orders` and `GET /orders/{orderId}` endpoints for a simplified order service.

Create `api/openapi.yaml`:

```yaml
openapi: 3.0.3
info:
  title: Order Service API
  version: 1.0.0
  description: Handles order creation and retrieval for the e-commerce platform

servers:
  - url: http://localhost:3000/api/v1

paths:
  /orders:
    post:
      operationId: createOrder
      summary: Create a new order
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateOrderRequest'
      responses:
        '201':
          description: Order created successfully
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Order'
        '400':
          description: Invalid request payload
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'

  /orders/{orderId}:
    get:
      operationId: getOrderById
      summary: Retrieve an order by ID
      parameters:
        - name: orderId
          in: path
          required: true
          schema:
            type: string
            format: uuid
      responses:
        '200':
          description: Order found
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Order'
        '404':
          description: Order not found
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'

components:
  schemas:
    CreateOrderRequest:
      type: object
      required:
        - customerId
        - items
      properties:
        customerId:
          type: string
          format: uuid
        items:
          type: array
          minItems: 1
          items:
            $ref: '#/components/schemas/OrderItem'
        shippingAddress:
          $ref: '#/components/schemas/Address'

    OrderItem:
      type: object
      required:
        - productId
        - quantity
        - unitPriceCents
      properties:
        productId:
          type: string
          format: uuid
        quantity:
          type: integer
          minimum: 1
        unitPriceCents:
          type: integer
          minimum: 0
          description: Price per unit in cents to avoid floating point issues

    Address:
      type: object
      required:
        - street
        - city
        - postalCode
        - country
      properties:
        street:
          type: string
        city:
          type: string
        postalCode:
          type: string
        country:
          type: string
          pattern: '^[A-Z]{2}$'  # ISO 3166-1 alpha-2

    Order:
      type: object
      properties:
        id:
          type: string
          format: uuid
        customerId:
          type: string
          format: uuid
        items:
          type: array
          items:
            $ref: '#/components/schemas/OrderItem'
        totalCents:
          type: integer
          description: Computed sum of all item totals
        status:
          type: string
          enum: [pending, confirmed, shipped, delivered, cancelled]
        createdAt:
          type: string
          format: date-time

    ErrorResponse:
      type: object
      properties:
        code:
          type: string
        message:
          type: string
        details:
          type: array
          items:
            type: object
            properties:
              field:
                type: string
              issue:
                type: string
```

Commit this to establish the shared baseline:

```bash
git add api/openapi.yaml
git commit -m "Add OpenAPI contract for order service"
```

### Step 2: Create Isolated Worktrees for Parallel Development

Git worktrees let us create separate working directories that share the same repository history. Each Claude Code instance will operate in its own worktree, preventing file conflicts during parallel execution.

```bash
#!/bin/bash
# scripts/setup-parallel-worktrees.sh

set -euo pipefail

REPO_ROOT=$(git rev-parse --show-toplevel)
BASE_BRANCH=$(git rev-parse --abbrev-ref HEAD)

# Create branches for each agent's work
git branch -f agent/test-author "$BASE_BRANCH"
git branch -f agent/implementer "$BASE_BRANCH"

# Create worktrees in a sibling directory to avoid nesting issues
WORKTREE_BASE="${REPO_ROOT}/../order-service-worktrees"
mkdir -p "$WORKTREE_BASE"

# Remove existing worktrees if present (idempotent setup)
git worktree remove --force "${WORKTREE_BASE}/test-author" 2>/dev/null || true
git worktree remove --force "${WORKTREE_BASE}/implementer" 2>/dev/null || true

# Create fresh worktrees
git worktree add "${WORKTREE_BASE}/test-author" agent/test-author
git worktree add "${WORKTREE_BASE}/implementer" agent/implementer

echo "Worktrees created:"
echo "  Test Author: ${WORKTREE_BASE}/test-author"
echo "  Implementer: ${WORKTREE_BASE}/implementer"
```

Why worktrees instead of separate clones? Worktrees share object storage, making branch operations instant and keeping disk usage minimal. They also guarantee both agents start from identical commit states.

### Step 3: Define Agent Prompts as Reusable Files

Each agent needs a detailed prompt that establishes its role, constraints, and expected outputs. We store these as files for version control and reproducibility.

Create `prompts/test-author.md`:

```markdown
# Role: Integration Test Author

You are writing integration tests for the Order Service API. Your tests will run against a real HTTP server (not mocked handlers), validating the contract defined in the OpenAPI specification.

## Your Constraints

1. **Read the OpenAPI spec** at `api/openapi.yaml` to understand the exact contract
2. **Write tests** in `tests/integration/orders.test.ts`
3. **Use Jest** with `supertest` for HTTP assertions
4. **Test both happy paths and error cases** specified in the API responses
5. **Do not implement handlers** — assume they will return appropriate responses
6. **Use realistic test data** that matches the schema constraints

## Required Test Coverage

For `POST /orders`:
- Successfully creates an order with valid payload (201)
- Rejects requests missing required fields (400)
- Rejects requests with invalid field formats (400)
- Validates that `totalCents` is correctly computed in response

For `GET /orders/{orderId}`:
- Returns existing order (200)
- Returns 404 for non-existent order ID
- Validates response matches Order schema

## Test File Structure

```typescript
// tests/integration/orders.test.ts
import request from 'supertest';
import { app } from '../../src/app';
// ... your tests here
```

## Definition of Done

- All tests are syntactically valid TypeScript
- Tests use descriptive names that document the behavior
- Tests are independent (no shared mutable state between tests)
- File includes necessary imports and type definitions
```

Create `prompts/implementer.md`:

```markdown
# Role: API Handler Implementer

You are implementing the Express.js route handlers for the Order Service API. Your handlers must conform exactly to the OpenAPI specification.

## Your Constraints

1. **Read the OpenAPI spec** at `api/openapi.yaml` to understand the exact contract
2. **Implement handlers** in `src/handlers/orders.ts`
3. **Create types** in `src/types/orders.ts` matching the OpenAPI schemas
4. **Use an in-memory store** for this implementation (Map-based, not a real database)
5. **Do not write tests** — assume tests exist and will validate your implementation
6. **Compute `totalCents`** as the sum of `quantity * unitPriceCents` for all items

## Required Implementation

Create these files:

### `src/types/orders.ts`
TypeScript interfaces matching all OpenAPI schemas (CreateOrderRequest, Order, etc.)

### `src/handlers/orders.ts`
- `createOrder` — validates input, generates UUID, computes total, stores order, returns 201
- `getOrderById` — retrieves from store, returns 200 or 404

### `src/routes/orders.ts`
Express Router wiring handlers to paths

### Update `src/app.ts`
Mount the orders router at `/api/v1`

## Validation Requirements

- Validate required fields are present
- Validate UUIDs are properly formatted
- Validate arrays have minimum length where specified
- Return structured error responses matching ErrorResponse schema

## Definition of Done

- All files are syntactically valid TypeScript
- Handlers match the operationId names from the spec
- Error responses include meaningful messages
- Code includes JSDoc comments for public functions
```

### Step 4: Create the Parallel Orchestration Script

This script spawns both Claude Code instances simultaneously, captures their outputs, and waits for both to complete.

Create `scripts/run-parallel-agents.sh`:

```bash
#!/bin/bash
# scripts/run-parallel-agents.sh
#
# Orchestrates parallel Claude Code execution for test authoring and implementation.
# Uses background processes with job control to achieve true parallelism.

set -euo pipefail

REPO_ROOT=$(git rev-parse --show-toplevel)
WORKTREE_BASE="${REPO_ROOT}/../order-service-worktrees"
LOG_DIR="${REPO_ROOT}/logs/parallel-run-$(date +%Y%m%d-%H%M%S)"

mkdir -p "$LOG_DIR"

echo "Starting parallel agent execution..."
echo "Logs will be written to: $LOG_DIR"

# Function to run an agent in its worktree
run_agent() {
    local agent_name=$1
    local worktree_path=$2
    local prompt_file=$3
    local log_file="${LOG_DIR}/${agent_name}.log"
    
    echo "[${agent_name}] Starting in ${worktree_path}"
    
    # Change to worktree and run Claude Code
    # --print outputs the final result without interactive mode
    # --max-turns limits iteration to prevent runaway agents
    (
        cd "$worktree_path"
        
        # Ensure dependencies are installed in worktree
        npm install --silent 2>/dev/null || true
        
        # Run Claude Code with the prompt file as input
        claude --print \
               --max-turns 25 \
               --verbose \
               < "${REPO_ROOT}/${prompt_file}" \
               > "$log_file" 2>&1
        
        # Commit any changes made by the agent
        git add -A
        git commit -m "Agent ${agent_name}: automated implementation" --allow-empty
        
        echo "[${agent_name}] Completed. See ${log_file}"
    ) &
    
    # Return the background process ID
    echo $!
}

# Launch both agents in parallel
TEST_PID=$(run_agent "test-author" "${WORKTREE_BASE}/test-author" "prompts/test-author.md")
IMPL_PID=$(run_agent "implementer" "${WORKTREE_BASE}/implementer" "prompts/implementer.md")

echo "Agents running in background:"
echo "  Test Author PID: $TEST_PID"
echo "  Implementer PID: $IMPL_PID"

# Wait for both to complete, capturing exit codes
TEST_EXIT=0
IMPL_EXIT=0

wait $TEST_PID || TEST_EXIT=$?
wait $IMPL_PID || IMPL_EXIT=$?

# Report results
echo ""
echo "=== Execution Complete ==="
echo "Test Author exit code: $TEST_EXIT"
echo "Implementer exit code: $IMPL_EXIT"

if [[ $TEST_EXIT -ne 0 || $IMPL_EXIT -ne 0 ]]; then
    echo "WARNING: One or more agents failed. Check logs for details."
    exit 1
fi

echo ""
echo "Both agents completed successfully."
echo "Next step: Run scripts/merge-agent-work.sh to combine their work."
```

### Step 5: Implement the Merge and Conflict Resolution Script

After both agents