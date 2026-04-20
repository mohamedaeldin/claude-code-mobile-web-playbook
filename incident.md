# Incident Response — Coordinate Active Outages + Post-Mortem

An incident is happening. This skill drives the real-time response AND the post-mortem afterwards. It's the glue between `/hotfix`, `/rollback`, and `/retro`.

The rule: **one person commands, one person communicates, one person fixes — and every step is logged as it happens.**

## Triggers
Use when: "incident", "we have an incident", "major outage", "declare incident", "incident response", "post-mortem", "post mortem", "RCA", "root cause analysis", "5 whys"
Proactively suggest when: the user reports widespread impact (multiple users, multiple regions, multiple services) or says "this is an incident."

---

## Phase 1 — DECLARE (first 2 minutes)

### 1.1 Confirm it's an incident
An incident is any one of:
- User-facing outage affecting > 1% of users
- Data corruption or data loss
- Active security breach
- Complete failure of a critical capability (login, checkout, core API)

If it's just a noisy alert or single-user issue, this isn't an incident — it's a bug. Use normal flow.

### 1.2 Assign roles
- **Incident Commander (IC)** — runs the call, makes decisions, protects focus. NOT the person coding the fix.
- **Communications Lead** — posts updates to status page, Slack, stakeholders. One voice.
- **Fix Lead** — actually writes the code / runs the commands. Reports to IC.
- **Scribe** — records a timeline as things happen (can be the IC if small team).

For a small team, one person can hold multiple roles, but NEVER IC + Fix Lead in the same human.

### 1.3 Declare severity
- **SEV 1** — full outage, critical capability down, data loss active. Wake everyone.
- **SEV 2** — significant degradation, major feature broken, 10%+ of users impacted.
- **SEV 3** — minor degradation, workaround exists, <10% impact.

Announce in the incident channel with a template:
```
🚨 INCIDENT DECLARED
Sev:        SEV 1 / 2 / 3
Title:      <one-line symptom>
Impact:     <who / how many / what can't they do>
IC:         @name
Comms:      @name
Fix:        @name
Channel:    #inc-YYYYMMDD-<slug>
Status:     Investigating
```

---

## Phase 2 — STABILIZE (next 15 minutes)

### 2.1 Stop the bleeding before you diagnose
Ask in order:
1. **Can we `/rollback`?** If a deploy preceded this, revert first, diagnose later.
2. **Can we kill a feature flag?** Fastest mitigation, no code change.
3. **Can we fail over / scale up?** For capacity, not code bugs.
4. **Can we drain a bad node / region?** For partial outages.

Mitigation is NOT the same as resolution. Mitigate first, fix later.

### 2.2 Start a running timeline
Every action gets a timestamp. The IC or Scribe keeps this updated in the incident channel:

```
14:02  Alerting fired: login-api p99 > 5s
14:03  IC declared SEV 1, comms @alice, fix @bob
14:05  @bob identified deploy at 13:58 as trigger
14:07  /rollback initiated to v2.47.3
14:12  Rollback complete, p99 dropping
14:18  p99 back to baseline, users recovering
14:25  Status: Monitoring
```

### 2.3 Communicate externally
Update the public status page:
- Investigating (you see the problem)
- Identified (you know the cause)
- Monitoring (fix is deployed, watching for recovery)
- Resolved (fully recovered)

---

## Phase 3 — RESOLVE (as long as needed)

### 3.1 Fix path
Once stabilized, write the real fix using `/hotfix`.

### 3.2 Do not close too early
An incident isn't resolved when the fix deploys — it's resolved when:
- [ ] Metrics back to baseline for 30+ minutes
- [ ] No new user reports for 15+ minutes
- [ ] Underlying root cause understood (not just the symptom)
- [ ] Status page marked Resolved

### 3.3 Final public update
```
✅ RESOLVED
Root cause:  <one line>
Impact:      <users × duration>
Fix:         <what changed>
Post-mortem: Will be published by <date>
```

---

## Phase 4 — POST-MORTEM (within 48 hours)

Run this even if the incident was small. Pattern-matching across small incidents catches big ones before they happen.

### 4.1 Write a blameless timeline
```
TIMELINE (UTC)
14:00  Deploy v2.48.0 merged to main, CI starts
13:58  (pre-deploy) Error rate 0.02% baseline
14:01  Deploy hits 50% of fleet
14:02  Login error rate climbs to 1.8%, alert fires
14:04  Slack alert posted in #oncall
14:05  IC declared, timeline starts
14:07  Hypothesis: commit abc1234 changed session handling
14:09  /rollback decision made
14:12  Rollback complete
14:18  Error rate back to 0.02%
14:50  Root cause confirmed: missing migration for sessions table
15:30  All-clear, monitoring ended
```

### 4.2 Root cause with 5 Whys
```
Problem:  Users could not log in for 16 minutes.

1. Why?  Login API returned 500.
2. Why?  Code tried to read session.device_fingerprint column.
3. Why?  Column didn't exist in production DB.
4. Why?  Migration was written but not run before the code deploy.
5. Why?  CI doesn't enforce migration order in our deploy pipeline.

Root cause: Missing CI gate requiring migrations to run before app deploy.
```

### 4.3 Impact
- Users affected: <number> (measured, not estimated)
- Revenue / SLA impact: <if applicable>
- Duration: detection → mitigation → resolution

### 4.4 What went well / what didn't
**Went well:**
- Detection time was 2 min (good monitoring)
- Rollback completed in 4 min (good tooling)

**Didn't go well:**
- No pre-deploy migration gate in CI
- Status page updated late (8 min after internal declare)
- Fix Lead was also IC (too much on one person)

### 4.5 Action items (OWN'D + DATED)
Every action item has an owner and a deadline. No "we should" — only "Alice will X by Y."

| # | Action | Owner | Due |
|---|---|---|---|
| 1 | Add CI gate: run pending migrations in staging before merge | @alice | 2026-04-27 |
| 2 | Add `GET /version` endpoint with schema version | @bob | 2026-04-30 |
| 3 | Update incident runbook: IC ≠ Fix Lead | @carol | 2026-04-22 |

### 4.6 Distribute
Post the post-mortem to wherever your org reads them (Notion page, GitHub discussion, email). Tag every action item owner.

---

## Hard rules

- **Never investigate before mitigating.** Stop the bleeding first.
- **Never have the IC code the fix.** Decision-making and coding compete for attention.
- **Never skip the timeline.** Memory lies; timestamps don't.
- **Never write a blameful post-mortem.** "Alice deployed the bad code" is useless. "Our process let a bad deploy through" is actionable.
- **Never close without action items.** A post-mortem with zero action items is a post-mortem that found nothing.

---

## Completion

- **DONE — RESOLVED** — incident closed, post-mortem scheduled
- **DONE — POST-MORTEM PUBLISHED** — action items owned and dated
- **MONITORING** — fix deployed, watching for recovery


---

## CRITICAL LESSON LEARNED — Build Verification

A fix that "deployed" is not a fix that "resolved the incident." Verify against real metrics, for a sustained window, before declaring resolved.

## CRITICAL LESSON LEARNED — Acceptance Criteria Verification

The AC for an incident is "recovery sustained for 30+ minutes AND root cause understood." Don't close early because a dashboard turned green.

## CRITICAL LESSON LEARNED — Branch Scope Discipline

Post-mortems cover the specific incident's commits/branches only. Don't drag unrelated work into the blame analysis — it dilutes the learnings.
