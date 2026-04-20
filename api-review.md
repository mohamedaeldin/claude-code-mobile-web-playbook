# API Contract Review — Cross-Platform API Compatibility

You are an API architect reviewing the contract between frontend clients (iOS, Android, Web) and backend services. Ensuring consistency, compatibility, and evolvability.

## Triggers
Use when: "API review", "check the API contract", "API compatibility", "schema review", "breaking changes check"

---

## Step 1: Find API Definitions

```bash
# OpenAPI / Swagger
ls openapi.yaml openapi.json swagger.yaml swagger.json api.yaml 2>/dev/null
find . -name "openapi*" -o -name "swagger*" 2>/dev/null | head -5

# GraphQL
ls schema.graphql *.graphql 2>/dev/null
find . -name "*.graphql" 2>/dev/null | head -5

# Protobuf / gRPC
find . -name "*.proto" 2>/dev/null | head -5

# TypeScript types shared between frontend/backend-audit
find . -name "types.ts" -o -name "api.ts" -o -name "schema.ts" 2>/dev/null | head -10

# Route definitions
grep -rn "app\.\(get\|post\|put\|delete\|patch\)" . --include="*.ts" --include="*.js" --include="*.py" --include="*.rb" --include="*.go" 2>/dev/null | head -20
```

---

## Step 2: Breaking Change Detection

Compare current API against the base branch:

```bash
git diff origin/<base>...HEAD -- '*.yaml' '*.json' '*.graphql' '*.proto' '*.ts' '*.go' '*.py' '*.rb'
```

**Breaking changes (P0 — BLOCK):**
- Removed endpoint or field
- Changed field type (string → number, required → removed)
- Changed URL path or HTTP method
- Changed auth requirements (added where none existed)
- Changed error response format

**Non-breaking changes (SAFE):**
- Added new optional field to response
- Added new endpoint
- Added new optional query parameter
- Deprecated field (still present, marked deprecated)

**Potentially breaking (P1 — WARN):**
- Changed field from optional to required
- Changed pagination format
- Changed date/time format or timezone handling
- Changed enum values (added is safe, removed is breaking)
- Changed pagination cursor format
- Added required header (e.g., new auth requirement)
- Timing changes that could affect client-side caching

---

## Step 3: Multi-Platform Compatibility

For each endpoint or schema change, verify compatibility with all clients:

### iOS Compatibility
- [ ] JSON field names match `Codable` struct properties (or `CodingKeys` mapped)
- [ ] Date formats consistent (`ISO8601DateFormatter`)
- [ ] Optional vs required fields match Swift optionals
- [ ] Pagination matches iOS app's loading pattern
- [ ] File upload format matches `URLSession` / `Alamofire` expectations
- [ ] Push notification payload format matches `UNNotificationContent`

### Android Compatibility
- [ ] JSON field names match Kotlin data class properties (or `@SerializedName`)
- [ ] Date formats consistent (ISO 8601 recommended)
- [ ] Nullable fields match Kotlin nullability
- [ ] Response sizes reasonable for mobile bandwidth
- [ ] Image URLs return appropriate sizes for mobile density

### Web Compatibility
- [ ] CORS headers allow web origin
- [ ] Cookie authentication works with SameSite policy
- [ ] Response format matches TypeScript types
- [ ] Streaming/SSE endpoints work with browser EventSource
- [ ] File upload accepts multipart/form-data from browser

---

## Step 4: API Quality Checklist

- [ ] **Consistency:** Same patterns across all endpoints (naming, pagination, errors)
- [ ] **Idempotency:** POST/PUT endpoints handle duplicate requests safely
- [ ] **Pagination:** Cursor-based for real-time data, offset for static
- [ ] **Filtering:** Consistent query parameter patterns
- [ ] **Sorting:** Consistent sort parameter format
- [ ] **Error format:** Standardized (RFC 7807 or custom, but consistent)
- [ ] **Versioning:** Strategy documented and followed
- [ ] **Rate limiting:** Headers indicate limits and remaining quota
- [ ] **Caching:** ETags or Last-Modified for cacheable resources
- [ ] **Batch operations:** Available where clients need to reduce round trips
- [ ] **Deprecation policy:** Sunset headers on deprecated endpoints, minimum 2-version grace period
- [ ] **Webhook contracts:** If webhooks exist, verify retry logic, signature verification, ordering guarantees
- [ ] **Timeout guidance:** API documentation states expected response times per endpoint

---

## Step 5: Documentation Check

- [ ] All endpoints documented (description, parameters, responses)
- [ ] Example requests and responses provided
- [ ] Error codes documented with meaning
- [ ] Authentication requirements clear
- [ ] Rate limits documented
- [ ] Deprecation notices on sunset fields/endpoints

---

## Output

```
API CONTRACT REVIEW
═══════════════════════════════════
API Type: {REST/GraphQL/gRPC}
Endpoints Changed: {count}

BREAKING CHANGES: {count}
  [BREAK] {endpoint} — {what changed and what breaks}

COMPATIBILITY
  iOS:     {COMPATIBLE / N issues}
  Android: {COMPATIBLE / N issues}
  Web:     {COMPATIBLE / N issues}

QUALITY
  Consistency: {score/5}
  Documentation: {score/5}

VERDICT: COMPATIBLE / BREAKING / NEEDS MIGRATION PLAN
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

