# Research — Deep Technical Investigation with Sourced Evidence

Take a topic, technology choice, or open question and **gather evidence systematically**: docs, existing code, benchmarks, similar projects, real-world experience. Produce a sourced recommendation with an honest confidence level.

The rule: **every claim in the output must cite its source. No "I think" or "it's probably" without evidence attached. When evidence is thin, say so — low confidence is a valid answer.**

This is proactive research BEFORE a decision. Use `/investigate` for reactive root-cause debugging (post-bug).

## Triggers
Use when: "research", "investigate this technology", "compare X vs Y", "what's the best way to", "deep dive on", "evaluate", "spike", "RFC prep", "survey the options"
Proactively suggest when: the user asks "should we use X?" or "how do other people solve this?" or is about to pick a library / pattern / technology without a sourced comparison.

---

## Step 0: Sharpen the Question

A vague question produces a vague answer. Rewrite the input into a falsifiable question:

| Vague | Sharpened |
|---|---|
| "Research auth" | "What auth library fits a single-tenant internal app on Node 20, given we need SSO + local dev with mock users?" |
| "Look into Liquid Glass" | "What's the minimum iOS version for Liquid Glass, what's the API surface we'd need to adopt, and what's the fallback story for iOS < 26?" |
| "Should we use Kotlin Multiplatform?" | "What's the realistic cost of adopting KMP for the data layer only, given our iOS team has 0 Kotlin experience?" |

State the question explicitly and confirm with the user before continuing. If the user objects, refine. Don't research the wrong question.

---

## Step 1: Map the Unknowns

Before searching, list what you don't know. Bucket each:
- **Factual** — what does the doc say? (answerable by reading)
- **Empirical** — how does it behave? (answerable by running)
- **Experiential** — what goes wrong in practice? (answerable by finding reports)
- **Comparative** — how does it stack up? (answerable by benchmarking or surveying)

Each unknown becomes a research subtask. Skip no bucket — answering only the factual questions and stopping is how teams adopt a technology that "looks great in the docs" and breaks at scale.

---

## Step 2: Gather Evidence — Multiple Sources

Never rely on one source. Required minimum:

### A. Official documentation
- Read the "Getting Started" AND the "Limitations" / "FAQ" sections
- Check changelog for breaking changes in last 2 years
- Note the version you're reading (docs drift)

### B. Source code / repo
- Stars, recent commit activity, open-vs-closed issues, maintainer count
- Grep the issues for your use case — someone else probably hit it
- Check the CHANGELOG for deprecations

### C. Existing usage in YOUR codebase
```bash
grep -r "{library-name}" --include="*.{ext}" -l 2>/dev/null
```
- Is there an existing pattern you can match?
- Has someone tried this already and reverted? Check `git log --all`.

### D. Real-world use
- Production stories (blog posts, talks, post-mortems from other companies)
- "{library} at scale" / "{library} in production" searches
- Discord / Slack / Reddit for the community's lived experience

### E. Benchmarks (if performance matters)
- Run a minimal spike in a throwaway branch
- Don't trust published numbers — they were run on someone else's hardware with someone else's data shape
- Measure the thing your use case actually does

### F. Alternatives survey
- Always list at least 3 alternatives, even if you think you know the answer
- For each: "fits when X, doesn't fit when Y"

---

## Step 3: Cite EVERYTHING

Every claim in the final output traces to one of:
- A doc URL + section name
- A code file:line (yours or theirs)
- A benchmark run + timestamp
- A named issue / PR / thread

No source → no claim. If the best you have is "I remember reading somewhere," mark it as UNVERIFIED and list what would verify it.

### Citation format
```
[1] https://library.com/docs/api-v2#rate-limits (read 2026-04-20, v2.3.1)
[2] github.com/library/issues/4521
[3] our-repo/src/network/interceptor.ts:42-67
[4] local benchmark — 2026-04-20, MacBook M3, node 20.11
```

---

## Step 4: Compare Honestly

When comparing alternatives, use the same dimensions for each. Typical dimensions:

| Dimension | What to measure |
|---|---|
| Fit | How well does it match YOUR actual requirements? |
| Maintenance | Last release, issue close rate, maintainer count |
| Ecosystem | TypeScript types? framework integrations? docs quality? |
| Performance | Real numbers on your data shape — not the landing page |
| Size | Bundle / binary impact |
| License | Compatible with your distribution? |
| Migration cost | If we adopt and later want to swap, how painful? |
| Security | Recent CVEs, track record of response time |
| Community | Active contributors, Stack Overflow tag count |
| Lock-in | Proprietary APIs, vendor-specific, reversal cost |

Produce a table. Fill every cell with an answer AND a citation. Empty cells mean unfinished research — don't skip.

---

## Step 5: Confidence Level

Every recommendation carries an explicit confidence band:

| Confidence | Meaning | When appropriate |
|---|---|---|
| **HIGH** | Multiple independent sources agree + I tested it on your data | Enough evidence to act without more research |
| **MEDIUM** | Docs + one real-world source agree, but I haven't tested | Safe for low-stakes decisions, risky for production-critical ones |
| **LOW** | Inferred from docs only, or sources conflict | Do more research OR run a time-boxed spike before committing |
| **SPECULATIVE** | Pattern-matched from experience, no direct evidence | Treat as a hypothesis, not a recommendation |

Downgrade your confidence when:
- Sources are < 1 year old but the project's version changed recently
- You only found marketing content (vendor blogs, sponsored comparisons)
- Your real use case differs materially from the examples you found

---

## Step 6: Produce the Research Report

```
RESEARCH REPORT
════════════════════════════════════════════
Question:    {sharpened question from Step 0}
Scope:       {what's in scope, what's out}
Duration:    {time spent}

EXECUTIVE SUMMARY (3-5 lines)
  Recommendation: {pick one}
  Confidence:     {HIGH / MEDIUM / LOW / SPECULATIVE}
  Reversal cost:  {how hard to change course later}
  Next step:      {ship with this / do a spike / get more info}

COMPARISON TABLE
  ┌─────────────┬────────┬────────┬────────┐
  │             │ OptionA│ OptionB│ OptionC│
  ├─────────────┼────────┼────────┼────────┤
  │ Fit         │        │        │        │
  │ Maintenance │        │        │        │
  │ Perf        │        │        │        │
  │ Size        │        │        │        │
  │ License     │        │        │        │
  │ Migration$  │        │        │        │
  │ Security    │        │        │        │
  │ Lock-in     │        │        │        │
  └─────────────┴────────┴────────┴────────┘
  (every cell cites a source)

EVIDENCE
  [1] source — finding
  [2] source — finding
  ...

KEY FINDINGS (3-7 bullets)
  - Finding + citation + implication

WHAT I DIDN'T VERIFY (be honest)
  - {gap} — would need {specific thing} to close
  - ...

RISKS OF THE RECOMMENDED OPTION
  - {concrete risk} + mitigation

ESCAPE HATCHES
  - If this option fails, migration to {alternative} costs {estimate}

OPEN QUESTIONS (for the decision-maker)
  - ...
```

---

## Hard rules

- **Never recommend without citations.** If every claim can't trace to a source, you're guessing.
- **Never hide uncertainty.** Low confidence with proof > high confidence with vibes.
- **Never skip the alternatives.** Even if the answer is obvious, listing 3 options keeps you honest.
- **Never trust vendor marketing.** Every landing page says "fastest, most scalable, enterprise-grade." Find independent sources.
- **Never copy benchmarks.** Measure on your hardware, your data, your code path.
- **Never exceed the time box.** State the box up front ("I'll spend 2 hours on this"). When it's done, submit what you have with a clear "incomplete" marker — don't silently grind for 6 more hours.

---

## When NOT to use /research

- You already know the answer and can name 3 sources (just decide)
- The question is "why did X break" (use `/investigate` — reactive)
- You want subjective product direction (use `/brainstorm`)
- You need architecture validation of a specific plan (use `/plan-review`)
- The cost of a wrong pick is small AND reversible (pick something, move on)

`/research` earns its weight on one-way-door decisions, novel technology, or anything that will anchor an architecture for years.

---

## Completion

- **DONE — HIGH CONFIDENCE** — recommendation produced, sources strong, ready to act
- **DONE — NEEDS SPIKE** — paper research complete, time-boxed code spike recommended before full commit
- **DONE — INCONCLUSIVE** — evidence conflicts or is missing; research couldn't answer the question within the time box


---

## CRITICAL LESSON LEARNED — Build Verification

Research that looks good on paper fails in production when the spike is skipped. For any HIGH-confidence recommendation on a load-bearing choice, run a time-boxed spike against real data before committing.

## CRITICAL LESSON LEARNED — Acceptance Criteria Verification

The "AC" for research is: "another engineer with no context can read the report and make the same decision from the evidence." If they can't reproduce your reasoning from your sources, the report isn't done.

## CRITICAL LESSON LEARNED — Branch Scope Discipline

Research the specific question asked — don't drift into "while I'm at it, let's also evaluate Z." Drift dilutes the output and burns the time box. Scope creep in research is as costly as scope creep in code.
