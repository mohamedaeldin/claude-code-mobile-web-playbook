# Hotfix — Emergency Production Fix Fast-Lane

Production is broken. A user-facing flow is failing, data is being corrupted, or a security hole is open. This skill is the fast-lane: it skips non-essential phases but enforces the safety bar that matters for an emergency fix.

The rule: **fix fast, but never without a regression test and never without a rollback plan.**

## Triggers
Use when: "hotfix", "production is broken", "users can't log in", "emergency fix", "urgent production bug", "prod is down", "critical bug", "fix this now"
Proactively suggest when: the user says "urgent", "emergency", "customers reporting", or references a live outage.

---

## Step 0: Triage — Is This Actually a Hotfix?

A hotfix is ONLY for:
- User-facing flow broken (can't log in, can't check out, can't load the app)
- Data corruption in progress
- Active security vulnerability being exploited
- Crash affecting > 1% of users
- Legal / compliance issue requiring immediate action

NOT a hotfix:
- UI bug with workaround
- Edge-case error on rare input
- Performance degradation (unless catastrophic)
- "We want this feature by end of day"

If it's not a real emergency, STOP and use the normal flow (`/code-review` → `/qa` → `/ship`).

Ask the user to confirm severity:
- **P0** — Active outage or data loss. Proceed immediately.
- **P1** — Critical degradation. Proceed but confirm with on-call lead.
- **P2 or lower** — Use normal flow, not hotfix.

---

## Step 1: Stabilize First, Fix Second

Before writing any code, ask:
1. **Can we rollback?** If yes, `/rollback` is faster than coding a fix.
2. **Can we toggle a feature flag off?** Faster still.
3. **Can we scale up / fail over?** If this is capacity, not code.

Only if none of those apply, proceed to code the fix.

Activate `/safe-mode-on` to lock scope to the bug file + its tests.

---

## Step 2: Understand the Bug

1. Gather evidence:
   - Error logs (link or paste)
   - User reports / reproduction steps
   - When did it start? (last deploy? config change? external service?)
   - Scope — % of users, which regions, which platforms

2. Reproduce locally IF possible. If you can't reproduce in < 10 minutes, note it and proceed cautiously — the fix is hypothesis-based.

3. Identify root cause with `/investigate` if the cause isn't obvious.

---

## Step 3: Write the Fix

Rules:
- **Minimal diff.** Fix the bug, nothing else. No refactors, no style changes, no adjacent "while I'm here" cleanups.
- **Revertable.** A future on-call engineer should be able to `git revert <sha>` without breakage.
- **Behind a flag** (if the risk of the fix is non-trivial). `hotfix_20260420_login_fix = true`.

After the fix:
- Read the diff yourself line-by-line.
- Verify you changed nothing beyond the root cause.

---

## Step 4: Regression Test (MANDATORY — NO EXCEPTIONS)

Write a test that:
- Fails against the current broken code
- Passes against your fix
- Names the bug so future engineers find it (`test_login_fails_when_email_has_uppercase_bug_20260420`)

If you skip this step, you WILL ship this bug again. No regression test → not a hotfix → back to normal flow.

---

## Step 5: Verify on Staging

Deploy to staging first. Hit the exact failure path with the exact conditions from production.

If you can't reproduce on staging (data differs, config differs), you don't fully understand the bug. Pause.

Never skip staging for "the fix is obvious." Obvious fixes cause second outages.

---

## Step 6: Ship with Rollback Ready

1. Deploy to a canary / 1% of production first if your infra supports it.
2. Watch the error rate for 5–10 minutes before full rollout.
3. Have the rollback command typed in a separate shell, ready to hit ENTER.

Rollback command must be pre-tested in `/rollback` — don't invent it in the moment.

---

## Step 7: Communicate

Write a status update NOW (even if the fix isn't deployed yet):
```
INCIDENT: <one-line title>
IMPACT:   <who / what / how many>
STATUS:   Investigating / Mitigating / Fixed / Monitoring
ROOT CAUSE: <if known>
FIX:      <link to PR / commit>
ETA:      <honest estimate>
ROLLBACK: ready (<command>)
```

Post this wherever the team watches — Slack incident channel, status page, whatever.

Update every 15–30 minutes until resolved.

---

## Step 8: After the Fire

Within 24 hours, run `/incident` to produce a post-mortem:
- Timeline (when did it start, when detected, when mitigated, when resolved)
- Root cause (5 whys)
- Detection gap (why didn't we catch this sooner?)
- Action items (what changes so this doesn't repeat)

Then run `/safe-mode-off` to restore normal scope.

---

## Hard rules — never break these

- **Never skip the regression test.** Not for "it's obvious," not for "we're in a rush." The test IS the rush protection.
- **Never deploy straight to production.** Staging first, canary second, full third. Always.
- **Never bundle unrelated changes into a hotfix.** A hotfix PR with > 20 lines of non-test code needs justification.
- **Never hotfix without a rollback plan.** Write it down, test it, keep it in the other terminal.
- **Never skip the post-mortem.** If you don't learn from this, you will do it again.

---

## Completion

- **DONE — RESOLVED** — fix deployed, error rate back to baseline, post-mortem scheduled
- **DONE — MITIGATED** — user impact stopped (via rollback/flag), fix still to come
- **ESCALATED** — handed off to a specialist; note who + why


---

## CRITICAL LESSON LEARNED — Build Verification

Production fixes that "worked locally" have crashed countless sites. Always deploy to staging, hit the exact failure path, and only then promote.

## CRITICAL LESSON LEARNED — Acceptance Criteria Verification

For a hotfix, the AC is "the reported bug no longer reproduces in production." That's verified by actually reproducing in staging first, then watching the error rate after deploy. Never claim fixed from code review alone.

## CRITICAL LESSON LEARNED — Branch Scope Discipline

Hotfixes often branch from a release tag, not develop. Push to the hotfix branch only, cherry-pick to develop separately. Never force-push to shared release branches.
