# Full Review — Run All Review Skills Back-to-Back

Run the complete review pipeline automatically: code review, security, platform audit, API contract, design, accessibility, dependencies, performance, and mobile release gate. Routes each review based on what the project changed.

This is the REPORT-ONLY orchestrator. If you want the same coverage PLUS automatic fixes, use `/auto-fix` instead.

## Triggers
Use when: "full review", "review everything", "run all reviews", "comprehensive review", "audit the diff", "pre-merge full audit"

---

## Step 0: Detect What Needs Review

```bash
# Platform
ls *.xcodeproj 2>/dev/null && echo "PLATFORM:ios"
ls build.gradle* 2>/dev/null && echo "PLATFORM:android"
[ -f package.json ] && echo "PLATFORM:web"
[ -f go.mod ] || [ -f requirements.txt ] || [ -f Gemfile ] && echo "PLATFORM:backend"

# What changed
git diff origin/main...HEAD --stat 2>/dev/null || git diff HEAD~5 --stat
git diff origin/main...HEAD --name-only 2>/dev/null | head -30
```

---

## Step 1: Route Reviews

Based on what changed, run applicable reviews in order:

### Always Run:
1. **`/code-review`** — Pre-landing universal diff audit
2. **`/qa`** (Standard mode, report-only) — Bug hunt across the diff
3. **`/security-audit`** — OWASP + secrets + threat model

### P0 Blocker Gate

After EACH review completes, check for P0 findings:
- If P0 found: **STOP** immediately. Show the P0 findings and ask:
  - A) Fix P0 issues now, then resume remaining reviews
  - B) Continue remaining reviews (collect all findings first, then fix)
  - C) Abort — fix manually

Do NOT silently continue past P0 findings.

### Platform-Specific (auto-run by detected platform):
4. **`/ios-audit`** — if .swift files changed
5. **`/android-audit`** — if .kt/.java files changed
6. **`/web-audit`** — if .tsx/.jsx/.vue/.svelte files changed
7. **`/backend-audit`** — if route / controller / service / migration files changed

### If API contracts changed:
8. **`/api-review`** — OpenAPI / GraphQL / gRPC / TS-shared-types compatibility check

### If UI / Frontend Files Changed:
9. **`/design-review`** — UI/UX compliance + polish
10. **`/a11y-audit`** — WCAG 2.1 AA automated + manual screen-reader

### Dependencies / Health:
11. **`/dependencies`** — CVE scan + outdated / abandoned deps

### Optional (ask user):
12. **`/benchmark`** — Performance audit (recommended for perf-sensitive changes)
13. **`/plan-review`** — Architecture re-review (recommended for large refactors)

### Mobile release-candidate only:
14. **`/mobile-quality`** — Lifecycle / device matrix / permissions / store readiness

### If the service ships to production:
15. **`/monitor`** — Observability check — are SLOs, alerts, runbooks in place for the new code path?

---

## Step 2: Run Reviews

For each review, run the skill inline (read the skill file and execute its workflow).

Between reviews, output a progress tracker:

```
FULL REVIEW PROGRESS
━━━━━━━━━━━━━━━━━━━━
[done]    /code-review     — CLEAR (3 P2 notes)
[done]    /qa              — 2 bugs found (1 P1, 1 P2)
[done]    /security-audit  — 1 MEDIUM finding
[running] /ios-audit       — analyzing...
[pending] /design-review   — queued
[pending] /a11y-audit      — queued
[pending] /dependencies    — queued
```

---

## Step 3: Consolidated Report

```
FULL REVIEW REPORT
═══════════════════════════════════
Branch: {branch}
Platform(s): {detected}

┌──────────────────┬────────┬──────────────────────┐
│ Review           │ Status │ Findings             │
├──────────────────┼────────┼──────────────────────┤
│ Code Review      │ CLEAR  │ 3 P2 notes           │
│ QA               │ WARN   │ 2 bugs               │
│ Security         │ WARN   │ 1 MEDIUM finding     │
│ iOS Audit        │ CLEAR  │ 0 issues             │
│ Design           │ CLEAR  │ 2 polish suggestions │
│ A11y             │ WARN   │ 1 missing label      │
│ Dependencies     │ CLEAR  │ 0 CVEs               │
│ API Contract     │ SKIP   │ no API changes       │
│ Backend          │ SKIP   │ no backend changes   │
│ Performance      │ SKIP   │ not requested        │
│ Mobile Quality   │ SKIP   │ not a release candidate │
│ Monitor          │ SKIP   │ not a service change │
└──────────────────┴────────┴──────────────────────┘

TOTAL FINDINGS: X
  P0 (block): 0
  P1 (warn):  3
  P2 (note):  5

DEDUPLICATION: If the same issue was found by multiple reviews (e.g., /code-review and /security-audit both flag a hardcoded secret), show it ONCE with note: "Found by: /code-review, /security-audit"

VERDICT: READY TO SHIP / NEEDS FIXES / BLOCKED
NEXT:
  - If CLEAR: /ship
  - If NEEDS FIXES: /auto-fix  (applies fixes) OR fix manually then re-run
  - If BLOCKED: fix P0s first, then re-run /full-review
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

