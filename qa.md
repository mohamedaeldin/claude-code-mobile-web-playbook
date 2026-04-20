# QA — Multi-Platform Test, Find Bugs, Fix & Report

Systematically QA test the application. Four modes: Quick, Standard, Exhaustive, or Report-Only (no code changes). Works across iOS, Android, Web, and Backend.

## Triggers
Use when: "qa", "QA", "test this", "find bugs", "test and fix", "fix what's broken", "does this work", "qa report", "find bugs only", "audit the code", "what's broken", "bug hunt"
Proactively suggest when: the user says a feature is ready for testing.

---

## Step 0: Detect Platform & Test Infrastructure

```bash
# Platform detection
ls *.xcodeproj *.xcworkspace 2>/dev/null && echo "PLATFORM:ios"
ls build.gradle* 2>/dev/null && echo "PLATFORM:android"
[ -f package.json ] && echo "PLATFORM:web-or-node"
[ -f go.mod ] || [ -f requirements.txt ] || [ -f Gemfile ] && echo "PLATFORM:backend"

# Test infrastructure
ls jest.config.* vitest.config.* playwright.config.* .rspec pytest.ini 2>/dev/null
ls -d test/ tests/ spec/ __tests__/ cypress/ e2e/ 2>/dev/null
ls *.xcodeproj 2>/dev/null && echo "TEST:xctest"

ls app/src/test app/src/androidTest 2>/dev/null && echo "TEST:android"
```

Ask via AskUserQuestion:
- A) Quick — critical/high bugs only, fix them
- B) Standard — + medium severity, fix them
- C) Exhaustive — + cosmetic, edge cases, accessibility, fix them
- D) Report Only — find all bugs but do NOT edit any files (read-only audit)

**If mode D selected:** apply the **HARD GATE** — do NOT edit any files for the rest of this session. Read-only analysis. Report findings for the team to fix.

---

## Step 1: Understand What to Test

1. Read recent changes: `git log --oneline -10` and `git diff --stat`
2. Read `TODOS.md`, `README.md` for feature context
3. Identify the feature/flow to test
4. Map the critical user paths

---

## Step 2: Run Existing Tests

### iOS
```bash
xcodebuild test -scheme <scheme> -destination 'platform=iOS Simulator,name=iPhone 16' -quiet 2>&1 | tail -30
# Or Swift Package
swift test 2>&1 | tail -30
```

### Android
```bash
./gradlew test 2>&1 | tail -30
./gradlew connectedAndroidTest 2>&1 | tail -30
```

### Web
```bash
npm test 2>&1 | tail -50
# Or
bun test 2>&1 | tail -50
# Or
npx vitest run 2>&1 | tail -50
```

### Backend
```bash
# Detect and run appropriate test suite
[ -f Gemfile ] && bundle exec rails test 2>&1 | tail -30
{ [ -f pytest.ini ] || [ -f pyproject.toml ]; } && pytest 2>&1 | tail -30
[ -f go.mod ] && go test ./... 2>&1 | tail -30
```

Record: total tests, passed, failed, skipped.

---

## Step 3: Code-Level Bug Hunt

**Scope:** Focus on files changed in recent commits. Don't review the entire codebase — review what's new or modified:
```bash
git diff --name-only HEAD~10 2>/dev/null || git diff --name-only 2>/dev/null
```

Read the changed files and look for:

### Universal Bugs
- Unhandled null/nil/undefined
- Off-by-one errors in loops/pagination
- Race conditions in async code
- Resource leaks (unclosed connections, file handles)
- Logic errors in conditionals
- Missing error handling at boundaries
- Hardcoded values that should be configurable

### iOS-Specific Bugs
- Force unwraps that will crash on nil
- Main thread violations (UI updates from background)
- Retain cycles (closures capturing self without weak/unowned)
- Missing @MainActor on UI-touching code
- Incorrect Sendable conformance
- CoreData context threading violations

### Android-Specific Bugs
- Memory leaks (Activity/Context references in long-lived objects)
- ANR risks (network/disk on main thread)
- Lifecycle bugs (work in wrong state, configuration change crashes)
- Missing null checks on Intent extras
- Coroutine scope leaks

### Web-Specific Bugs
- XSS vectors in user-rendered content
- Stale closure bugs in React hooks
- Missing loading/error states
- Hydration mismatches (SSR)
- Infinite re-render loops

### Backend-Specific Bugs
- SQL injection in dynamic queries
- N+1 queries in list endpoints
- Missing auth checks on endpoints
- Unbounded queries without pagination
- Race conditions in concurrent operations

---

## Step 4: Fix Bugs Found (skip if Report Only mode)

For each bug found:

1. **Document it:**
   ```
   BUG #{n}: {title}
   Severity: CRITICAL / HIGH / MEDIUM / LOW / COSMETIC
   Location: {file}:{line}
   Impact: {what breaks for the user}
   ```

2. **Fix it** (if severity >= current tier threshold and NOT in Report Only mode)

3. **Commit atomically:**
   ```bash
   git add <fixed-files>
   git commit -m "fix: {description of bug}"
   ```

4. **Verify the fix** — re-run the specific test or verify the code path

---

## Step 5: Write Regression Tests (skip if Report Only mode)

For each bug fixed, write a regression test appropriate to the platform:

- **iOS:** XCTest function with Arrange/Act/Assert
- **Android:** JUnit `@Test` function
- **Web:** vitest/jest `test()` block
- **Backend:** framework-appropriate test

---

## Step 6: Re-run Full Test Suite (skip if Report Only mode)

Run all tests again to confirm:
- All previously passing tests still pass
- New regression tests pass
- No new failures introduced

---

## Step 7: QA Report

### If Fix mode (A/B/C):
```
QA REPORT
═══════════════════════════════════
Platform: {detected}
Scope: {quick/standard/exhaustive}
Feature: {what was tested}

HEALTH SCORE: {before} → {after}

TESTS
  Before: {passed}/{total} passing
  After:  {passed}/{total} passing
  New:    {count} regression tests added

BUGS FOUND: {count}
  [CRITICAL] #{n} {title} — FIXED
  [HIGH]     #{n} {title} — FIXED
  [MEDIUM]   #{n} {title} — DEFERRED
  [LOW]      #{n} {title} — NOTED

FIXES
  {count} bugs fixed, {count} commits
  {count} regression tests added

SHIP READINESS: READY / NOT READY
{1-line summary}
```

### If Report Only mode (D):
```
QA REPORT (READ-ONLY)
═══════════════════════════════════
Platform: {detected}
Feature: {what was analyzed}

BUGS FOUND: {count}
  [CRITICAL] #{n} {file}:{line} — {description}
    Impact: {user impact}
    Suggested Fix: {what to change}
  [HIGH]     #{n} {file}:{line} — {description}
    Impact: {user impact}
    Suggested Fix: {what to change}
  [MEDIUM]   #{n} {file}:{line} — {description}

TEST RESULTS: {passed}/{total}

RECOMMENDED FIX ORDER:
  1. {most critical bug first}
  2. ...

STATUS: REPORT ONLY — no fixes applied
```


---

## MOBILE-SPECIFIC QA CHECKLIST

Run these in addition to the universal/iOS/Android bug hunt when the detected platform is iOS or Android. The deeper mobile pre-release hard gate lives in `/mobile-quality` — this checklist is the minimum QA-level coverage so obvious mobile regressions don't ship.

### Lifecycle Testing
- [ ] App background → foreground (state preserved, no crash)
- [ ] App killed (swipe away / process death) → reopen
- [ ] Interrupted flows: incoming call, notification, system alert, Control Center / quick settings
- [ ] Low-memory warning: views rebuild without state loss
- [ ] Configuration change: locale switch, theme switch, font-scale change

### Network Conditions
- [ ] Offline mode — clear empty/offline UI, no silent stalls
- [ ] Slow network (3G/edge) — loading states visible, no ANR
- [ ] Intermittent connection — retries succeed, no duplicate writes
- [ ] Airplane mode toggle mid-request — fails gracefully
- [ ] Token expiry mid-request — refresh + replay, no silent drop

### Navigation & Deep Links
- [ ] Every deeplink opens the correct screen from cold launch
- [ ] Every deeplink opens the correct screen from warm resume
- [ ] Push notification tap routes correctly when app is backgrounded
- [ ] Push notification tap routes correctly when app is killed
- [ ] Deeplink while logged out → login → route continues to target
- [ ] Back-navigation from deeplink target lands on a sensible parent

### Device Matrix
- [ ] Smallest supported screen (iPhone SE / small Android): no clipping
- [ ] Largest screen (Pro Max / tablet): no wasted space
- [ ] iPad / Android tablet: adaptive layout or explicit block
- [ ] RTL (Arabic): layouts mirror, no fixed-LTR screens
- [ ] Accessibility text sizes (Dynamic Type XXL / fontScale 2.0): no overflow
- [ ] Dark mode: every screen has a dark variant

### Permissions
- [ ] Denied → feature degrades gracefully, app explains what's missing
- [ ] Limited (iOS "Selected Photos", Android "While Using") → works in reduced mode
- [ ] Allowed → full feature
- [ ] Re-request flow works after user flipped the switch in Settings
- [ ] Rationale copy matches the platform (no Android wording in iOS prompts or vice versa)

If any of these fail on a release candidate, escalate to `/mobile-quality` for the full pre-submission gate.


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

