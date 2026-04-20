# Safe Mode On — Lock Scope for Sensitive Operations

Activate safety guardrails to prevent accidental damage. Three levels: Careful (warn on destructive commands), Freeze (lock edit scope), or Full (both).

## Triggers
Use when: "guard", "careful", "be careful", "safety mode", "freeze", "lock scope", "only edit these files", "don't touch anything else", "production mode", "don't break anything", "production debugging", "maximum careful"

---

## Activation

Ask via AskUserQuestion:
- A) **Careful** — Warn before destructive commands (git force-push, rm -rf, DROP TABLE, etc.)
- B) **Freeze** — Lock edits to a specific directory/file set (prevent scope creep)
- C) **Full Guard** — Both Careful + Freeze

---

## Careful Mode Rules

When Careful is active, **STOP AND WARN** before ANY of these:

### Git Operations
- `git reset --hard` — loses uncommitted changes
- `git push --force` / `git push -f` — overwrites remote history
- `git branch -D` — deletes branch without merge check
- `git checkout -- .` / `git restore .` — discards all changes
- `git clean -f` — deletes untracked files
- `git rebase` on shared branches — rewrites history
- Any operation on `main` / `master` / `production` / `release` branches

### Database Operations
- `DROP TABLE`, `DROP DATABASE`, `TRUNCATE`
- `DELETE` without WHERE clause
- `ALTER TABLE` that drops columns
- Migration `down` / `rollback` in production
- Any direct SQL on production database

### File Operations
- `rm -rf` on any directory
- Deleting configuration files (.env, docker-compose, CI configs)
- Overwriting files without backup

### Deployment
- Deploy to production without tests passing
- Deploy during non-business hours without explicit approval
- Deploying breaking API changes without client updates

### Warning Format:
```
GUARD WARNING
━━━━━━━━━━━━━
Command: {the command}
Risk: {what could go wrong}
Impact: {who/what is affected}
Reversible: {yes/no/partially}

Alternatives:
  - {safer approach if one exists}

Proceed? [requires explicit confirmation]
```

---

## Freeze Mode Rules

When Freeze is active, ask user for scope:
- A) Current directory only: `{cwd}`
- B) Specific directory: (user provides path)
- C) Only files changed on this branch
- D) Custom file list

Then, **BEFORE any Edit or Write operation**, check if the target file is within the frozen scope:

- **Outside scope:** STOP and warn:
  ```
  GUARD: Edit blocked
  ━━━━━━━━━━━━━━━━━━
  Target: {file path}
  Scope: {frozen scope}
  
  This file is outside the frozen edit scope.
  Use /safe-mode-off to remove the restriction, or
  explain why this edit is necessary.
  ```
- **Inside scope:** proceed normally

---

## Override Mechanism

If guard blocks a legitimate operation:
1. User must explain WHY the operation is needed
2. Claude acknowledges the override: "Override accepted: {reason}. Proceeding with {command}."
3. Log the override in conversation for audit trail

No silent overrides. Every bypass must be explicit and documented in the conversation.

---

## Auto-Activation

Guard activates automatically (Careful mode only) when:
- Working on `main`, `master`, `production`, or `release` branches
- Files in `deploy/`, `infrastructure/`, or `terraform/` are being edited
- Database migration files are being modified

Print: "Guard auto-activated (Careful mode) — you're working on sensitive files/branch."

---

## Activation Confirmation

Print status based on selected level:

```
GUARD ACTIVE
═══════════════
  Careful: {ON/OFF} — destructive ops require confirmation
  Freeze:  {ON/OFF} — edits locked to {scope or "unrestricted"}
  
Deactivate with /safe-mode-off.
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

