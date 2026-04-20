# Challenge — Critical Thinker / Devil's Advocate

Take a plan, decision, architecture, or proposed approach and **deliberately attack it** — find the hidden assumptions, failure modes, and costs that the author missed. Produce structured counter-arguments and alternatives.

The rule: **your job here is to steelman the objection, not to feel clever. Every pushback must name a concrete harm or a specific alternative — "I don't like it" is not a challenge.**

This is structured dissent before commitment. Use it BEFORE `/plan-review` approves something, BEFORE a major architectural choice, BEFORE a hotfix changes production behavior.

## Triggers
Use when: "challenge this", "devil's advocate", "find the flaws", "what am I missing", "pre-mortem", "stress-test this plan", "what could go wrong", "red team this", "steelman the opposing view"
Proactively suggest when: the user says "I'm pretty sure this is the right approach" or "let's just ship it" on a non-trivial change.

---

## Step 0: Lock the Target

Ask what exactly is being challenged:
- An implementation plan?
- An architectural choice (library X vs Y, pattern A vs B)?
- A decision to ship / not ship?
- A specific diff?
- A post-mortem root-cause claim?

The target MUST be a specific, written artifact. If the user says "challenge my idea" and can't point at something concrete, stop — make them write it down first. You can't critique vapor.

---

## Step 1: Apply Five Attack Frames

Run each frame in order. For each, record concrete findings (not generic worries).

### Frame 1 — Pre-Mortem
Imagine it's 6 months from now and this plan **failed catastrophically**. Write the post-mortem:
- What did we miss that is obvious in hindsight?
- What assumption turned out to be wrong?
- Which stakeholder got blindsided?
- Which edge case ate us?

Name at least 3 failure modes with specific triggers ("when user count exceeds 10k, the backfill query takes 8 hours and blocks writes").

### Frame 2 — Hidden Assumptions
List every assumption the plan silently makes. Format each as:
> "This plan assumes that X. If X is false, then Y breaks."

Common assumption classes to probe:
- **Data shape** — "assumes users have at most 1 org" / "assumes all timestamps are UTC"
- **Scale** — "assumes < 100 rows/sec" / "assumes < 1 GB per file"
- **Trust** — "assumes the upstream API is honest about its rate limits"
- **Availability** — "assumes the DB is up when we write"
- **Ordering** — "assumes deploys happen before migrations"
- **Locale** — "assumes LTR layout / English copy"
- **Time** — "assumes user clock is correct"
- **Identity** — "assumes personId is stable across sessions"

Flag any assumption that isn't explicitly justified by data or a design note.

### Frame 3 — Chesterton's Fence
For every EXISTING behavior the plan removes or changes, ask: **why does it currently work this way?**

> "You're removing the feature flag. Why was it there? Who added it? What incident led to it? The git blame says 2024 and references an incident that might still be relevant."

If the author can't explain why the old thing existed, they're not ready to change it. Stop and make them dig.

### Frame 4 — Conway's / Hyrum's Law
- **Conway:** the architecture will mirror team communication structures. "You're proposing one service owned by two teams — which team will answer pager at 3 AM?"
- **Hyrum:** every observable behavior will be depended on by someone. "You're changing the error message from 'Not Found' to 'not found'. Who greps for the old wording in their scripts / tests / support docs?"

Find at least one Conway concern and one Hyrum concern per plan.

### Frame 5 — Reversibility + Blast Radius
Bucket every decision in the plan:
- **Two-way door** — easy to reverse. Fine to move fast.
- **One-way door** — hard to reverse. Deserves extra scrutiny.

For every one-way door:
- Confirm it's intentional, not accidental
- Document the rollback plan (even if theoretical)
- Ask: is there a two-way-door equivalent that gets 80% of the value?

Examples of one-way doors that authors often don't see:
- Renaming a public endpoint / event / column
- Removing a feature that external tools scrape
- Changing the meaning of an ID or status enum value
- Schema migrations that destroy the `down` path
- Public API changes in a library
- User-facing copy changes that trigger re-translation

---

## Step 2: Steelman the Opposing View

For EACH significant choice in the plan, write the strongest possible case FOR the alternative:

```
DECISION:  Use PostgreSQL JSONB for the event payload column.
STEELMAN:  Use a separate `events` table with normalized columns.
  - Better query planner stats on real columns than JSONB paths
  - Easier to add indexes exactly where needed later
  - Schema becomes self-documenting in pgAdmin
  - Non-DB tooling (Redash, Metabase) queries it naturally
  - Easier to backfill / audit than JSONB introspection
  REBUTTAL: JSONB wins because {specific reason}, the steelman's
            costs are {specific costs}.
```

If you can't produce a rebuttal, the plan MIGHT BE WRONG. Surface that.

---

## Step 3: Stakeholder Perspectives

Walk the plan from other points of view. You're looking for concerns the author didn't consider because they're outside their role.

| Stakeholder | Concern they'd raise |
|---|---|
| **Security** | "What new attack surface does this add?" |
| **On-call** | "What new alerts do I need? What new failure mode can page me?" |
| **Support** | "What new user-facing errors will I see? Do I have a runbook?" |
| **Legal / Compliance** | "What new data do we collect / retain / share?" |
| **Finance** | "What's the cost at 10× scale? Is it linear?" |
| **Next-on-call** | "Can I operate this system without the author?" |
| **New-hire (6 months from now)** | "Can I understand this from the docs alone?" |
| **User at p99 latency** | "What happens when the slow path hits me?" |

---

## Step 4: Produce the Challenge Report

```
CHALLENGE REPORT
════════════════════════════════════════
Target:     {plan / decision / diff}
Summary:    {1-line author's intent}
Stance:     CHALLENGE (no matter how good the plan, you're the adversary)

CONCRETE OBJECTIONS
  [CRITICAL]   Objection — specific failure mode
    WHY:       hidden assumption / missing mitigation
    EVIDENCE:  file:line or concrete scenario
    IMPACT:    who suffers when this fires, how often
    ALTERNATIVE: specific different approach

  [HIGH]       ...
  [MEDIUM]     ...

HIDDEN ASSUMPTIONS
  1. "This assumes X" — falsifiable because {evidence}
  2. ...

CHESTERTON FENCES
  - {existing behavior being removed}: can't explain why it's there → BLOCK until investigated

ONE-WAY DOORS
  - {decision}: reversal cost = HIGH → must be intentional, must have rollback plan
  - ...

STAKEHOLDER GAPS
  - Security didn't see: {concern}
  - On-call didn't see: {concern}
  - ...

STEELMANS (alternatives worth considering)
  - {alternative} — at least 3 reasons it might be better

MY VERDICT
  - SHIP AS-IS:  {n} objections are non-blocking
  - REVISE:      {n} objections must be addressed first
  - RETHINK:     fundamental assumption wrong, restart

CONFIDENCE IN MY CRITIQUE: {High / Medium / Low, with reason}
```

---

## Hard rules

- **Never fake objections.** If the plan is good, say so and stop. A challenge that invents flaws loses trust.
- **Every objection names a concrete harm.** "It feels wrong" is not a harm. "When 10k users hit this concurrently, p99 goes to 8s" is.
- **Every objection has an alternative.** Saying "don't do it" without saying "do this instead" is noise.
- **Attack the plan, not the author.** "This misses X" beats "You missed X."
- **Never hide behind hedges.** "Might" and "could" soften everything into noise. Say "will" when you have evidence; say "might" when you don't, but say why.
- **Challenge decisions, not style.** Bikeshedding (tabs vs spaces, which library name is prettier) isn't what this skill is for.

---

## When NOT to use /challenge

- Small reversible change (just ship it; attack is overkill for a 5-line fix)
- Author hasn't committed to a position yet (use `/brainstorm` — explore options, don't attack them)
- You need to actually pick one (use `/plan-review` — architecture scoring)
- You want to know WHY something broke (use `/investigate` — root cause, not critique)

`/challenge` earns its weight on big, hard-to-reverse decisions where the cost of being wrong is high.

---

## Completion

- **DONE — PLAN HOLDS** — critique surfaced nothing blocking; author can proceed with documented risk acceptance
- **DONE — NEEDS REVISION** — objections captured; author should respond point-by-point before proceeding
- **DONE — RETHINK** — fundamental flaw; halt and use `/brainstorm` to redesign


---

## CRITICAL LESSON LEARNED — Build Verification

A "challenge" session doesn't build anything — but its output drives future builds. Verify the author addresses each CRITICAL/HIGH objection with code + tests, not just a reassuring reply.

## CRITICAL LESSON LEARNED — Acceptance Criteria Verification

Every objection raised must have a corresponding acceptance criterion added to the plan ("system handles N concurrent writes without deadlock"). If an objection isn't ACed, it's a ticking clock.

## CRITICAL LESSON LEARNED — Branch Scope Discipline

Challenge the plan on its own merits, not "this branch also changes Y unrelated thing." If scope creep is present, flag it as a separate objection, but focus the critique on the stated intent.
