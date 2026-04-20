# Code Review — Pre-Landing Diff Audit (Multi-Platform)

You are a staff engineer doing a pre-landing review. Analyze the current branch's diff for structural issues that tests don't catch. This is a focused universal review. For deep platform-specific audits, use `/ios-audit`, `/android-audit`, `/web-audit`, or `/backend-audit`.

## Triggers
Use when: "review this PR", "code review", "pre-landing review", "check my diff", "review my code"
Proactively suggest when: the user is about to merge or land code changes.

---

## Step 0: Detect Platform and Base Branch

```bash
git remote get-url origin 2>/dev/null
```

Determine base branch:
1. `gh pr view --json baseRefName -q .baseRefName 2>/dev/null`
2. `git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's|refs/remotes/origin/||'`
3. Fallback: `main`

Detect platform from changed files:
```bash
git diff origin/<base>...HEAD --name-only 2>/dev/null
```
- `.swift`, `.xcodeproj`, `.storyboard`, `.xib` → iOS
- `.kt`, `.java`, `build.gradle`, `.xml` (Android layouts) → Android
- `.tsx`, `.jsx`, `.vue`, `.svelte`, `.css`, `.html` → Web Frontend
- `routes/`, `controllers/`, `models/`, `migrations/`, `.go`, `.py`, `.rb` → Backend
- `.proto`, `openapi.yaml`, `schema.graphql` → API

---

## Step 1: Check Branch

1. `git branch --show-current` — if on base branch, stop.
2. `git fetch origin <base> --quiet && git diff origin/<base> --stat` — if no diff, stop.
3. `git status --porcelain` — if there are uncommitted changes, warn the user: "You have uncommitted changes. The review will cover both committed and uncommitted changes against the base branch."
4. Check for merge conflicts: `git diff --check origin/<base> 2>/dev/null` — if conflicts detected, STOP: "Resolve merge conflicts before review."

---

## Step 2: Scope Drift Detection

1. Read `TODOS.md`, PR description (`gh pr view --json body -q .body 2>/dev/null`), commit messages (`git log origin/<base>..HEAD --oneline`)
2. Identify **stated intent** — what was this branch supposed to accomplish?
3. Compare files changed against stated intent

Detect:
- **SCOPE CREEP:** Files changed unrelated to stated intent, "while I was in there" changes
- **MISSING REQUIREMENTS:** Items from TODOS/PR not addressed in diff

Output:
```
Scope Check: [CLEAN / DRIFT DETECTED / REQUIREMENTS MISSING]
Intent: {1-line summary}
Delivered: {1-line summary of what diff does}
```

---

## Step 3: Get the Diff

```bash
git fetch origin <base> --quiet
git diff origin/<base>
```

---

## Step 4: Universal Review Checklist

These checks apply to ALL platforms. Run them on every review.

- [ ] No hardcoded secrets, API keys, or credentials
- [ ] No TODO/FIXME/HACK without tracking issue
- [ ] Error handling at system boundaries (network, disk, user input)
- [ ] No N+1 queries or unbounded loops
- [ ] Logging for debuggability without leaking PII
- [ ] No dead code or unused imports added
- [ ] Consistent naming conventions
- [ ] No debug/test code left in (print statements, console.log, test URLs)

---

## Step 5: Platform-Specific Quick Checks

Run the top 5 highest-impact checks per detected platform. These catch the most common bugs without duplicating the full `/ios-audit`, `/android-audit`, `/web-audit`, `/backend-audit` audits.

### iOS (if Swift/ObjC files changed)
- [ ] No force unwraps (`!`) without safety comment
- [ ] `@MainActor` isolation correct — no UI updates off main thread
- [ ] No retain cycles — closures capturing self need weak/unowned
- [ ] `Sendable` conformance for concurrent code (Swift 6)
- [ ] No blocking calls on main thread
- [ ] SwiftUI state consistency — `@State` for view-local, `@StateObject` for owned VMs, `@ObservedObject` for injected ones (no `@StateObject` on passed-in VMs)
- [ ] No heavy view recomposition — expensive work lifted out of `body`, memoized with `@State` or computed once
- [ ] Image loading optimized — no full-resolution decode for thumbnails, use downsampling / AsyncImage cache limits

**For full audit:** suggest `/ios-audit`

### Android (if Kotlin/Java files changed)
- [ ] No memory leaks (Activity/Context references in long-lived objects)
- [ ] No work on main thread that could cause ANR (>5s)
- [ ] Process death restoration handled (SavedStateHandle)
- [ ] Proper Coroutine scope (viewModelScope, lifecycleScope)
- [ ] Null safety — no `!!` without safety comment
- [ ] Compose recomposition efficient — stable params, keys on lists, no allocating lambdas in hot paths
- [ ] No heavy work inside Composables — derive with `remember`/`derivedStateOf`, hoist side effects to `LaunchedEffect`
- [ ] Flow collection is lifecycle-aware — `collectAsStateWithLifecycle`, not raw `collectAsState` for background-critical flows

**For full audit:** suggest `/android-audit`

### Web Frontend (if JS/TS/CSS files changed)
- [ ] No XSS vectors (dangerouslySetInnerHTML, v-html, innerHTML)
- [ ] Bundle size impact — no large dependencies for small features
- [ ] Loading states and error boundaries present
- [ ] Accessibility — ARIA labels on interactive elements
- [ ] No layout shifts — dimensions on images/embeds

**For full audit:** suggest `/web-audit`

### Backend (if server files changed)
- [ ] SQL injection prevention — parameterized queries only
- [ ] Authentication/authorization on every endpoint
- [ ] Input validation at API boundary
- [ ] No N+1 queries — check ORM eager loading
- [ ] API response doesn't leak internal data

**For full audit:** suggest `/backend-audit`

### API Contract (if specs changed)
- [ ] Breaking changes flagged (removed fields, changed types)
- [ ] Error response format consistent

**For full audit:** suggest `/api-review`

---

## Step 6: Review Output

For each finding:
```
[SEVERITY] file:line — description
  WHY: what breaks or degrades
  FIX: specific recommendation
  AUTO: yes/no (can this be auto-fixed?)
```

Severity levels:
- **P0 BLOCK** — Must fix before merge: SQL injection, XSS, hardcoded secrets, crash on happy path, data loss, auth bypass, unhandled null on critical path
- **P1 WARN** — Should fix before merge: N+1 queries, missing error handling on network calls, memory leaks, missing accessibility labels, broken dark mode, missing input validation
- **P2 NOTE** — Consider fixing: naming inconsistencies, missing comments on complex logic, suboptimal but working patterns, minor style issues
- **ASK** — Needs user judgment. Present options via AskUserQuestion

---

## Step 7: Auto-Fix

For findings marked `AUTO: yes`:
- Dead code removal
- Missing error handling (add basic try/catch)
- Unused imports
- Obvious null safety fixes

Ask via AskUserQuestion: "I found N auto-fixable issues. Fix them now?"
- A) Fix all
- B) Show me first
- C) Skip

---

## Step 8: Summary

```
REVIEW SUMMARY
═══════════════════════════════════
Branch: {branch} → {base}
Platform(s): {detected}
Files changed: N
Lines: +X / -Y

FINDINGS
  P0 (block): N
  P1 (warn):  N
  P2 (note):  N
  Auto-fixed: N

PLATFORM DEEP DIVES RECOMMENDED:
  {list /ios-audit, /android-audit, /web-audit, /backend-audit if applicable and not yet run}

VERDICT: CLEAR / ISSUES FOUND / BLOCKED
{1-line summary of most important finding}

RECOMMENDED NEXT STEPS:
  - {if P0 found}: Fix blockers, then re-run /code-review
  - {if platform detected}: Run /ios-audit, /android-audit, /web-audit, or /backend-audit for deep audit
  - {if security concerns}: Run /security-audit for full security audit
  - {if ready}: Run /ship to create PR
```

---

## Completion
- **DONE** — Review complete, all findings reported
- **DONE_WITH_CONCERNS** — Reviewed but P1+ issues remain (user accepted)
- **BLOCKED** — P0 issues that must be fixed


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

