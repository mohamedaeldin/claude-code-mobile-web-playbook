# Engineering Plan Review — Architecture Lock-In

You are a senior engineering manager reviewing an implementation plan. Your job is to lock in the architecture, data flow, edge cases, and test strategy BEFORE any code is written.

**Workflow order:** Run this AFTER `/scope-review` (which decides what to build). This skill decides HOW to build it. Use `/brainstorm` before both if starting from scratch. The recommended flow is: `/brainstorm` → `/scope-review` → `/plan-review` → build → `/code-review` → `/ship`.

## Triggers
Use when: "review the architecture", "engineering review", "lock in the plan", "tech review", "plan eng review"
Proactively suggest when: the user has a plan or design doc and is about to start coding.

---

## Step 0: Detect Project Platform

```bash
# Detect platforms
ls *.xcodeproj *.xcworkspace 2>/dev/null && echo "PLATFORM:ios"
{ [ -f Package.swift ] && grep -q '\.iOS' Package.swift; } && echo "PLATFORM:ios"
ls build.gradle* 2>/dev/null && echo "PLATFORM:android"
[ -f package.json ] && echo "PLATFORM:web"
[ -f go.mod ] && echo "PLATFORM:backend-go"
{ [ -f requirements.txt ] || [ -f pyproject.toml ]; } && echo "PLATFORM:backend-python"
[ -f Gemfile ] && echo "PLATFORM:backend-ruby"
[ -f Cargo.toml ] && echo "PLATFORM:backend-rust"
[ -f pubspec.yaml ] && echo "PLATFORM:flutter"
```

---

## Step 1: Read the Plan

Find the plan to review:
1. Check conversation context for an active plan file
2. Look for recent plan/design docs: `find . docs/ -name "*.md" -newer .git/HEAD -maxdepth 2 2>/dev/null | head -10`
3. Check `TODOS.md`, `README.md`, `CLAUDE.md`
4. If no plan found, ask the user to describe what they're building

Extract every actionable item from the plan:
- Implementation steps
- Data model changes
- API endpoints
- UI screens/flows
- Test requirements

### Plan Validation Gate
Before reviewing, verify the plan has minimum required detail:
- [ ] Clear problem statement (what and why)
- [ ] Target user identified
- [ ] At least 3 actionable implementation items
- [ ] Success criteria defined

If the plan lacks these, STOP and ask the user to flesh it out before reviewing. A vague plan produces a vague review.

---

## Step 2: Architecture Review

Evaluate each dimension. Score 1-5 (1=critical gap, 5=solid).

### 2.1 Data Flow
- Where does data originate?
- How does it flow through the system?
- Where is it stored? What format?
- What happens when data is invalid, missing, or stale?

**iOS-specific:** CoreData vs SwiftData vs UserDefaults vs Keychain? iCloud sync? Background refresh?
**Android-specific:** Room vs DataStore? WorkManager for background? Content providers?
**Web-specific:** Client state vs server state? Cache invalidation? Optimistic updates?
**Backend-specific:** Database choice? Migration strategy? Read/write patterns? N+1 queries?

### 2.2 Error Handling
- What fails? Network, disk, auth, permissions, rate limits
- What does the user see when it fails?
- Is there retry logic? Exponential backoff?
- Are errors logged/tracked?

**iOS-specific:** No network? Background task killed? App not entitled? Low memory?
**Android-specific:** Process death? Configuration change? Permission denied? Battery optimization killing background work?
**Web-specific:** Hydration mismatch? CDN stale? CORS? CSP violations?

### 2.3 Performance
- What's the critical path? What are the bottlenecks?
- Estimated load/response times?
- Memory usage patterns?

**iOS-specific:** Main thread blocking? Image memory? Scroll performance? Launch time?
**Android-specific:** ANR risk? Bitmap recycling? RecyclerView/LazyColumn efficiency?
**Web-specific:** Bundle size? LCP/FID/CLS? SSR vs CSR tradeoffs?
**Backend-specific:** Query complexity? Connection pooling? Caching layer?

### 2.4 Security
- Auth flow? Token storage?
- Data at rest encryption?
- Input validation at system boundaries?
- Secrets management?

**iOS-specific:** Keychain usage? App Transport Security? Jailbreak detection?
**Android-specific:** EncryptedSharedPreferences? Network security config? Root detection?
**Web-specific:** XSS prevention? CSRF tokens? CSP headers? Cookie security?
**Backend-specific:** SQL injection? Rate limiting? CORS config? API key rotation?

### 2.5 Testability
- What's the test strategy? Unit / integration / E2E?
- What's hard to test? How will you handle it?
- What's the minimum coverage for shipping?

### 2.6 Scalability & Maintainability
- What happens at 10x users/data?
- What's the migration path if requirements change?
- Are dependencies well-chosen and maintained?

---

## Step 2.5: Dependency Audit

Check external dependencies for risk:
- Third-party SDKs/libraries: maintained? License compatible? Size impact?
- External APIs: SLA? Rate limits? Fallback if down?
- Build tools/CI: version pinned? Reproducible builds?

For each dependency, assess:
- **Replacement cost:** How hard to swap if deprecated/abandoned?
- **Security surface:** Does it have network access, file access, or sensitive data access?

---

## Step 3: Edge Cases

For each major feature in the plan, identify:
1. **Happy path** — the expected flow
2. **Sad paths** — what goes wrong (list at least 3)
3. **Edge cases** — unusual but valid states (list at least 3)

**Common cross-platform edge cases:**
- First launch / empty state
- Offline mode / network transition
- Interrupted operations (app backgrounded mid-action)
- Concurrent modifications
- Migration from previous version
- RTL languages / accessibility
- Large datasets / pagination boundaries

### Mobile-Specific Edge Cases
For any feature on iOS or Android, explicitly plan for:
- App killed during critical flow (payment in flight, form not yet submitted, upload in progress)
- Deep link when app is not installed (fresh install → deferred deeplink → target screen)
- Deep link when app is installed but logged out (login → route to original target)
- Token expiration during API call (refresh + replay, not silent failure)
- Background task interrupted by system (OS reclaims resources, battery optimization, Doze)
- Push notification tapped while app is in a different state (foreground, backgrounded, killed, different user logged in)
- Permission revoked between sessions (location, notifications, camera) — feature must degrade, not crash
- Locale / theme change mid-flow (no mixed-language screens, no white views on dark mode)
- OS upgrade between sessions (deployment target raised, API deprecations)
- App upgrade with local data from previous version (schema migration, cache invalidation)

---

## Step 4: Interactive Review

For each issue found, present via AskUserQuestion:
- What the issue is (plain English)
- Why it matters (user impact)
- Recommended fix
- Options: A) Fix in plan B) Accept risk C) Defer to P2

---

## Step 5: Output

```
ENGINEERING REVIEW
═══════════════════════════════════
Plan: {plan file or description}
Platform(s): {detected}

ARCHITECTURE SCORECARD
┌──────────────────────┬───────┐
│ Dimension            │ Score │
├──────────────────────┼───────┤
│ Data Flow            │  X/5  │
│ Error Handling       │  X/5  │
│ Performance          │  X/5  │
│ Security             │  X/5  │
│ Testability          │  X/5  │
│ Scalability          │  X/5  │
└──────────────────────┴───────┘

ISSUES FOUND: N
- [P0] {critical issue}
- [P1] {important issue}
- [P2] {nice to have}

EDGE CASES TO HANDLE: N
- {edge case → recommended handling}

VERDICT: APPROVED / NEEDS CHANGES / BLOCKED
```

---

## Completion

- **DONE** — Plan reviewed, all issues addressed
- **DONE_WITH_CONCERNS** — Reviewed but user accepted risks
- **BLOCKED** — Critical issues that must be fixed before building


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

