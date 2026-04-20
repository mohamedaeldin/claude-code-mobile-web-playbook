# Docs — Write, Update, Generate Documentation

Write or update the documentation a project needs to onboard a new engineer, explain an API, or hand off to another team. Generates or audits: README, API reference, inline code comments, runbooks, architecture docs.

The rule: **docs answer questions humans ask. If a human wouldn't look there for that answer, don't write it there.**

## Triggers
Use when: "write docs", "update the README", "document this", "generate API docs", "write a runbook", "architecture doc", "ADR", "document this feature"
Proactively suggest when: the user finishes a non-trivial feature that requires explanation, adds a new API surface, or says "hand this off."

---

## Step 0: What Kind of Doc?

Different audiences need different docs. Pick one per task:

| Doc type | Audience | Answers |
|---|---|---|
| **README** | First-time visitor | What is it? How do I run it? Where do I go next? |
| **API reference** | Integrator | What endpoints / functions exist? What are their inputs, outputs, errors? |
| **Architecture doc** | Future maintainer | Why is it built this way? What are the boundaries? |
| **ADR** (Architecture Decision Record) | Future self | Why did we choose X over Y at this specific moment? |
| **Runbook** | On-call engineer | Something is broken — what do I do? |
| **Changelog** | User of the library | What changed in this version? |
| **Inline comments** | Reader of the code | Why does this look wrong but isn't? |
| **Tutorial** | New user | Take me from zero to working example. |

If the user asked "document this," pick the type that fits and confirm.

---

## Step 1: Read Before You Write

Don't write docs from imagination — read the code, the tests, the commit history first.

```bash
# Scope
git log --oneline --since="1 month ago" | head -20
find . -name "README*" -not -path "*/node_modules/*" 2>/dev/null
find . -type d -name "docs" 2>/dev/null
```

For API docs: read the actual endpoint definitions, test fixtures, error responses.
For architecture: read the module boundaries, import graph, dependency injection wiring.
For runbooks: read the actual alert definitions and past incident post-mortems.

---

## Step 2: Draft by Section

### README template (every project needs this)
```markdown
# {Project Name}

{One-sentence description — what it does, for whom}

## Quick Start
{Copy-paste commands that work on a clean machine}
$ git clone ...
$ ./bootstrap.sh
$ ...

## Prerequisites
- {Required tools + minimum versions}

## Architecture
{One paragraph + a diagram link}

## Development
{How to run tests, common commands}

## Deploy
{Link to runbook or explain here}

## Project conventions
{Anything non-obvious a new dev should know in week 1}

## Troubleshooting
{Top 3 things that go wrong + fixes}
```

### API reference
Per endpoint:
```markdown
### `POST /api/v1/widgets`
Create a new widget.

**Auth:** Bearer token, scope `widgets:write`

**Request**
\`\`\`json
{ "name": "string", "color": "red|blue|green" }
\`\`\`

**Response 201**
\`\`\`json
{ "id": "wgt_...", "name": "...", "createdAt": "2026-04-20T..." }
\`\`\`

**Errors**
- `400 invalid_name` — name empty or > 100 chars
- `409 conflict` — name taken
- `429 rate_limit` — retry after `Retry-After` seconds
```

Always auto-generate where possible (OpenAPI, TypeDoc, javadoc, Dokka, GoDoc, rustdoc). Hand-write only the parts the generator can't produce.

### ADR (one decision, one page)
```markdown
# ADR-042: Use PostgreSQL for time-series data

Date: 2026-04-20
Status: Accepted

## Context
{What forces are pushing the decision}

## Decision
{What we decided}

## Alternatives considered
- InfluxDB — {why rejected}
- TimescaleDB — {why rejected}
- Cassandra — {why rejected}

## Consequences
**Positive:** {what gets better}
**Negative:** {what we're paying in exchange}
**Mitigations:** {what we'll do to reduce the negatives}
```

### Runbook
```markdown
# Runbook: High API Latency Alert

**Alert:** `p99_latency_seconds > 2 for 5m`
**Severity:** SEV 2 by default
**On-call:** Backend

## First 2 minutes
1. Check dashboards: {link}
2. Recent deploy? (`/recent-deploys`) → if yes, prep `/rollback`

## Diagnose
- Is one endpoint slow or all? Check p99 by route.
- Is the DB slow? Check slow-query log.
- External dependency? Check upstream status.

## Mitigate
- Kill offending feature flag: {command}
- Scale up: {command}
- Rollback: see `/rollback`

## Resolve
- Fix root cause via `/hotfix`
- Post-mortem within 48h via `/incident`
```

---

## Step 3: Style Rules

- **Sentences over bullets** for narrative; bullets for lists of parallel items.
- **Active voice.** "The service retries on 503" not "The service will be retrying on 503s."
- **One idea per paragraph.** If a paragraph has two ideas, split it.
- **Code examples that actually run.** Copy-paste should work without edits.
- **No marketing copy.** "Robust, scalable, enterprise-grade" = noise. Delete.
- **No TODOs without owners + dates.** `TODO(@alice, 2026-05-01): handle empty list` or don't write it.
- **No secrets in examples.** Use `YOUR_API_KEY`, not a real token you'll "remember to remove."
- **Dates in ISO format.** `2026-04-20`, not `April 20, 2026` or `04/20/2026`.

---

## Step 4: Verify the Docs Work

For a README or tutorial, docs are "done" when someone unfamiliar can follow them end-to-end. Proxy test:
- Copy the Quick Start commands into a clean shell
- Do they run without error?
- Does each link actually resolve?
- Does each code example compile?

Tools:
```bash
# Broken link check
npx linkinator README.md

# Code block extraction + run
# (language-specific — pytest-md, doctest-typescript, etc.)
```

For API docs, run the example requests against the real endpoint and confirm the responses match.

---

## Step 5: Inline Comments — When and Why

Default: **write no comments.** Code with good names explains itself.

Exception — write a comment when:
- **Why not what.** "We retry 3 times because the upstream has a known 1-in-1000 flake" — a reader can't derive this from the code.
- **Invariant.** "Caller must hold `lock` before calling this" — violating it crashes at runtime.
- **Workaround.** "iOS 17.2 rounds differently, so we pre-round to 2 decimals" — with a dated issue link if possible.
- **Surprise.** "This `if` looks redundant but catches a race we hit in March 2026 incident."

Never:
- Restate the code (`// increment counter` above `counter++`)
- Leave `// TODO` without owner + date
- Describe the PR this commit came from (goes in commit message, not code)
- Reference the ticket ID (goes in commit/PR, rots in code)

---

## Step 6: Where Docs Live

- **README** — repo root, first thing GitHub shows
- **CHANGELOG** — repo root, `CHANGELOG.md`
- **API reference** — auto-generated site (docusaurus, starlight, mkdocs) OR repo `docs/api/`
- **ADRs** — `docs/adr/NNNN-title.md`, numbered sequentially, immutable once Accepted
- **Runbooks** — `docs/runbooks/alert-name.md`, discoverable from alert messages
- **Architecture** — `docs/architecture/` with one page per major system

Never scatter the same info across 3 places. Pick one canonical location and link from elsewhere.

---

## Step 7: Keep Docs Alive

Docs rot fastest when they sit far from the code. Tactics that help:
- CI job that fails the build if public API changes but docs don't
- `readme-ci` action to fail if README commands don't run
- Quarterly audit via this skill
- Link every major doc from the README so nothing becomes orphaned

---

## Hard rules

- **Never write docs that contradict the code.** Delete stale docs; don't leave them.
- **Never leave untested code examples.** A broken example is worse than no example.
- **Never write "will be updated" or "coming soon" — either it's there or it isn't.**
- **Never document from memory. Read the code first.**
- **Never commit docs that fail link check.**

---

## Completion

- **DONE** — docs written, examples verified, links check
- **DRAFT** — structure + content, awaiting review
- **BLOCKED** — code unclear / undecided; capture as a question list, don't guess


---

## CRITICAL LESSON LEARNED — Build Verification

Docs are built artifacts. Run the link check, run the example code, build the docs site. Don't trust "it looks right."

## CRITICAL LESSON LEARNED — Acceptance Criteria Verification

"Docs updated" is proven by a fresh reader walking the Quick Start successfully. If you can't test that, label the docs as UNVERIFIED.

## CRITICAL LESSON LEARNED — Branch Scope Discipline

Docs follow the code they describe. If the code is on a feature branch, the docs belong on that same branch — not dropped into main separately.
