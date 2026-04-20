# Deliver — End-to-End Task Lifecycle Orchestrator

One command that takes any task — feature, bug, hotfix, design, refactor, migration, investigation, or docs — and walks it from idea to shipped using the right subset of your skills, with gates between every phase.

The rule: **detect the task type, compose the right sequence, enforce the gates. No phase runs if the previous phase's gate failed.**

## Triggers
Use when: "deliver this", "full cycle", "end to end", "take this to production", "complete this task", "run the full pipeline", "deliver", "ship this feature/bug/fix"
Proactively suggest when: the user describes a task without specifying the flow, or says "just do it properly."

---

## Step 0: Diagnose the Task

Ask the user via AskUserQuestion — three questions, in order:

### Question 1 — Task type
- **Feature** — new capability or user-visible functionality
- **Bug** — something broken in staging or reported by a user
- **Hotfix** — production is actively broken NOW
- **Design** — UI refresh, new component, visual change only
- **Refactor** — internal restructure with no behavior change
- **Migration** — schema, data, or dependency migration
- **Investigation** — figure out what's happening, no code yet
- **Docs** — documentation update only

### Question 2 — Platform (auto-detect first)

```bash
ls *.xcodeproj *.xcworkspace 2>/dev/null && echo "ios"
ls build.gradle* 2>/dev/null && echo "android"
[ -f package.json ] && echo "web"
[ -f go.mod ] || [ -f pyproject.toml ] || [ -f Gemfile ] && echo "backend"
```

Confirm with user: iOS / Android / Web / Backend / **Multi** (mobile + web, or web + backend, etc.)

### Question 3 — Scope
- **Small** — hours of work, < 200 lines, single file or small area
- **Medium** — days of work, multiple files, single feature area
- **Large** — weeks of work, architectural impact, multi-platform

---

## Step 1: Compose the Flow

Pick the flow from the catalog below based on (task, platform, scope). Present it to the user before executing:

```
PROPOSED FLOW
═══════════════════════════════════════
Task:      {type}
Platform:  {platform}
Scope:     {scope}

Phase 1 — THINK
  /brainstorm → /scope-review → /plan-review

Phase 2 — DESIGN
  /design-review

Phase 3 — BUILD
  /ios-dev

Phase 4 — REVIEW
  /code-review → /ios-audit

Phase 5 — VERIFY
  /qa → /mobile-quality

Phase 6 — SHIP
  /safe-mode-on → /ship → /release-ios → /safe-mode-off

Phase 7 — AFTER
  /retro

GATES (pipeline stops here if failed):
  - After THINK:    plan approved
  - After REVIEW:   no P0 findings
  - After VERIFY:   no P0/P1 bugs open
  - After SHIP:     smoke test passes on store build
```

Ask: "Proceed with this flow? (yes / skip phases / redesign)"

---

## Step 2: Execute Phase by Phase

For each phase:
1. **Announce:** "▶ Phase N — {name}. Running /{skill}."
2. **Invoke the skill.** Load its content; execute its steps.
3. **Report:** brief summary of what the skill produced.
4. **Check the gate** (see rules below).
5. **If gate passes → next phase. If gate fails → STOP, ask user.**

Never silently skip a gate. Never move to the next phase if the previous one's gate failed — that's the entire point of having the orchestrator.

---

## Optional THINK-phase reinforcements

Before committing to a plan in any flow below, you can insert:
- **`/research`** — if the task involves a technology / pattern / library choice that's new to the codebase. Produces a sourced comparison with confidence level.
- **`/challenge`** — after `/plan-review` is "approved," run `/challenge` to attack the plan before building. Catches hidden assumptions and one-way doors.

These are inserted AFTER `/plan-review` and BEFORE moving to DESIGN / BUILD. Skip them for small-scope tasks; they earn their weight on medium+ work.

---

## Shortcut — single-skill reviews

Most flows below list each review skill individually (e.g., `/code-review → /ios-audit → /qa → /security-audit → /a11y-audit → /dependencies`). If you want to compress those review phases into one call, substitute `/auto-fix` (which runs every applicable review AND applies fixes) in place of the individual review skills.

Trade-off: `/auto-fix` is faster but less granular — gates fire per-category, not per-skill. Use individual skills when you want to stop between reviews to decide.

---

## Step 3: Flow Catalog

### FEATURE × iOS / Android (mobile)
```
THINK:    /brainstorm (if large) → /scope-review → /plan-review
DESIGN:   /design-system (only if tokens change) → /design-review
BUILD:    /ios-dev OR /android-dev
REVIEW:   /code-review → /ios-audit OR /android-audit
VERIFY:   /qa → /mobile-quality → /a11y-audit (if customer-facing)
SHIP:     /safe-mode-on → /ship → /release-ios OR /release-android → /safe-mode-off
AFTER:    /retro
```

### FEATURE × Web
```
THINK:    /brainstorm (if large) → /scope-review → /plan-review
DESIGN:   /design-system (if tokens change) → /design-review
BUILD:    /web-dev
REVIEW:   /code-review → /web-audit
VERIFY:   /qa → /a11y-audit → /benchmark
SHIP:     /safe-mode-on → /ship → /deploy → /safe-mode-off
AFTER:    /monitor (confirm observability) → /retro
```

### FEATURE × Backend
```
THINK:    /brainstorm (if large) → /scope-review → /plan-review → /api-review
BUILD:    /backend-dev (+ /migrate if schema change)
REVIEW:   /code-review → /backend-audit → /api-review
VERIFY:   /qa → /security-audit → /benchmark
SHIP:     /safe-mode-on → /ship → /deploy → /safe-mode-off
AFTER:    /monitor → /retro
```

### FEATURE × Multi-platform
```
THINK:    /brainstorm → /scope-review → /plan-review → /api-review (if backend involved)
DESIGN:   /design-system → /design-review (all platforms)
BUILD:    /backend-dev (first, if API change) → /ios-dev + /android-dev + /web-dev (one at a time)
REVIEW:   /code-review per platform → /ios-audit + /android-audit + /web-audit + /backend-audit
VERIFY:   /qa → /mobile-quality → /a11y-audit → /security-audit → /benchmark
SHIP:     /safe-mode-on → /ship → /release-ios + /release-android + /deploy → /safe-mode-off
AFTER:    /monitor → /retro
```

### BUG × Mobile
```
INVESTIGATE: /investigate
BUILD:       /ios-dev OR /android-dev (MUST include regression test)
REVIEW:      /code-review → /ios-audit OR /android-audit
VERIFY:      /qa (confirm regression test fails before + passes after)
SHIP:        /ship → /release-ios OR /release-android
AFTER:       /retro (or /incident if bug caused an outage)
```

### BUG × Web / Backend
```
INVESTIGATE: /investigate
BUILD:       /web-dev OR /backend-dev (MUST include regression test)
REVIEW:      /code-review → /web-audit OR /backend-audit
VERIFY:      /qa
SHIP:        /ship → /deploy
AFTER:       /retro (or /incident if bug caused an outage)
```

### HOTFIX (production broken)
```
STABILIZE:  /hotfix  (handles triage, mitigate, fix, staging, deploy, monitor)
IF NEEDED:  /rollback  (if mitigation > fix)
COORDINATE: /incident  (real-time command + post-mortem)
AFTER:      /retro
```

`/hotfix` is itself a mini-orchestrator — don't duplicate its internal gates here.

### DESIGN × Mobile
```
DESIGN:   /design-system → /design-review
BUILD:    /ios-dev OR /android-dev
REVIEW:   /code-review → /ios-audit OR /android-audit
VERIFY:   /qa → /a11y-audit → /mobile-quality
SHIP:     /ship → /release-ios OR /release-android
AFTER:    /retro
```

### DESIGN × Web
```
DESIGN:   /design-system → /design-review
BUILD:    /web-dev
REVIEW:   /code-review → /web-audit
VERIFY:   /qa → /a11y-audit → /benchmark (ensure no CLS regression)
SHIP:     /ship → /deploy
AFTER:    /retro
```

### REFACTOR
```
THINK:    /plan-review (MANDATORY — refactors need a plan)
BUILD:    /{platform}-dev (per area)
REVIEW:   /code-review → /{platform}-audit
VERIFY:   /qa (heavy — tests must prove behavior unchanged)
SHIP:     /ship → /release-* OR /deploy
AFTER:    /retro
```

### MIGRATION (schema / data / dependency)
```
PLAN:     /plan-review → /migrate (design expand-backfill-contract)
BUILD:    /backend-dev + /migrate
REVIEW:   /code-review → /backend-audit
VERIFY:   /qa → /security-audit (if PII) → /benchmark (big tables)
SHIP:     /safe-mode-on → /ship → /deploy (watch metrics 30min) → /safe-mode-off
AFTER:    /monitor → /retro
```

### INVESTIGATION (no code yet)
```
/investigate

Then based on findings, re-enter /deliver with the right task type.
(Often: "turns out it's a hotfix" or "turns out it's not actually broken")
```

### DOCS
```
BUILD:    /docs
REVIEW:   /code-review (if docs live in repo)
SHIP:     /ship (for the PR) → /deploy (if docs site auto-builds)
```

### DEPENDENCY UPGRADE
```
AUDIT:    /dependencies
PLAN:     /plan-review (only if major version bump)
BUILD:    upgrade + fix breakage
REVIEW:   /code-review → /security-audit
VERIFY:   /qa → /benchmark (regressions?)
SHIP:     /ship → /deploy
AFTER:    /monitor (for 24h) → /retro
```

---

## Step 4: Gates (hard-stop conditions)

A gate is a pass/fail check between phases. The flow STOPS and returns to the user if ANY gate fails.

| Gate | Fails if |
|---|---|
| **After THINK** | Plan is vague, no AC defined, user hasn't approved |
| **After DESIGN** | Figma missing, tokens not defined, RTL or dark mode not handled |
| **After BUILD** | Compile fails, self-audit has a red box |
| **After REVIEW** | Any P0 finding from code-review or platform audit |
| **After VERIFY** | Any P0/P1 bug still open, mobile-quality hard gate failed, CVE open |
| **After SHIP** | Canary unhealthy, smoke test fails on store build, error rate spiked |

On a failed gate:
- Report the failure clearly
- Suggest the fix path (usually: "run /investigate, then resume")
- Do NOT move forward silently

---

## Step 5: Scope-Based Shortcuts

### Small scope (< 1 day)
Skip THINK phase when scope is obvious. Start at BUILD.
```
Minimal: /{platform}-dev → /code-review → /qa → /ship
```

### Large scope (weeks)
Add checkpoints. At end of each day:
- Integration build
- Smoke test
- Status update to team via `/docs` (runbook update) or Slack

Don't wait until the end to find out the pieces don't fit.

---

## Step 6: Output — Final Summary

When the full flow completes (or stops at a gate):

```
DELIVERY REPORT
═══════════════════════════════════════
Task:      {type}
Platform:  {platform}
Scope:     {scope}
Duration:  {start → end}

PHASES RUN
  ✅ THINK     — plan approved
  ✅ DESIGN    — Figma reviewed, tokens updated
  ✅ BUILD     — N files, M lines
  ✅ REVIEW    — code-review CLEAR, ios-audit CLEAR
  ✅ VERIFY    — qa (K bugs fixed), mobile-quality PASS
  ✅ SHIP      — PR #1234 merged, TestFlight build 42
  ⬜ AFTER     — retro pending

METRICS
  Files:       N new, M modified
  Tests:       K regression tests added
  Bugs fixed:  J
  CVEs:        0 introduced

NEXT STEPS
  - [user action] approve TestFlight build for App Store
  - [user action] schedule /retro next Monday
```

On a failed gate:

```
DELIVERY HALTED
═══════════════════════════════════════
Halted at:   REVIEW phase
Reason:      code-review found 2 P0 issues
Issues:      - file.swift:42 force-unwrap on network response
             - file.swift:88 blocking call on main thread
Next step:   Fix the P0s, then resume /deliver
```

---

## Hard rules

- **Never skip a gate.** Gates exist for a reason.
- **Never run phases out of order.** The order is a dependency chain, not a preference.
- **Never proceed if a gate fails.** Go back, fix, verify, resume.
- **Never run skills in parallel that share the main context** (e.g., two `/*-dev` skills simultaneously will thrash the files). Sequential only for build phases.
- **Never skip `/retro` on large tasks.** If you don't learn from it, the next large task will hit the same rocks.
- **Never claim DONE without the final summary.** Report is part of the deliverable.

---

## When to NOT use /deliver

- A 5-minute fix (just run `/code-review` + `/ship`)
- You're exploring / researching (use `/investigate` or `/brainstorm`)
- You already know exactly what you need — just invoke specific skills

`/deliver` earns its weight on medium-to-large tasks where coordination is the hard part.

---

## Completion

- **DONE — DELIVERED** — all phases passed, final summary produced, retro scheduled
- **HALTED — GATE FAILED** — stopped at phase X, reason + next step reported
- **DONE — PARTIAL** — user explicitly opted to skip later phases (e.g., "ship without retro")


---

## CRITICAL LESSON LEARNED — Build Verification

The orchestrator doesn't compile anything — each invoked skill does. But the orchestrator MUST verify that each skill's compile/test step passed before moving on. Never accept a skill's handoff without checking its own build verification step ran green.

## CRITICAL LESSON LEARNED — Acceptance Criteria Verification

The orchestrator checks that every acceptance criterion has a verification step somewhere in the flow. If an AC has no phase that exercises it, flag it before BUILD even starts — don't discover it at SHIP.

## CRITICAL LESSON LEARNED — Branch Scope Discipline

The orchestrator operates on a single feature branch. Never cross branches mid-flow. If the task legitimately spans branches (e.g., port to release branch after main merge), that's a NEW invocation of `/deliver`, not a phase of the existing one.
