# Technical Requirements & Best Practices — Colmena

> **Purpose:** This document acts as guardrails for Claude Code. All code generation, refactoring, or review MUST comply with the rules defined here. If a rule conflicts with a user request, Claude Code must warn before proceeding.

---

## 1. General Principles

### 1.1 Design Philosophy

- Prioritize readability over cleverness. Code is read far more often than it is written.
- Favor composition over inheritance.
- Apply the principle of least surprise: a function's behavior should be predictable from its name and signature.
- Design for change: external dependencies (databases, APIs, services) must sit behind abstractions that allow substitution.
- Every module, function, or component should have a single reason to change (SRP).
- **Everything runs in Docker.** No service, database, or tool should require local installation beyond Docker and Docker Compose.

### 1.2 Language & Naming Conventions

- Code, variable names, functions, classes, types, and interfaces: **English**.
- Inline comments and technical documentation: **English**.
- Product documentation and commits: English.
- Naming conventions:
  - `camelCase` for variables, functions, and methods.
  - `PascalCase` for classes, interfaces, types, enums, and React components.
  - `UPPER_SNAKE_CASE` for environment constants and immutable configuration values.
  - `kebab-case` for file and folder names (except React components which use `PascalCase`).
  - `I` prefix for interfaces is NOT allowed (use the semantic name directly: `User`, not `IUser`).
  - `T` prefix for types is NOT allowed.
  - Booleans must read as assertions: `isLoading`, `hasPermission`, `canEdit`.

---

## 2. Docker & Containerization

### 2.1 Core Principle

The entire development and production environment MUST be containerized. A new developer should be able to clone the repo, run `docker compose -f docker-compose.dev.yml up`, and have the full stack running — no other local dependency required apart from Docker itself.

### 2.2 Project Structure

```
├── docker/
│   ├── server/
│   │   ├── Dockerfile              # Multi-stage production build
│   │   └── Dockerfile.dev          # Development with hot reload
│   ├── ui/
│   │   ├── Dockerfile              # Multi-stage production build
│   │   └── Dockerfile.dev          # Development with hot reload
│   └── nginx/
│       └── nginx.conf              # Reverse proxy (production)
├── docker-compose.yml              # Production: Colmena + Tasks.md + Postgres + nginx
├── docker-compose.dev.yml          # Development: hot reload, bind mounts
├── docker-compose.test.yml         # Test environment (ephemeral DB)
├── .dockerignore
└── .env.example                    # Template for required env vars
```

### 2.3 Dockerfile Standards

#### Multi-stage builds are mandatory for production images

```dockerfile
# ── Stage 1: Dependencies ────────────────────────────────────────
FROM node:22-alpine AS deps
WORKDIR /app
RUN corepack enable
COPY package.json pnpm-workspace.yaml pnpm-lock.yaml ./
COPY server/package.json server/
COPY ui/package.json ui/
COPY packages/shared/package.json packages/shared/
COPY packages/db/package.json packages/db/
RUN pnpm install --frozen-lockfile

# ── Stage 2: Build ───────────────────────────────────────────────
FROM node:22-alpine AS build
WORKDIR /app
RUN corepack enable
COPY --from=deps /app /app
COPY . .
RUN pnpm --filter @colmena/ui build
RUN pnpm --filter @colmena/server build

# ── Stage 3: Production ─────────────────────────────────────────
FROM node:22-alpine AS production
WORKDIR /app
RUN apk add --no-cache ca-certificates curl git \
  && corepack enable \
  && addgroup -g 1001 appgroup && adduser -u 1001 -G appgroup -s /bin/sh -D appuser \
  && npm install --global @anthropic-ai/claude-code@latest \
  && mkdir -p /colmena && chown appuser:appgroup /colmena
COPY --from=build --chown=appuser:appgroup /app/dist ./dist
COPY --from=build --chown=appuser:appgroup /app/node_modules ./node_modules
COPY --from=build --chown=appuser:appgroup /app/package.json ./
COPY --from=build --chown=appuser:appgroup /app/skills ./skills

ENV NODE_ENV=production
VOLUME ["/colmena", "/tasks", "/workspace"]
EXPOSE 3100

USER appuser
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:3100/api/health || exit 1
CMD ["node", "dist/server.js"]
```

#### Development Dockerfile

```dockerfile
# docker/server/Dockerfile.dev
FROM node:22-alpine
WORKDIR /app
RUN corepack enable
COPY package.json pnpm-workspace.yaml pnpm-lock.yaml ./
COPY server/package.json server/
COPY packages/shared/package.json packages/shared/
COPY packages/db/package.json packages/db/
RUN pnpm install
COPY . .
CMD ["pnpm", "dev:server"]
```

```dockerfile
# docker/ui/Dockerfile.dev
FROM node:22-alpine
WORKDIR /app
RUN corepack enable
COPY package.json pnpm-workspace.yaml pnpm-lock.yaml ./
COPY ui/package.json ui/
COPY packages/shared/package.json packages/shared/
RUN pnpm install
COPY . .
CMD ["pnpm", "dev:ui"]
```

#### Mandatory Dockerfile rules

- Always use `node:22-alpine` (pinned major version, alpine for size).
- Use `pnpm install --frozen-lockfile` for deterministic builds (production); `pnpm install` for dev.
- Run the application as a non-root user in production images.
- Always include a `HEALTHCHECK` instruction.
- Copy `package.json` and lock file before source code to leverage Docker layer caching.
- Production images must NOT contain devDependencies, source code, test files, or `.env` files.
- Pre-install Claude Code CLI globally in the production image.
- Development images use bind mounts (volumes) for hot reload — source code is NOT copied in `docker-compose.dev.yml`.

### 2.4 Docker Compose

#### Development (`docker-compose.dev.yml`)

```yaml
services:
  server:
    build:
      context: .
      dockerfile: docker/server/Dockerfile.dev
    volumes:
      - ./server/src:/app/server/src           # Hot reload
      - ./packages:/app/packages               # Hot reload
      - ./skills:/app/skills                   # Skills
      - /app/node_modules                      # Prevent overwrite
      - colmena_data:/colmena
      - ./tasks:/tasks
      - ./workspace:/workspace
    ports:
      - "${COLMENA_PORT:-3100}:3100"
    env_file: .env
    depends_on:
      db:
        condition: service_healthy

  ui:
    build:
      context: .
      dockerfile: docker/ui/Dockerfile.dev
    volumes:
      - ./ui/src:/app/ui/src                   # Hot reload
      - ./packages:/app/packages               # Hot reload
      - /app/node_modules
    ports:
      - "${UI_PORT:-5173}:5173"
    env_file: .env
    depends_on:
      - server

  db:
    image: postgres:17-alpine
    volumes:
      - db_data:/var/lib/postgresql/data
    environment:
      POSTGRES_USER: ${DB_USER:-colmena}
      POSTGRES_PASSWORD: ${DB_PASSWORD:-colmena}
      POSTGRES_DB: ${DB_NAME:-colmena}
    ports:
      - "${DB_PORT:-5432}:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER:-colmena}"]
      interval: 5s
      timeout: 3s
      retries: 5

  tasks-md:
    image: baldissaramatheus/tasks.md
    ports:
      - "${TASKS_PORT:-8080}:8080"
    environment:
      - PUID=1000
      - PGID=1000
      - TITLE=Colmena Board
    volumes:
      - ./tasks:/tasks

volumes:
  db_data:
  colmena_data:
```

#### Production (`docker-compose.yml`)

```yaml
services:
  server:
    build:
      context: .
      dockerfile: docker/server/Dockerfile
    ports:
      - "${COLMENA_PORT:-3100}:3100"
    environment:
      DATABASE_URL: postgres://${DB_USER:-colmena}:${DB_PASSWORD:-colmena}@db:5432/${DB_NAME:-colmena}
      ANTHROPIC_API_KEY: ${ANTHROPIC_API_KEY:-}
      COLMENA_HOME: /colmena
    volumes:
      - colmena_data:/colmena
      - ./tasks:/tasks
      - ./workspace:/workspace
    depends_on:
      db:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:3100/api/health"]
      interval: 30s
      timeout: 5s
      retries: 3

  db:
    image: postgres:17-alpine
    volumes:
      - db_data:/var/lib/postgresql/data
    environment:
      POSTGRES_USER: ${DB_USER:-colmena}
      POSTGRES_PASSWORD: ${DB_PASSWORD:-colmena}
      POSTGRES_DB: ${DB_NAME:-colmena}
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER:-colmena}"]
      interval: 5s
      timeout: 3s
      retries: 5

  tasks-md:
    image: baldissaramatheus/tasks.md
    ports:
      - "${TASKS_PORT:-8080}:8080"
    environment:
      - PUID=1000
      - PGID=1000
      - TITLE=Colmena Board
    volumes:
      - ./tasks:/tasks
    depends_on:
      server:
        condition: service_healthy

volumes:
  db_data:
  colmena_data:
```

#### Docker Compose rules

- Every service MUST have a `healthcheck`.
- Use `depends_on` with `condition: service_healthy` — never plain `depends_on`.
- Use named volumes for persistent data (databases). Never bind-mount database data directories.
- All port mappings use environment variables with sensible defaults.
- No hardcoded credentials — use `.env` file (gitignored) with `.env.example` as template.
- Infrastructure services (DB) use official Alpine images with a pinned major version.
- Colmena and Tasks.md share the `/tasks` volume for filesystem-based task sync.
- The `/workspace` volume is where agents' code projects live.
- Development uses bind mounts for hot reload; production uses COPY in Dockerfile.

### 2.5 .dockerignore

```
node_modules
dist
build
.git
.env
.env.*
!.env.example
*.md
.vscode
.idea
coverage
.nyc_output
```

- The `.dockerignore` MUST exist and MUST exclude `node_modules`, `.git`, and environment files.

### 2.6 Testing Environment

```yaml
# docker-compose.test.yml
services:
  test-server:
    build:
      context: .
      dockerfile: docker/server/Dockerfile.dev
    command: pnpm test:run
    env_file: .env.test
    depends_on:
      test-db:
        condition: service_healthy

  test-db:
    image: postgres:17-alpine
    environment:
      POSTGRES_USER: test
      POSTGRES_PASSWORD: test
      POSTGRES_DB: colmena_test
    tmpfs: /var/lib/postgresql/data     # In-memory for speed
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U test"]
      interval: 3s
      timeout: 2s
      retries: 5
```

- Test databases use `tmpfs` mounts for speed — data is ephemeral.
- Tests run inside containers in CI, using the same images as development.
- A single command runs the full test suite: `docker compose -f docker-compose.test.yml up --abort-on-container-exit --exit-code-from test-server`.

### 2.7 Local Development Workflow

```bash
# First time setup
cp .env.example .env                                    # Configure environment
docker compose -f docker-compose.dev.yml up -d          # Start all services

# Daily development
docker compose -f docker-compose.dev.yml up -d          # Start
docker compose -f docker-compose.dev.yml logs -f server # Tail logs
docker compose -f docker-compose.dev.yml down           # Stop

# Run tests
docker compose -f docker-compose.test.yml up --abort-on-container-exit

# Run a one-off command (e.g., migrations)
docker compose -f docker-compose.dev.yml exec server pnpm db:migrate

# Rebuild after dependency changes
docker compose -f docker-compose.dev.yml up -d --build
```

---

## 3. TypeScript — Global Rules

### 3.1 Strict Configuration (mandatory)

```jsonc
// tsconfig.json — mandatory fields
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "exactOptionalPropertyTypes": true,
    "forceConsistentCasingInFileNames": true,
    "isolatedModules": true,
    "skipLibCheck": true,
    "esModuleInterop": true,
    "resolveJsonModule": true,
    "module": "ESNext",
    "moduleResolution": "bundler",
    "target": "ES2022"
  }
}
```

### 3.2 Typing Rules

- **`any` is forbidden.** Use `unknown` when the type is genuinely unknown and apply type narrowing.
- **`as` casting is forbidden** except in tests or integrations with untyped libraries. In that case, encapsulate the cast in a utility function with runtime validation (e.g., with Zod).
- **`@ts-ignore` is forbidden.** Use `@ts-expect-error` only with a comment justifying the reason.
- **`!` (non-null assertion) is forbidden** in production code. Perform explicit checks.
- Prefer `interface` for public contracts (API responses, component props) and `type` for unions, intersections, and derived types.
- Use `satisfies` to validate that a literal meets a type without losing inference.
- Enums are only allowed as `const enum` or, preferably, replaced by string union types: `type Status = "active" | "inactive" | "pending"`.
- Use `readonly` on object properties that must not be mutated.
- Use `as const` for arrays and objects acting as constants.
- Return types of public functions must be explicit. Internal/private functions may rely on inference if clear.

### 3.3 Imports & Modules

- Use path aliases (`@/`) to avoid relative paths deeper than one level (`../../`).
- Do not use `default export` except in page/route components where the framework requires it. Prefer `named exports`.
- Group imports in this order (separated by blank lines):
  1. Node built-ins (`node:fs`, `node:crypto`).
  2. External libraries (express, drizzle-orm, react…).
  3. Internal aliases (`@colmena/shared`, `@colmena/db`…).
  4. Relative imports from the same module.
  5. Types (with `import type`).
- Use `import type` whenever importing exclusively a type.

---

## 4. Architecture — Backend (Node.js)

### 4.1 Monorepo Structure

```
colmena/
├── server/                      # Express.js API server
│   ├── src/
│   │   ├── modules/             # Domain modules (feature-based)
│   │   │   └── <module>/
│   │   │       ├── <module>.controller.ts    # HTTP layer: request parsing, response
│   │   │       ├── <module>.service.ts       # Pure business logic
│   │   │       ├── <module>.repository.ts    # Data access (Drizzle queries)
│   │   │       ├── <module>.router.ts        # Route definitions
│   │   │       ├── <module>.schema.ts        # Zod validation schemas
│   │   │       ├── <module>.types.ts         # Module-specific types
│   │   │       └── __tests__/
│   │   ├── adapters/            # Agent runtime adapters (claude_local)
│   │   ├── shared/
│   │   │   ├── errors/          # Custom error classes
│   │   │   ├── middleware/      # Auth, logging, error handling
│   │   │   └── config/         # Configuration and env vars
│   │   ├── infra/
│   │   │   ├── database/        # Connection, migrations
│   │   │   ├── secrets/         # Secret provider implementations
│   │   │   ├── storage/         # File storage (run logs)
│   │   │   └── realtime/        # WebSocket server
│   │   ├── app.ts               # Application composition
│   │   └── server.ts            # Entry point (bootstrap only)
│   └── package.json
│
├── ui/                          # React SPA
│   ├── src/
│   │   ├── pages/               # Route pages
│   │   ├── components/
│   │   │   ├── ui/              # shadcn/ui primitives
│   │   │   └── *.tsx            # Feature components
│   │   ├── api/                 # API client modules
│   │   ├── hooks/               # Custom hooks
│   │   ├── context/             # React context providers
│   │   └── lib/                 # Utilities, query keys
│   └── package.json
│
├── packages/
│   ├── db/                      # Drizzle schema + migrations
│   │   ├── src/
│   │   │   ├── schema/          # One file per table
│   │   │   ├── migrations/      # SQL migration files
│   │   │   ├── client.ts        # DB connection + migration runner
│   │   │   └── index.ts         # Public exports
│   │   └── package.json
│   │
│   └── shared/                  # Shared types, constants, validators
│       ├── src/
│       │   ├── types/           # TypeScript interfaces
│       │   ├── validators/      # Zod schemas
│       │   └── constants.ts     # Status enums, role lists, etc.
│       └── package.json
│
├── skills/                      # Built-in Colmena agent skill
│   └── colmena/
│       └── SKILL.md
│
├── docker/                      # Docker configurations
│   ├── server/
│   │   ├── Dockerfile
│   │   └── Dockerfile.dev
│   ├── ui/
│   │   ├── Dockerfile
│   │   └── Dockerfile.dev
│   └── nginx/
│       └── nginx.conf
│
├── docker-compose.yml           # Production
├── docker-compose.dev.yml       # Development
├── docker-compose.test.yml      # Testing
├── pnpm-workspace.yaml
├── package.json
└── tsconfig.base.json
```

### 4.2 Layers & Responsibilities

| Layer | Responsibility | Knows about |
|---|---|---|
| **Router** | Define routes and attach middleware | Controller |
| **Controller** | Parse request, invoke service, format response | Service, Schema |
| **Service** | Pure business logic. No direct HTTP or database access | Repository (via injection) |
| **Repository** | Encapsulate database queries. Returns domain entities | Drizzle ORM (infra/database) |

- The controller NEVER contains business logic.
- The service NEVER accesses `req`, `res`, or sends HTTP responses.
- The repository NEVER contains business logic; it only translates between the persistence model and the domain.
- The service NEVER uses Drizzle directly; all database access goes through repositories.

```typescript
// ✅ Repository: encapsulates Drizzle queries
export function createTaskRepository(db: Db): TaskRepository {
  return {
    async findByFilters(filters?: TaskFilters) {
      const conditions = [eq(tasks.status, "todo")];
      if (filters?.assignee) conditions.push(eq(tasks.assigneeAgentId, filters.assignee));
      return db.select().from(tasks).where(and(...conditions));
    },
    async create(data: CreateTaskInput) {
      return db.insert(tasks).values(data).returning().then(rows => rows[0]);
    },
    async checkoutAtomically(taskId: string, agentId: string, runId: string) {
      return db.update(tasks).set({ ... }).where(and(...)).returning();
    },
  };
}

// ✅ Service: pure business logic, receives repository
export function createTaskService(deps: {
  taskRepository: TaskRepository;
  fileSync: TaskFileSync;
}): TaskService {
  return {
    async checkout(taskId: string, agentId: string, runId: string) {
      const task = await deps.taskRepository.checkoutAtomically(taskId, agentId, runId);
      if (!task) throw new ConflictError("Task checkout conflict");
      await deps.fileSync.moveToLane(task, "In Progress");
      return task;
    },
  };
}
```

### 4.3 Dependency Injection

- Use constructor injection or factory functions (no complex IoC container).
- Each service receives its dependencies as parameters, facilitating testing.
- Repositories receive `Db`; services receive repositories and other services.

```typescript
// ✅ Composition at app startup
const db = createDb(databaseUrl);
const taskRepo = createTaskRepository(db);
const agentRepo = createAgentRepository(db);
const secretRepo = createSecretRepository(db);
const fileSync = createTaskFileSync(tasksDir);
const taskService = createTaskService({ taskRepository: taskRepo, fileSync });
const agentService = createAgentService({ agentRepository: agentRepo });
const secretService = createSecretService({ secretRepository: secretRepo, masterKey });
const heartbeatService = createHeartbeatService({ taskService, agentService, secretService });
```

### 4.4 Error Handling

- Define a domain error hierarchy extending `Error`:
  - `AppError` (base) with `statusCode`, `code`, and `isOperational`.
  - `NotFoundError`, `ValidationError`, `UnauthorizedError`, `ForbiddenError`, `ConflictError`.
- Services throw domain errors. A global middleware translates them to HTTP responses.
- Never use generic `try/catch` that swallows errors. If caught, re-throw or handle explicitly.
- Log with appropriate levels: `error` for failures, `warn` for recoverable anomalies, `info` for normal flows, `debug` for development.

### 4.5 Validation

- Validate ALL inputs at the system boundary (controller) with Zod.
- Zod schemas live in `@colmena/shared` (shared between server and UI) or in the module's `<module>.schema.ts`:

```typescript
// packages/shared/src/validators/task.ts
import { z } from "zod";

export const createTaskSchema = z.object({
  title: z.string().min(1),
  epicId: z.string().uuid(),
  priority: z.enum(["critical", "high", "medium", "low"]).default("medium"),
  assigneeAgentId: z.string().uuid().optional(),
});

export type CreateTaskInput = z.infer<typeof createTaskSchema>;
```

### 4.6 Environment Variables

- Validate ALL environment variables at startup with Zod and fail fast if any are missing.
- Do not access `process.env` directly outside the configuration module.
- The configuration module exports a typed, immutable object.

### 4.7 REST API — Conventions

- Use plural nouns for resources: `/api/tasks`, `/api/agents`.
- Success responses: return the entity directly (consistent with Paperclip).
- Error responses: `{ error: string }`.
- Correct HTTP status codes: 200 (ok), 201 (created), 204 (no content), 400 (bad request), 401 (unauthorized), 403 (forbidden), 404 (not found), 409 (conflict), 422 (unprocessable entity), 500 (internal error).
- No URL versioning for MVP.

---

## 5. Architecture — Frontend (React + shadcn/ui)

### 5.1 Folder Structure

```
ui/src/
├── pages/                     # Route pages
├── components/
│   ├── ui/                    # shadcn/ui components (do not modify the API)
│   └── *.tsx                  # Feature components
├── api/                       # API client modules (one per resource)
├── hooks/                     # Custom hooks
├── context/                   # React context providers
├── lib/                       # Utilities, query keys, router config
├── adapters/                  # Adapter-specific UI (transcript parsers)
└── styles/                    # Global styles
```

### 5.2 Component Rules

- One component per file. The file is named `PascalCase.tsx`.
- Functional components exclusively. No class components.
- Props typed with `interface`, exported alongside the component.
- UI components must be "dumb" (presentational): they receive data and callbacks via props.
- 200-line limit per component. If exceeded, extract sub-components or hooks.
- Do not use `React.FC`.

### 5.3 State & Data Fetching

- Server state: **TanStack Query** is mandatory. Do not manage server state in local stores.
  - Every query has a descriptive and stable `queryKey`.
  - Mutations use `onSuccess` to invalidate related queries.
- Client state: `useState` for simple state, `useReducer` for complex transitions. Zustand only if truly global client state is needed.
- No prop drilling beyond 2 levels. Use composition or Context.
- Do not use `useEffect` to derive state. Compute derived values directly in render or with `useMemo`.

### 5.4 shadcn/ui — Specific Rules

- Do not modify base shadcn/ui components directly. For variants, use `cva` or wrapper components.
- All forms must use React Hook Form + Zod resolver.
- Tailwind CSS is the only permitted styling system. No CSS modules, no styled-components, no inline styles.
- Do not use arbitrary Tailwind values (`w-[347px]`) unless documented justification exists.
- Tailwind classes ordered with the `prettier-plugin-tailwindcss` plugin.

### 5.5 Performance

- Use `React.lazy()` + `Suspense` for code-splitting routes.
- Do not use `useMemo` or `useCallback` by default. Only with measured evidence of costly re-renders.
- Long lists (>100 items): virtualization is mandatory.

### 5.6 Accessibility (a11y)

- All interactive elements must be keyboard-accessible.
- Use semantic elements (`button`, `nav`, `main`, `section`).
- Forms: every input has an associated `label`.
- Images: descriptive `alt` mandatory.
- Colors: minimum WCAG AA contrast.
- shadcn components are a11y-compliant by design; do not break this contract.

---

## 6. Clean Code — Cross-Cutting Rules

### 6.1 Functions

- Maximum 20 lines per function (guideline; justify if exceeded).
- Maximum 3 parameters. If more, use an options object.
- A function does one thing. If you describe it with "and", split it.
- No unexpected side effects.
- Prefer early return / guard clauses over nesting.

### 6.2 Immutability

- Do not mutate input parameters.
- Use spread operator, `Array.map()`, `Array.filter()` to create new values.
- Mark arrays and objects as `readonly` when they must not be mutated.
- The `delete` operator on objects is forbidden. Use destructuring to omit properties.

### 6.3 Strings & Template Literals

- Use template literals for interpolation. No concatenation with `+`.
- Do not build SQL by concatenating strings. Use Drizzle's query builder or `sql` template literals.

### 6.4 Handling null/undefined

- Prefer `undefined` over `null` for optional values.
- Use optional chaining (`?.`) and nullish coalescing (`??`).
- Do not use `||` for numeric or boolean defaults. Use `??`.

### 6.5 Async/Await

- Always `async/await`. No `.then()/.catch()` chains.
- Use `Promise.all()` for independent parallel operations.
- Always type async function returns: `Promise<Task>`, not `Promise<any>`.

### 6.6 Comments

- Code must be self-documenting. Comments explain the "why", not the "what".
- TODO/FIXME MUST be accompanied by an issue/ticket identifier.

### 6.7 Files

- One file, one responsibility.
- Maximum 300 lines per file (guideline). If exceeded, evaluate extraction.
- Barrel files (`index.ts`) only at the root of a package to re-export the public API.

---

## 7. Testing

### 7.1 Philosophy

- Tests are first-class citizens. Written alongside the code, not "later".
- Follow the testing pyramid: many unit tests, some integration tests, few e2e.
- Minimum acceptable coverage is 80% on lines and branches for business code (services).

### 7.2 Framework & Tools

- **Vitest** as the test runner and assertion framework.
- **Testing Library** for React component tests.
- **Faker.js** for test data generation.

### 7.3 Test Structure

- **AAA pattern** is mandatory: Arrange → Act → Assert. Separated by blank lines.
- Descriptive test names indicating behavior: `"should return 404 when task does not exist"`.
- One `it()` = one concept to verify.
- No `test.skip` or `test.only` in commits.
- Use factories/builders for test objects:

```typescript
import { faker } from "@faker-js/faker";

export function buildTask(overrides?: Partial<Task>): Task {
  return {
    id: faker.string.uuid(),
    title: faker.lorem.sentence(),
    status: "todo",
    priority: "medium",
    approval: "pending",
    ...overrides,
  };
}
```

### 7.4 Running Tests

```bash
# Unit tests (inside Docker)
docker compose -f docker-compose.test.yml run --rm test-server pnpm test:unit

# Full test suite (with ephemeral DB)
docker compose -f docker-compose.test.yml up --abort-on-container-exit --exit-code-from test-server

# Watch mode during development
docker compose -f docker-compose.dev.yml exec server pnpm test:watch

# Coverage report
docker compose -f docker-compose.test.yml run --rm test-server pnpm test:coverage
```

---

## 8. Static Analysis & Tooling

### 8.1 ESLint

Mandatory base configuration with:
- `@typescript-eslint/strict-type-checked`
- `@typescript-eslint/no-explicit-any`: error
- `@typescript-eslint/no-floating-promises`: error
- `@typescript-eslint/consistent-type-imports`: error
- `eqeqeq`: always
- `no-var`: error
- `prefer-const`: error
- For frontend: `react-hooks/exhaustive-deps`: error, `jsx-a11y/strict`

### 8.2 Prettier

```jsonc
{
  "semi": true,
  "singleQuote": false,
  "trailingComma": "all",
  "printWidth": 100,
  "tabWidth": 2,
  "arrowParens": "always",
  "plugins": ["prettier-plugin-tailwindcss"]
}
```

### 8.3 Husky + lint-staged

- `--max-warnings=0`: lint warnings are errors in CI.
- Prettier runs on pre-commit via lint-staged.

### 8.4 Conventional Commits

Mandatory format:

```
<type>(<scope>): <description>
```

Allowed types: `feat`, `fix`, `docs`, `style`, `refactor`, `perf`, `test`, `build`, `ci`, `chore`, `revert`.

- Scope is the affected module: `feat(heartbeat): add session persistence`.
- Description in imperative mood, lowercase, no trailing period.

---

## 9. Database — Drizzle ORM + PostgreSQL

### 9.1 Database Setup

- PostgreSQL 17 runs in Docker (both dev and production).
- Connection via `DATABASE_URL` environment variable.
- Drizzle ORM with `postgres-js` driver.
- No embedded Postgres — Docker provides the database in all environments.

### 9.2 Schema Conventions

- One file per table in `packages/db/src/schema/`.
- UUID primary keys with `defaultRandom()`.
- All timestamps with timezone: `timestamp("created_at", { withTimezone: true })`.
- JSONB for flexible/polymorphic fields (adapter config, metadata, context snapshots).
- Single workspace — no company-scoped FKs. Foreign keys reference `agents`, `tasks`, etc. directly.
- Index naming: `{table}_{columns}_idx` (e.g., `tasks_status_idx`, `agents_name_idx`).

### 9.3 Migrations

- Generate: `docker compose -f docker-compose.dev.yml exec server pnpm db:generate`
- Apply: `docker compose -f docker-compose.dev.yml exec server pnpm db:migrate`
- Auto-apply on startup is acceptable for development.
- Never hand-edit generated migration files.
- Migration files are committed to the repository.

### 9.4 Query Patterns

- All Drizzle queries live in **repository** files, never in services or controllers.
- Use Drizzle's query builder for standard CRUD.
- Use `sql` template literal for complex aggregations.
- Use `db.transaction()` for multi-table atomic operations.
- Use `returning()` to get the inserted/updated row without a separate SELECT.

---

## 10. Security — Minimum Rules

- Never commit secrets, tokens, or credentials. Use `.env` (gitignored) + Zod validation at startup.
- Sanitize outputs to prevent XSS. React escapes by default; never use `dangerouslySetInnerHTML`.
- Validate ALL user inputs with Zod at the boundary.
- Use security headers: `helmet` in Express.
- SQL: always use Drizzle's query builder or `sql` template literals. Never concatenate user inputs.
- Agent output is untrusted — parse defensively, never execute arbitrary code from agent stdout.
- Secrets encrypted at rest (AES-256-GCM). Master key file with `0600` permissions.
- Agent JWTs are short-lived (48h default) and scoped to agent + run.

---

## 11. Git & Workflow

### 11.1 Branching

- Main branch: `main` (always deployable).
- Feature branches: `feat/<short-description>`.
- Fix branches: `fix/<short-description>`.
- No direct commits to `main`.

### 11.2 Pull Requests

- Maximum 400 changed lines per PR (excluding generated files/lock files).
- CI must pass: lint + types + tests + build.

### 11.3 Rules for Claude Code

When Claude Code generates or modifies code in this project:

1. **Before writing code**, verify it complies with the architecture rules from the corresponding section (backend or frontend).
2. **All new code includes tests.** No production code is generated without its associated test.
3. **Respect the monorepo structure** (`server/src/modules/`, `ui/`, `packages/db/`, `packages/shared/`).
4. **Do not generate `any`**, `@ts-ignore`, non-null assertions, or castings without justification.
5. **Validate inputs with Zod** at the controller layer.
6. **Respect the layered architecture**: controller → service → repository. Services never use Drizzle directly.
7. **Follow Conventional Commits** in commit messages.
8. **Run lint and type-check** before proposing code as finished.
9. **Do not install dependencies** without consulting. Prefer already established tools.
10. **If a user request violates these rules**, inform the user and propose an alternative.
11. **All services must be containerized.** Any new service, database, or tool must have its own Docker configuration and be added to `docker-compose.dev.yml`.
12. **Never propose commands that require local installation** of databases, runtimes, or services. Everything runs through Docker Compose.

---

## 12. Quick Review Checklist

Before considering any task complete, verify:

- [ ] Code compiles without errors (`pnpm typecheck`).
- [ ] ESLint passes with zero warnings (`eslint --max-warnings=0`).
- [ ] Tests pass and cover the main paths.
- [ ] No unjustified `any`, `@ts-ignore`, `!`, or `as`.
- [ ] Inputs are validated with Zod at the controller boundary.
- [ ] Layered architecture respected: controller → service → repository.
- [ ] Monorepo structure is respected (`server/src/modules/`, `packages/`).
- [ ] Names follow conventions (camelCase, PascalCase, etc.).
- [ ] No secrets or credentials in the code.
- [ ] React components are accessible (semantics, labels, contrast).
- [ ] Commit message follows Conventional Commits.
- [ ] PR has a reasonable size (<400 lines).
- [ ] `.env.example` is updated if new environment variables were added.
- [ ] Drizzle schema changes have a generated migration.
- [ ] New services are containerized and added to Docker Compose.
- [ ] Dockerfiles follow multi-stage build pattern for production.
- [ ] Docker images run as non-root user.
- [ ] Health checks are defined for all services.
