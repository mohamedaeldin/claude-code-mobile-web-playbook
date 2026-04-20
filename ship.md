# Ship — Automated Ship Workflow

Fully automated ship workflow: merge base, run tests, review diff, commit, push, create PR. Non-interactive by default.

## Triggers
Use when: "ship", "ship it", "create a PR", "push to main", "get it deployed", "ready to merge"
Proactively suggest when: the user says code is ready or asks about deploying.

---

**This is a NON-INTERACTIVE, FULLY AUTOMATED workflow.** The user said /ship. DO IT.

**Only stop for:**
- On base branch (abort)
- Merge conflicts that can't be auto-resolved
- Test failures caused by this branch's changes
- P0 review findings that need user judgment

**Never stop for:**
- Uncommitted changes (always include them)
- Commit message approval (auto-generate)

---

## Step 0: Detect Platform and Base Branch

```bash
git remote get-url origin 2>/dev/null
git branch --show-current
```

Determine base branch:
1. `gh pr view --json baseRefName -q .baseRefName 2>/dev/null`
2. `gh repo view --json defaultBranchRef -q .defaultBranchRef.name 2>/dev/null`
3. `git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's|refs/remotes/origin/||'`
4. Fallback: `main`

If on base branch: **ABORT** — "Ship from a feature branch."

---

## Step 1: Pre-flight

1. `git status` (never `-uall`)
2. `git diff <base>...HEAD --stat` — what's being shipped
3. `git log <base>..HEAD --oneline` — commit history

If uncommitted changes exist:
1. Show the user what's uncommitted: `git status --porcelain`
2. Ask via AskUserQuestion: "You have uncommitted changes. Include them in this ship?"
   - A) Yes, include all changes
   - B) Let me review first (show diff, then ask again)
   - C) Skip uncommitted changes (ship only committed work)
3. If A: stage tracked files only: `git add -u` (never `git add -A` — prevents accidental .env, secrets, binaries)
4. Commit: `git commit -m "chore: include uncommitted changes for ship"`

---

## Step 2: Merge Base Branch

```bash
git fetch origin <base> --quiet && git merge origin/<base> --no-edit
```

If merge conflicts: try auto-resolve simple ones. If complex, **STOP** and show conflicts.

---

## Step 3: Run Tests

Detect and run test suite:

```bash
# iOS
xcodebuild test -scheme <scheme> -destination 'platform=iOS Simulator,name=iPhone 16' -quiet 2>&1 | tail -20

# Android
./gradlew test 2>&1 | tail -20

# Web
npm test 2>&1 | tail -30

# Backend
# (auto-detect: rails test, pytest, go test, cargo test)
```

**If no test infrastructure exists:** Note in PR body: "No automated tests found. Consider adding tests with /qa." Continue without blocking.

**If tests fail:**
- Check if failure is caused by this branch (in-branch) or pre-existing
- In-branch failures: **STOP** — "Fix your tests before shipping"
- Pre-existing failures: Note in PR body, continue

---

## Step 4: Quick Review

Run a focused pre-landing review (subset of /code-review):
- Security: no secrets, no injection vectors
- Scope: no accidental files (node_modules, .env, large binaries)
- Quality: no obvious bugs, no debug code left in
- Secrets: `grep -rn --include="*.swift" --include="*.kt" --include="*.ts" --include="*.js" --include="*.py" --include="*.go" -E '(api[_-]?key|secret|password|token)\s*[:=]' . 2>/dev/null | grep -v node_modules | grep -v test | grep -v mock | head -10`
- Accidental files: `git diff --cached --name-only | grep -E '\.(env|pem|key|p12|keystore|log)$'` — if found, STOP.

If P0 issues found: **STOP** with AskUserQuestion
If P1 or below: note in PR body, continue

---

## Step 5: Push and Create PR

```bash
git push -u origin $(git branch --show-current)
```

Generate PR title and body from the diff:

```bash
gh pr create --title "<title>" --body "$(cat <<'EOF'
## Summary
<bullet points summarizing changes>

## Platform
<detected platform(s)>

## Test Results
<pass/fail counts>

## Review Notes
<any P1/P2 findings noted>

## Test Plan
- [ ] <testing checklist>

Generated with Claude Code
EOF
)"
```

---

## Step 6: Output

```
SHIPPED
═══════════════════════════════════
Branch: {branch} → {base}
PR: {URL}

Changes: {N files}, +{added} / -{removed}
Tests: {passed}/{total} passing
Review: {CLEAN / N findings noted}

STATUS: DONE
```

Print the PR URL.


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

