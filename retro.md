# Retrospective — What Worked, What Didn't, What's Next

Run a retrospective on recent work. Review git history, identify patterns, and create actionable improvements.

## Triggers
Use when: "retro", "retrospective", "what did we ship", "weekly review", "sprint review"

---

## Step 1: Gather Data

```bash
# Recent commits (last 7 days by default)
git log --oneline --since="7 days ago" --all
git log --oneline --since="7 days ago" --all --format="%h %s" | wc -l

# Files most changed
git log --since="7 days ago" --name-only --format="" | sort | uniq -c | sort -rn | head -15

# Contributors
git log --since="7 days ago" --format="%aN" | sort | uniq -c | sort -rn

# Branches merged
git log --since="7 days ago" --merges --oneline

# Lines changed
git log --since="7 days ago" --stat --format="" | tail -1
```

---

## Step 2: Categorize Changes

Group commits into:
- **Features** — new functionality
- **Fixes** — bug fixes
- **Refactors** — code improvements without behavior change
- **Tests** — test additions/improvements
- **Docs** — documentation changes
- **Chores** — dependencies, CI/CD, tooling

---

## Step 3: Identify Patterns

### What Went Well
- Features shipped on time
- Bugs fixed quickly
- Clean code reviews
- Good test coverage

### What Didn't Go Well
- Hotfixes needed after merge
- Tests that were skipped or broken
- Large PRs that were hard to review
- Repeated work in the same area (churn)

### Hot Spots
Files changed more than 3 times in the period — these are either actively developed or unstable:
```bash
git log --since="7 days ago" --name-only --format="" | sort | uniq -c | sort -rn | awk '$1 > 3'
```

---

## Step 4: Metrics

```
RETRO
═══════════════════════════════════
Period: {start} → {end}

VELOCITY
  Commits: {count}
  Features: {count}
  Fixes: {count}
  Lines: +{added} / -{removed}

HOT SPOTS (files changed >3x)
  {file} — {count} changes
  ...

PATTERNS
  Went well:
    - {observation}
  Needs improvement:
    - {observation}

ACTION ITEMS
  1. {concrete action with owner}
  2. {concrete action with owner}
  3. {concrete action with owner}
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

