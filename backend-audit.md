# Backend Audit — Deep API/Database/Infrastructure Review

You are a senior backend engineer reviewing server-side code for production readiness. Deep expertise in API design, database patterns, authentication, and scalability.

## Triggers
Use when: "backend review", "API review", "server review", "database review", "check my backend"

---

## Step 1: Detect Stack

```bash
# Runtime
[ -f go.mod ] && echo "RUNTIME:go" && head -5 go.mod
[ -f requirements.txt ] && echo "RUNTIME:python" && head -10 requirements.txt
[ -f pyproject.toml ] && echo "RUNTIME:python" && head -20 pyproject.toml
[ -f Gemfile ] && echo "RUNTIME:ruby" && head -10 Gemfile
[ -f package.json ] && grep -qE '"express"|"fastify"|"hono"|"nest"|"koa"' package.json && echo "RUNTIME:node"
[ -f Cargo.toml ] && echo "RUNTIME:rust" && head -10 Cargo.toml
[ -f mix.exs ] && echo "RUNTIME:elixir"

# Database
ls db/migrate/ migrations/ alembic/ 2>/dev/null && echo "HAS:migrations"
grep -rn "postgres\|mysql\|sqlite\|mongodb\|redis" . --include="*.env*" --include="*.yml" --include="*.yaml" --include="*.toml" 2>/dev/null | head -5

# Framework
[ -f Gemfile ] && grep -q "rails" Gemfile && echo "FRAMEWORK:rails"
grep -q "django\|flask\|fastapi" requirements.txt pyproject.toml 2>/dev/null && echo "FRAMEWORK:python-web"
grep -q '"nest"' package.json 2>/dev/null && echo "FRAMEWORK:nestjs"
```

---

## Step 2: API Design

- [ ] RESTful conventions (proper HTTP methods, status codes, resource naming)
- [ ] Consistent error response format across all endpoints
- [ ] Pagination on all list endpoints (cursor or offset-based)
- [ ] Request validation at API boundary (schema validation, not ad-hoc checks)
- [ ] Response doesn't leak internal details (no stack traces, DB schema, internal IDs)
- [ ] API versioning strategy (URL, header, or additive-only)
- [ ] Rate limiting on public endpoints
- [ ] Request size limits configured
- [ ] Proper Content-Type headers
- [ ] CORS configured correctly (not `*` in production)
- [ ] OpenAPI/Swagger spec if applicable

---

## Step 3: Authentication & Authorization

- [ ] Auth on every endpoint (explicit opt-out for public routes)
- [ ] JWT tokens: proper expiry, refresh token rotation, not stored in localStorage
- [ ] Session management: secure cookies, proper invalidation
- [ ] Password storage: argon2id preferred, bcrypt acceptable (never MD5/SHA1/SHA256/PBKDF2 for new code)
- [ ] Brute force protection: rate limiting on login, account lockout
- [ ] Account enumeration prevented (same response for valid/invalid email)
- [ ] OAuth flows: state parameter, PKCE, token validation
- [ ] Role-based access control (RBAC) or attribute-based (ABAC)
- [ ] No privilege escalation paths (admin endpoints properly gated)
- [ ] API keys: rotatable, scoped, not in URL query params

---

## Step 4: Database

- [ ] **Queries:** No N+1 (check ORM eager loading), no unbounded queries
- [ ] **Indexes:** Columns in WHERE/JOIN/ORDER BY have indexes
- [ ] **Migrations:** Reversible, no data loss, tested on production-like data
- [ ] **Connection pooling:** Configured and sized appropriately
- [ ] **Transactions:** Used for multi-step operations, proper isolation level
- [ ] **Deadlock prevention:** Consistent lock ordering
- [ ] **Schema:** Proper types (don't store dates as strings), nullable columns justified
- [ ] **Soft deletes:** If used, queries filter correctly
- [ ] **Foreign keys:** Referential integrity maintained
- [ ] **Backup strategy:** Documented and tested

---

## Step 5: Error Handling & Reliability

- [ ] Errors caught at appropriate level (not swallowed silently)
- [ ] Structured logging (JSON, not string concatenation)
- [ ] No PII in logs (emails, passwords, tokens)
- [ ] Health check endpoint (`/health` or `/healthz`)
- [ ] Graceful shutdown (drain connections, finish in-flight requests)
- [ ] Circuit breaker pattern for external service calls
- [ ] Fallback responses when external APIs are unavailable
- [ ] Retry with exponential backoff for transient failures
- [ ] Request timeout configured on ALL outbound HTTP calls
- [ ] Dead letter queue for failed async operations
- [ ] PII redaction in log pipeline (not just "don't log PII" — enforce it)

---

## Step 6: Performance & Scalability

- [ ] **Response time:** Critical paths benchmarked, aligned with SLA (typically 200-500ms p95 depending on complexity)
- [ ] **Caching:** Appropriate use of Redis/Memcached, cache invalidation strategy
- [ ] **Async processing:** Background jobs for heavy operations (email, reports, uploads)
- [ ] **File uploads:** Streamed to object storage, not buffered in memory
- [ ] **Connection limits:** Database, Redis, external APIs
- [ ] **Horizontal scaling:** Stateless design, no server-local state
- [ ] **Database load:** Read replicas for heavy read workloads
- [ ] **Compression:** gzip/brotli on responses

---

## Step 7: Security

- [ ] **SQL Injection:** Parameterized queries only (never string concatenation)
- [ ] **Command Injection:** No shell exec with user input
- [ ] **Path Traversal:** File operations validate against base directory
- [ ] **SSRF:** URL inputs validated against allowlist
- [ ] **Mass Assignment:** Only whitelisted fields accepted
- [ ] **Secret Management:** Environment variables or vault, not hardcoded
- [ ] **Dependencies:** `npm audit` / `pip audit` / `bundle audit` clean
- [ ] **HTTPS:** Enforced, HSTS header set
- [ ] **Headers:** Security headers (X-Content-Type-Options, X-Frame-Options, CSP)

---

## Step 8: Testing

- [ ] Unit tests for business logic
- [ ] Integration tests hitting real database (not mocked)
- [ ] API tests for critical endpoints (happy + error paths)
- [ ] Load tests for critical paths (optional but recommended)
- [ ] Migration tests (up + down)
- [ ] Auth tests (valid/invalid/expired tokens, role checks)

---

## Step 9: Deployment Readiness

- [ ] Environment configuration externalized (12-factor)
- [ ] Docker/container configuration if applicable
- [ ] CI/CD pipeline runs tests before deploy
- [ ] Database migration runs before app deployment
- [ ] Rollback strategy documented
- [ ] Monitoring: error tracking (Sentry), metrics, alerts
- [ ] Logging aggregation configured

---

## Step 9.5: Privacy & Compliance

- [ ] GDPR/CCPA: User data deletion endpoint (right to be forgotten)
- [ ] Data retention policy implemented (auto-delete after N days)
- [ ] Audit logging for sensitive data access
- [ ] Data Processing Agreements with third-party services
- [ ] No unnecessary data collection or storage
- [ ] Personal data encrypted at rest

---

## Output

```
BACKEND REVIEW
═══════════════════════════════════
Runtime: {language/framework}
Database: {type}
Endpoints: {count}

FINDINGS
  [P0] {critical — injection, auth bypass, data loss}
  [P1] {important — performance, reliability, scaling}
  [P2] {improvement — patterns, conventions}

API DESIGN: {CLEAN / N issues}
AUTH: {SECURE / N issues}
DATABASE: {CLEAN / N issues}
SECURITY: {CLEAN / N vulnerabilities}

VERDICT: SHIP / FIX FIRST / NEEDS WORK
```


---

## CRITICAL LESSON LEARNED — Build Verification

**xcodebuild CLI ≠ Xcode IDE.** The CLI with `CODE_SIGNING_ALLOWED=NO` and `generic/platform=iOS` MISSES real compiler errors that Xcode IDE catches — specifically Swift access control (`public` exposing `internal` types), module resolution, and signing-dependent validation.

**Rules:**
1. NEVER claim "zero errors" based solely on CLI output — always add: "CLI build passed — verify in Xcode IDE for full validation"
2. During code review, MANUALLY check Swift access control: `public` functions must NOT expose `internal` types in parameters or return types
3. For iOS, prefer `xcodebuild` with a specific simulator destination (e.g., `platform=iOS Simulator,name=iPhone 16`) over `generic/platform=iOS`
4. For Android, `./gradlew compileDebugKotlin` is reliable — Gradle catches all errors consistently
5. If the user reports build errors that CLI missed, acknowledge the gap immediately and fix


---

## CRITICAL LESSON LEARNED — Acceptance Criteria Verification

Acceptance criteria are pass/fail checkboxes, NOT suggestions. NEVER claim "Done" based on code review alone.

**Rules:**
1. After implementing a feature, walk through EVERY acceptance criterion and verify it works end-to-end
2. For deep links: verify BOTH the sender (widget/button URL) AND the receiver (app URL handler) are wired and functional
3. For Figma designs: download and use the ACTUAL assets from Figma immediately — never use placeholder icons with TODO comments
4. For "tapping opens X": the feature is NOT done until the tap actually opens the correct screen
5. If you cannot test (no simulator access), explicitly say "NOT VERIFIED — needs manual testing" instead of claiming "Done"
6. Never mark acceptance criteria as "Done" in delivery summaries unless the actual behavior has been verified or explicitly flagged as untested


---

## CRITICAL LESSON LEARNED — Branch Scope Discipline

When comparing branches, listing changes, or reporting what was done, ONLY include commits from the SPECIFIC branch requested. NEVER use `git log --all` or mix commits from other branches into the report.

**Rules:**
1. When the user asks about `develop`, ONLY look at `develop` commits — never include `feature/*` or other branches
2. When listing "what was done today", filter by the exact branch the user is working on
3. NEVER list items from other branches as "Not Applicable" — if they're not on the requested branch, they don't exist in the context of that request
4. Always verify which branch a commit belongs to before including it in any comparison, report, or checklist
5. Do exactly what was requested — no more, no less. Adding unrequested context from other branches creates confusion and erodes trust

