# Auto-Fix — Full Review + Apply Fixes

Run every applicable review skill across the changed code AND apply fixes as you go. Produces a clean, ship-ready diff or a prioritized blocker list when something can't be auto-fixed.

The rule: **don't just report what's wrong — fix it, verify nothing regressed, and flag only what actually needs a human decision.**

This is `/full-review` + fixing enabled + cross-dimensional (code, bugs, security, perf, a11y, deps — not just bugs like `/qa`).

## Triggers
Use when: "auto-fix", "review and fix", "fix everything", "polish this", "clean the diff", "make it ship-ready", "fix all issues", "sweep the code"
Proactively suggest when: the user finishes a feature and asks "is it ready to merge?" or "what's left before I ship?"

---

## Step 0: Detect Platform + Scope

```bash
# Platform
ls *.xcodeproj *.xcworkspace 2>/dev/null && echo "ios"
ls build.gradle* 2>/dev/null && echo "android"
[ -f package.json ] && echo "web-or-node"
[ -f go.mod ] || [ -f pyproject.toml ] || [ -f Gemfile ] && echo "backend"

# Scope: what files to fix — only the diff, not the world
git fetch origin <base> --quiet
git diff --name-only origin/<base>...HEAD 2>/dev/null | head -30
```

Fix scope is **only** the current branch's diff vs base. Never edit files outside the diff — that's scope creep.

---

## Step 1: Ask the User for Mode

Via AskUserQuestion — picks how aggressive the fixing is:

- **A) Critical only** — fix P0 (crashes, security, data loss). Flag everything else.
- **B) Standard (Recommended)** — fix P0 + P1 (bugs, N+1s, missing error handling, a11y violations). Flag P2.
- **C) Aggressive** — fix everything auto-fixable down to P2 (naming, dead code, minor perf). Flag only things needing human judgment.
- **D) Report only** — run every review, no edits (identical to `/full-review`).

Default to B if the user doesn't specify.

---

## Step 2: Activate Safety Mode

Before fixing, invoke `/safe-mode-on` internally — lock scope to the diff files + their tests. Prevents accidental edits elsewhere.

---

## Step 3: Run Every Applicable Review — Sequentially

Skills fire in this order (skip ones that don't apply to the detected platform):

### Universal (always)
1. `/code-review` — structural diff audit
2. `/qa` (same mode as chosen above) — bug hunt + regression tests
3. `/security-audit` — secrets, OWASP, auth, input validation
4. `/dependencies` — CVEs only (upgrade proposals go to a separate PR)

### Platform-specific (based on Step 0)
5. `/ios-audit` — if .swift files changed
6. `/android-audit` — if .kt/.java files changed
7. `/web-audit` — if .tsx/.jsx/.vue/.svelte files changed
8. `/backend-audit` — if route/controller/migration files changed
9. `/api-review` — if openapi.yaml / *.proto / schema.graphql changed

### UI-involved (customer-facing)
10. `/a11y-audit` — automated scan only (manual screen-reader testing stays manual)

### Optional (only if mode = Aggressive)
11. `/benchmark` — perf deltas vs base
12. `/mobile-quality` (if mobile) — lifecycle, network, devices

Record every finding with: severity (P0/P1/P2), file:line, category (bug/security/a11y/perf/style), AND whether it's fixable automatically.

---

## Step 4: Classify Every Finding

For each finding from Step 3, tag one of:

| Tag | Meaning | Action |
|---|---|---|
| **AUTO** | Safe to fix without judgment | Fix it now |
| **ASSISTED** | Fix is obvious but touches logic | Propose fix, show diff, apply on user approval |
| **HUMAN** | Needs judgment (API change, naming, behavior) | Flag only, list in report |

Examples:

| Finding | Tag | Why |
|---|---|---|
| Missing error handling on `await fetch(...)` | AUTO | Wrap with try/catch that surfaces the error |
| Force unwrap on network response | AUTO | Convert to `if let` with error path |
| Hardcoded bilingual ternary | AUTO | Replace with StringManager / localizeString |
| SQL query uses string concat | AUTO (critical) | Parameterize |
| N+1 query in list endpoint | ASSISTED | Usually obvious which association to eager-load, confirm scope |
| Missing index on frequently-queried column | ASSISTED | Propose migration, confirm column is the right one |
| Renaming a public API method | HUMAN | Breaks callers, product decision |
| Changing error message wording | HUMAN | User-facing copy decision |
| A race condition in concurrent code | HUMAN | Requires architectural decision |

---

## Step 5: Apply Fixes — One Category at a Time

### Rules for safe auto-fix:
- **One category per commit.** Don't mix "fix security bugs" with "fix a11y labels" in one commit.
- **Compile after every category.** Never stack fixes on a broken build.
- **Run tests after every category.** Regression > productivity.
- **Add a regression test for every bug fix.** If you can't write one, the bug isn't fixed — it's moved.

### Category order (do in this sequence):
1. **P0 security fixes** — SQL injection, XSS, hardcoded secrets, auth bypass. Stop-the-world.
2. **P0 bugs** — crashes, data loss, broken happy paths.
3. **P1 bugs** — error handling, N+1 queries, race conditions (if auto-fixable).
4. **Localization** — hardcoded strings → StringManager/i18n.
5. **A11y** — missing alt text, labels, roles, keyboard traps.
6. **Style / cleanup** — dead code, unused imports, deprecated APIs (only if mode = Aggressive).

### Per-category loop:
```
for category in ordered_categories:
    apply_auto_fixes(category.findings_tagged_AUTO)
    if build_fails:
        revert last batch
        flag as ASSISTED
        continue to next category
    run_tests()
    if tests_fail:
        revert last batch
        halt, report what broke
    commit "fix({category}): auto-fix N findings"
```

### ASSISTED fixes
For each, show the diff and ask: **Apply / Skip / Customize**. Batch similar ones when possible (e.g., "I found 6 N+1 queries — review them one by one?").

### HUMAN fixes
No edit. Added to the final report's "needs your call" section.

---

## Step 6: Re-Run Reviews

After all fixes are applied, re-run the key reviews to verify clean state:

- `/code-review` again — zero NEW findings
- `/qa` again (Quick mode) — zero NEW failing tests
- `/security-audit` again — zero P0 security findings open

If any of these re-runs fail, go back to Step 5 for the affected category.

---

## Step 7: Deactivate Safety Mode

Run `/safe-mode-off` — restore normal scope.

---

## Step 8: Final Report

```
AUTO-FIX REPORT
═══════════════════════════════════════
Platform:       {detected}
Mode:           Critical / Standard / Aggressive / Report-Only
Diff scope:     {n} files

FINDINGS FOUND: {n}
  P0:   {n}  (all must be addressed)
  P1:   {n}
  P2:   {n}

FIXED AUTOMATICALLY: {n} commits
  fix(security): {n} findings     — SHA {short}
  fix(bugs):     {n} findings     — SHA {short}
  fix(i18n):     {n} findings     — SHA {short}
  fix(a11y):     {n} findings     — SHA {short}
  fix(cleanup):  {n} findings     — SHA {short} (Aggressive mode only)

REGRESSION TESTS ADDED: {n}

ASSISTED — APPLIED WITH APPROVAL: {n}
ASSISTED — SKIPPED: {n}

NEEDS YOUR CALL (HUMAN): {n}
  - {location} — {why it needs judgment} — suggested approach
  - ...

TEST SUITE: {passed}/{total}  (was {baseline} before)

VERDICT: READY TO SHIP / NEEDS HUMAN DECISIONS / BLOCKED
```

---

## Hard rules

- **Never fix a build-broken state.** If compile fails, revert and flag.
- **Never skip tests after a fix batch.** Regression protection is the whole point.
- **Never auto-fix anything that changes a public API shape.** Flag as HUMAN.
- **Never bundle categories into one commit.** Clean history = easy bisect later.
- **Never auto-fix copy / wording / naming.** Those are human decisions.
- **Never run outside the diff scope.** Safe-mode-on prevents this, but double-check.
- **Never declare DONE without re-running reviews.** Every fix can introduce a new bug.

---

## When NOT to use /auto-fix

- One-line typo fix (just edit + commit)
- Exploring the code (use `/investigate` or `/code-review` in report mode)
- You already know exactly what you want to change (just do it)

`/auto-fix` earns its weight when you've finished a feature and want every cross-cutting concern (bugs, security, a11y, perf, deps, code smell) swept in one pass before review.

---

## Completion

- **DONE — READY** — all auto-fixable findings addressed, re-runs clean, HUMAN list is empty or reviewed
- **DONE — HUMAN DECISIONS PENDING** — auto-fixes applied, N human calls awaiting
- **HALTED — BUILD BROKE** — stopped mid-sweep, rolled back; report explains where


---

## CRITICAL LESSON LEARNED — Build Verification

An "auto-fix" that broke the build isn't a fix — it's a new bug. Compile + run tests after every fix batch. Revert on failure.

## CRITICAL LESSON LEARNED — Acceptance Criteria Verification

"Findings fixed" ≠ "feature still works." After auto-fixing, manually exercise the critical user flow. Auto-fixes can change behavior in ways unit tests miss.

## CRITICAL LESSON LEARNED — Branch Scope Discipline

Auto-fix ONLY the current branch's diff. Files outside the diff aren't your scope — fixing them creates phantom changes that confuse reviewers and pollute bisect.
