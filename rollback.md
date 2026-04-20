# Rollback — Revert a Broken Release

Something shipped broke production. This skill drives the fastest safe path back to the last known good state.

The rule: **speed is the point, but never roll back blindly — always verify the rollback landed and the metric actually recovered.**

## Triggers
Use when: "rollback", "revert the release", "undo the deploy", "roll back production", "revert to previous version", "the release is broken", "undo this"
Proactively suggest when: the user reports a regression immediately after a deploy or release submission.

---

## Step 0: Confirm Rollback Is the Right Move

Quick decision tree:
- **Error spike + recent deploy?** → Rollback is usually right.
- **Error spike + no recent deploy?** → Probably external (third party, infra). Rollback won't help — go to `/incident`.
- **Data corruption in flight?** → Rollback may make it WORSE (schema change, migration). Stop and `/incident`.
- **Security vulnerability?** → Rollback only if the vuln was introduced in the latest release. If it's pre-existing, rollback doesn't fix it.

Ask the user to confirm:
- What's the symptom?
- What was the last deploy / release?
- Is there schema / data migration in the release being rolled back? (If yes, rollback requires extra care.)

---

## Step 1: Pick the Rollback Type

### Web / Backend (server-side deploys)
- **Platform rollback** — Vercel / Netlify / Fly / Heroku have a "redeploy previous" button. Fastest.
- **Git + redeploy** — `git revert <sha>` on main → CI redeploys. Cleaner history.
- **Image tag rollback** — `kubectl set image deployment/app app=app:v1.2.3` to pin to last good image.

### Mobile (iOS / Android)
- **Staged rollout halt** — Play Console: pause/halt rollout. App Store Connect: pause phased release.
- **Full rollback is NOT possible** — a store-accepted binary can't be un-shipped. You can only stop further distribution + push a fix.
- **Feature flag kill** — if the bug is behind a flag, flip it off in the remote config. This is your actual "rollback" for mobile.

### Database / Schema
- **Forward-only fix preferred** — most ORMs make `down` migrations brittle. If possible, write a new migration that fixes the broken state rather than running `down`.
- **Never roll back a migration that's already written new data** — you lose the new data. Write a compensating migration instead.

---

## Step 2: Pre-Check — Do We Actually Know What's Good?

Before rolling back, confirm:
- [ ] What commit / build / version are we rolling TO?
- [ ] Was that version actually healthy in production (not just "it was deployed")?
- [ ] Is there a schema migration between now and the target? (If yes, rolling back code without reverting schema = broken.)
- [ ] Do ENV vars match the target version's expectations?
- [ ] Are there any feature flags that need to flip in sync?

If you can't answer all of these in < 2 minutes, the rollback itself is risky. Proceed with a canary.

---

## Step 3: Execute

### Preferred path — platform button
Use the platform's "redeploy previous" or "halt rollout" button. Built for this exact case.

### Fallback — git revert
```bash
git checkout main
git pull
git revert --no-edit <bad-sha>
git push origin main
# CI deploys the revert
```

### Mobile feature flag kill
```
# Whatever your remote config is:
remote_config.set("new_checkout_flow_enabled", false)
remote_config.publish()
```

### Database — compensating migration (NOT `migrate:down`)
```sql
-- If the bad migration added a NOT NULL column with bad default:
ALTER TABLE orders ALTER COLUMN new_field DROP NOT NULL;
-- Or revert the data side-effect:
UPDATE orders SET status = 'pending' WHERE updated_at >= '2026-04-20 14:00';
```

---

## Step 4: Verify the Rollback Landed

Don't walk away. Watch for 10–15 minutes:
- [ ] Error rate returned to pre-deploy baseline
- [ ] p95 latency returned to pre-deploy baseline
- [ ] New deploy version matches the target (inspect `/version` endpoint, build number, or image tag)
- [ ] Smoke test the critical user flow manually — don't trust dashboards alone

If error rate did NOT recover, the rollback didn't fix it:
- Your hypothesis was wrong — it wasn't the deploy
- Or the rollback didn't actually roll back (check what version is live)
- Escalate to `/incident`

---

## Step 5: Communicate

Update the incident channel + status page:
```
ROLLBACK
  Rolled back to:  <version / commit>
  Reason:          <one line>
  Executed at:     <timestamp>
  Recovery:        <metric> returned to baseline at <timestamp>
  Next step:       Fix coming in /hotfix, ETA <time>
```

---

## Step 6: Plan the Forward Fix

A rollback is a mitigation, not a resolution. The original feature still needs to ship.

- Add a regression test that would have caught this bug
- Identify WHY CI didn't catch it (gap in test coverage, staging missing production-like data, flag not tested)
- File follow-up work before the post-mortem

Run `/incident` within 24 hours.

---

## Hard rules

- **Never roll back without confirming the target version was healthy.**
- **Never `migrate:down` on a migration that touched production data.** Write a compensating migration instead.
- **Never assume rollback succeeded.** Watch the metrics for 10+ minutes.
- **Never skip the communication step.** Silent rollbacks create confusion downstream.
- **Never skip the post-mortem.** A rollback you don't learn from is a rollback you'll do again.

---

## Completion

- **DONE — ROLLED BACK** — previous version live, metrics recovered
- **DONE — HALTED** — mobile rollout halted, fix in progress
- **ESCALATED** — rollback didn't fix it, handed to incident commander


---

## CRITICAL LESSON LEARNED — Build Verification

A rollback isn't done when the deploy finishes — it's done when the metric you were protecting actually recovers. Watch the dashboard.

## CRITICAL LESSON LEARNED — Acceptance Criteria Verification

The AC for a rollback is "production is healthy again." Verify by real metrics AND manual smoke test — never rely on dashboards alone, they can lag.

## CRITICAL LESSON LEARNED — Branch Scope Discipline

Rollback on a release branch; cherry-pick the revert to develop separately. Never force-push over published history.
