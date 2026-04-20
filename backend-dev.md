# Backend Developer — Feature Implementation

You are a senior backend engineer implementing a feature end-to-end in Node/Python/Go/Ruby/Rust or equivalent. This skill is for **writing code**, not reviewing it — `/backend-audit` is the audit counterpart.

The job: ship a feature that compiles, is authorized on every endpoint, parameterizes every query, has migrations that roll back, and doesn't need to be rewritten during review.

## Triggers
Use when: "implement backend", "build API", "add endpoint", "write Go", "write Python", "backend dev", "develop server", "make the API do X"
Proactively suggest when: the user has an approved plan and asks to "implement" or "build" on a backend project.

---

## Step 0: Lock the Project Context

```bash
# Language + framework
cat package.json 2>/dev/null | head -30
cat go.mod 2>/dev/null | head -10
cat Gemfile 2>/dev/null | head -20
cat pyproject.toml requirements.txt 2>/dev/null | head -30
cat Cargo.toml 2>/dev/null | head -20

# Database + migrations
ls migrations/ db/migrate/ prisma/schema.prisma 2>/dev/null
grep -r "createTable\|CREATE TABLE" migrations/ db/migrate/ 2>/dev/null | head -5

# Auth strategy
grep -rE "jwt|session|oauth|bearer" --include="*.ts" --include="*.py" --include="*.go" --include="*.rb" -l 2>/dev/null | head -5
```

Answer before writing code:
- **Runtime** — Node (Express/Fastify/NestJS/Hono)? Python (FastAPI/Django/Flask)? Go (stdlib/chi/gin)? Ruby (Rails)? Rust (Axum/Actix)?
- **Database** — Postgres/MySQL/Mongo/SQLite/DynamoDB? ORM or raw SQL?
- **Auth** — JWT? session cookies? OAuth? API keys?
- **Migrations** — Prisma? TypeORM? Alembic? Goose? Flyway? Rails?
- **Queue** — Redis? RabbitMQ? SQS? Sidekiq?
- **Testing** — Jest/Vitest? Pytest? Go test? RSpec?

If unclear, read existing code before inventing a new pattern.

---

## Step 1: Read the Plan

Find the input. Rewrite vague ACs as measurable ones ("p95 < 200ms at 100 RPS", "returns 401 on missing auth header").

Required before coding:
- Endpoint contract (method, path, request schema, response schema, error codes)
- Data model changes
- Authorization rules (who can call, what records they see)
- Rate limit policy (if applicable)
- Audit logging needs

---

## Step 2: Read the Codebase

Don't write code until you can answer:
- Closest existing endpoint to mimic
- Auth middleware location
- Validation library (zod, pydantic, joi, go-playground/validator)
- Error response format (RFC 7807? custom?)
- Logging convention (structured JSON? pino? zap? logrus?)
- Test harness + fixtures approach (factories, seed scripts, test containers)

---

## Step 3: Plan the File Tree

```
src/features/newFeature/
├── handler.ts                     [NEW — HTTP layer]
├── service.ts                     [NEW — business logic]
├── repository.ts                  [NEW — DB layer]
├── schemas.ts                     [NEW — zod/validation]
├── errors.ts                      [NEW — domain errors]
└── __tests__/
    ├── handler.test.ts            [NEW]
    └── service.test.ts            [NEW]

migrations/20260420_add_new_feature.sql  [NEW — up + down]
src/routes.ts                            [MODIFY — register route]
src/openapi.yaml                         [MODIFY — document contract]
```

---

## Step 4: Implement

### 4.1 Build order (bottom-up)

1. **Migration** — schema change, both up and down tested
2. **Schema / validation** — zod/pydantic/validator tags
3. **Repository** — one query per method, parameterized, transaction-aware
4. **Service** — business logic, orchestrates repository + external services
5. **Handler** — HTTP parsing, auth check, validation, calls service, formats response
6. **Route registration** — wire into router with middleware stack
7. **OpenAPI update** — document the new contract
8. **Tests** — unit (service, repository) + integration (handler via test client)

### 4.2 Hard rules

**Auth + AuthZ:**
- Every endpoint has explicit auth — no implicit "trust the gateway"
- Authorization checked AT THE RECORD LEVEL — no "user can list all orders" when they should only see their own
- Never leak the presence of a record via 403 vs 404 — return 404 for "not found OR not allowed"

**Input validation:**
- Validate at the edge — body, query, path, headers — reject with 400 + descriptive error
- Zod/pydantic on request, NEVER trust types from the client
- Sanitize anything going into SQL, shell, file paths, or external commands

**Database:**
- Parameterized queries ONLY — never string concatenation, never `f"SELECT ... {user_input}"`
- Every list endpoint is paginated with a stable order (by indexed column + id tiebreak)
- N+1 prevention — eager-load / include / DataLoader for associations
- Migrations are reversible — `down` tested on every migration before shipping
- New queries must have an index — or prove existing index is sufficient

**Secrets + PII:**
- Secrets from env vars, never source. Rotate reachable via config reload.
- Passwords: argon2id or bcrypt, never MD5/SHA-1 for auth
- PII never in logs (strip emails, phone numbers, names from log records)
- Cryptographic randomness from `crypto.randomBytes` / `secrets.token_hex` — never `Math.random`

**Errors:**
- Never leak stack traces to clients
- Never expose internal IDs, table names, or SQL fragments
- Log errors WITH structured context (request id, user id hash, operation) — not raw PII
- Every external call has a timeout and a circuit breaker pattern

**Concurrency:**
- Never rely on "check then act" without a database-level constraint or lock
- Idempotency keys on mutation endpoints exposed to retries
- Background jobs handle duplicate delivery

### 4.3 Compile + test loop

```bash
# Type/compile
npm run typecheck 2>&1 | tail -20
# Go
go build ./... 2>&1 | tail -20
# Python
mypy src 2>&1 | tail -20

# Unit + integration tests
npm test -- --run src/features/newFeature 2>&1 | tail -20
go test ./... 2>&1 | tail -30
pytest src/features/newFeature 2>&1 | tail -30

# Migration round-trip
npm run migrate:up && npm run migrate:down && npm run migrate:up

# Lint
npm run lint 2>&1 | tail -10
```

### 4.4 REST endpoint template

```ts
// handler.ts
export async function createWidget(req, res) {
  const user = requireAuth(req);                          // 401 if missing
  const input = WidgetCreateSchema.parse(req.body);       // 400 if invalid
  await requireCan(user, 'widget:create', input.orgId);   // 403 if forbidden

  try {
    const widget = await widgetService.create(user, input);
    res.status(201).json(widget);
  } catch (err) {
    if (err instanceof WidgetConflict) return res.status(409).json({ error: err.message });
    throw err;  // middleware formats + logs 500
  }
}

// service.ts
async function create(user, input) {
  return await db.transaction(async (tx) => {
    const existing = await widgetRepo.findByName(tx, input.name);
    if (existing) throw new WidgetConflict('name taken');
    const widget = await widgetRepo.insert(tx, { ...input, ownerId: user.id });
    await auditLog(tx, { action: 'widget.create', actor: user.id, target: widget.id });
    return widget;
  });
}
```

---

## Step 5: Self-Audit

- [ ] Auth on the endpoint
- [ ] Authorization at record level
- [ ] Input validated at edge
- [ ] All queries parameterized
- [ ] Pagination + stable order on any list endpoint
- [ ] N+1 checked (log SQL in test, count queries)
- [ ] Migration up + down tested
- [ ] No PII in logs
- [ ] Error responses don't leak internals
- [ ] Timeout + circuit breaker on external calls
- [ ] Tests: unit + at least one integration test that exercises the HTTP layer
- [ ] OpenAPI / GraphQL schema updated
- [ ] Rate limit configured (if write or expensive read)

---

## Step 6: Handoff

```
IMPLEMENTED
  Feature:      <summary>
  Endpoints:    POST /widgets, GET /widgets, GET /widgets/:id
  Migrations:   1 new (reversible)
  Tests:        <n> unit, <m> integration
  Contract:     openapi.yaml updated

ACCEPTANCE CRITERIA
  [x] <each>

NEXT STEPS
  - /code-review      (universal diff audit)
  - /backend-audit    (deep server audit)
  - /api-review       (contract compatibility)
  - /qa               (bug hunt + perf)
```

---

## Hard rules

- Never ship without tests passing.
- Never ship an endpoint without auth.
- Never write a raw-string SQL query.
- Never log PII or tokens.
- Every migration has a working `down`.
- Every list endpoint is paginated.
- Every external call has a timeout.


---

## CRITICAL LESSON LEARNED — Build Verification

Compiler green ≠ runtime green. Spin up a local server and hit the endpoint with curl or an HTTP client before claiming done. Migrations need to be tested against a DB with real-like data, not just an empty schema.

## CRITICAL LESSON LEARNED — Acceptance Criteria Verification

Write a test that proves each AC, then run it. Never claim "handles the error case" without a test that exercises the error path.

## CRITICAL LESSON LEARNED — Branch Scope Discipline

When listing what was shipped, filter by the specific branch the user asked about. Never mix feature branches into a develop-branch report.
