# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

n8n is a workflow automation platform written in TypeScript, using a monorepo structure managed by pnpm workspaces with Turbo build orchestration. It consists of a Node.js backend, Vue.js frontend, and extensible node-based workflow engine.

**Requirements:** Node.js >= 22.16, pnpm >= 10.22.0 (npm installs are blocked)

## Essential Commands

### Building

Always redirect build output to a file:
```bash
pnpm build > build.log 2>&1
tail -n 20 build.log
```

### Development
```bash
pnpm dev                  # Full dev mode (frontend + backend, hot reload)
pnpm dev:be               # Backend only
pnpm dev:fe               # Frontend only (editor-ui + design-system)
pnpm dev:ai               # AI/LangChain nodes only
```

Per-package development (recommended for performance):
```bash
# Terminal 1: Backend
cd packages/cli && pnpm dev

# Terminal 2: Frontend
cd packages/frontend/editor-ui && pnpm dev
```

Use `N8N_DEV_RELOAD=true` for hot reload when developing nodes. Dev server at http://localhost:5678, frontend dev server at http://localhost:8080.

### Testing
```bash
pnpm test                 # Run all tests
pnpm test:affected        # Tests based on changes since last commit

# Run a single test file (from the package directory):
cd packages/cli && pnpm test <test-file>
cd packages/nodes-base && pnpm test <test-file>

# Update snapshots
pnpm test -- -u

# Coverage
COVERAGE_ENABLED=true pnpm test

# E2E (Playwright)
pnpm --filter=n8n-playwright test:local
pnpm --filter=n8n-playwright test:local --ui          # Interactive mode
pnpm --filter=n8n-playwright test:local --grep="name"  # Specific tests
```

### Code Quality

Run lint and typecheck from the specific package directory you're working on. Run full-repo checks only when preparing the final PR:
```bash
cd packages/cli && pnpm lint
cd packages/cli && pnpm typecheck
pnpm format               # Prettier + Biome formatting
```

When changes affect type definitions, interfaces in `@n8n/api-types`, or cross-package dependencies, build first before running lint/typecheck.

## Architecture

### Package Structure

| Package | Purpose |
|---------|---------|
| `packages/cli` | Express server, REST API, CLI commands. Entry point for n8n |
| `packages/core` | Workflow execution engine. **Contact n8n before changes** |
| `packages/workflow` | Core workflow interfaces, types, expression evaluation |
| `packages/nodes-base` | 400+ built-in integration nodes and credentials |
| `packages/@n8n/nodes-langchain` | AI/LangChain integration nodes |
| `packages/frontend/editor-ui` | Vue 3 SPA workflow editor (Vite) |
| `packages/frontend/@n8n/design-system` | Reusable Vue component library |
| `packages/frontend/@n8n/stores` | Pinia state management stores |
| `packages/frontend/@n8n/i18n` | Internationalization |
| `packages/frontend/@n8n/rest-api-client` | Typed API client |
| `packages/frontend/@n8n/chat` | Chat interface components |
| `packages/frontend/@n8n/composables` | Shared Vue composables |
| `packages/@n8n/api-types` | Shared TypeScript interfaces between FE/BE |
| `packages/@n8n/config` | Centralized configuration management |
| `packages/@n8n/di` | IoC dependency injection container |
| `packages/@n8n/db` | Database entities, migrations, connection (TypeORM) |
| `packages/@n8n/errors` | Error classes and handling |
| `packages/@n8n/task-runner` | Task runner for isolated execution |
| `packages/testing/playwright` | E2E test suite |
| `packages/testing/containers` | Testcontainers for local stacks |

### Technology Stack

- **Frontend:** Vue 3 + TypeScript + Vite + Pinia + Element Plus
- **Backend:** Node.js + TypeScript + Express + TypeORM
- **Database:** SQLite (dev default), PostgreSQL (production recommended). MySQL/MariaDB deprecated
- **Testing:** Jest (backend unit), Vitest (frontend unit), Playwright (E2E)
- **Code Quality:** Biome (formatting) + ESLint + Prettier + lefthook git hooks

### Key Architectural Patterns

1. **Dependency Injection**: Uses `@n8n/di` for IoC container
2. **Controller-Service-Repository**: Backend follows MVC-like pattern
3. **Event-Driven**: Internal event bus for decoupled communication
4. **State Management**: Frontend uses Pinia stores
5. **Design System**: Pure Vue components go in `@n8n/design-system`

### Workflow Traversal

`workflow.connections` is indexed by **source node**. To find parent nodes, invert first:
```typescript
import { getParentNodes, getChildNodes, mapConnectionsByDestination } from 'n8n-workflow';

const connectionsByDestination = mapConnectionsByDestination(workflow.connections);
const parents = getParentNodes(connectionsByDestination, 'NodeName', 'main', 1);
const children = getChildNodes(workflow.connections, 'NodeName', 'main', 1);
```

## Code Style

### Formatting (Prettier)
- **Indentation:** Tabs (tabWidth: 2)
- **Quotes:** Single quotes
- **Semicolons:** Yes
- **Trailing commas:** All
- **Print width:** 100
- **Line endings:** LF
- **Arrow parens:** Always

### TypeScript Rules
- **NEVER use `any`** — use proper types or `unknown`
- **Avoid `as` type casting** — use type guards instead (except in tests)
- **No `ts-ignore`**
- Define shared FE/BE interfaces in `@n8n/api-types`

### Error Handling
- Don't use `ApplicationError` (deprecated). Use `UnexpectedError`, `OperationalError`, or `UserError` instead
- In nodes: use `NodeOperationError` (user-facing) or `NodeApiError` (API errors)

### Frontend Rules
- **All UI text must use i18n** — add translations to `@n8n/i18n`
- **Use CSS variables** — never hardcode spacing as px values (see `packages/frontend/AGENTS.md` for variable reference)
- **data-testid must be a single value** (no spaces)
- Use `useDebounceFn` with constants from `@/app/constants/durations`

### Testing
- Backend: Jest with `nock` for HTTP mocking, `jest-mock-extended` for interfaces
- Frontend: Vitest
- E2E: Playwright (see `packages/testing/playwright/AGENTS.md` for patterns)
- Workflow tests for nodes use JSON-based definitions with `NodeTestHarness`
- Mock all external dependencies in unit tests
- Confirm test cases with user before writing

## Common Development Workflow

1. Define API types in `packages/@n8n/api-types`
2. Implement backend logic in `packages/cli` (follow `packages/cli/scripts/backend-module/backend-module-guide.md`)
3. Add API endpoints via controllers
4. Update frontend in `packages/frontend/editor-ui` with i18n support
5. Write tests with proper mocks
6. Run `pnpm typecheck` to verify types

## PR Conventions

- Use `gh pr create --draft` for draft PRs
- Follow conventions in `.github/pull_request_title_conventions.md`
- One feature/fix per PR, keep PRs small
- Tests required (auto-closed after 14 days without them)
- New node PRs auto-closed unless explicitly requested by n8n team
- Reference Linear ticket: `https://linear.app/n8n/issue/[TICKET-ID]`
