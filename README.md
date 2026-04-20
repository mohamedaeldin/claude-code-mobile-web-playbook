# Your Skills Toolkit

This folder contains 41 skills. Each one is a recipe Claude follows when you ask for a specific kind of help. Type `/name` or just describe what you need — Claude picks the right one.

---

## I want to... (quick-find)

| I want to... | Use this |
|---|---|
| Think about a new idea | `/brainstorm` |
| Figure out what exactly to build | `/scope-review` |
| Design the architecture | `/plan-review` |
| Research a technology choice | `/research` |
| Find holes in my plan | `/challenge` |
| Set up colors and theme | `/design-system` |
| Check the UI looks right | `/design-review` |
| Check the API contract | `/api-review` |
| Code an iOS feature | `/ios-dev` |
| Code an Android feature | `/android-dev` |
| Code a web feature | `/web-dev` |
| Code a backend feature | `/backend-dev` |
| Review my diff before merge | `/code-review` |
| Deep-audit iOS / Android / web / backend code | `/ios-audit` / `/android-audit` / `/web-audit` / `/backend-audit` |
| Find and fix bugs | `/qa` |
| Review AND fix everything | `/auto-fix` |
| Run every review (report only) | `/full-review` |
| Check if my app is ready for the App Store / Play Store | `/mobile-quality` |
| Run a security check | `/security-audit` |
| Check performance | `/benchmark` |
| Check accessibility (WCAG) | `/a11y-audit` |
| Scan dependencies for CVEs | `/dependencies` |
| Debug a bug (root cause) | `/investigate` |
| Set up logs, metrics, alerts | `/monitor` |
| Fix production RIGHT NOW | `/hotfix` |
| Undo a bad release | `/rollback` |
| Coordinate an active outage | `/incident` |
| Write a DB migration | `/migrate` |
| Be extra careful with edits | `/safe-mode-on` / `/safe-mode-off` |
| Create a PR | `/ship` |
| Deploy to servers | `/deploy` |
| Submit to App Store | `/release-ios` |
| Submit to Play Store | `/release-android` |
| Write docs / README / runbook | `/docs` |
| Retrospective | `/retro` |
| Do ALL of the above in order | `/deliver` |
| Full multi-agent treatment | `/team-agents` |

---

## The normal flow, told like a story

You have an idea. What happens next?

**1. Figure out if it's worth building**
Start with `/brainstorm`. Claude will push back, explore angles, list alternatives. If the idea survives, move on.

**2. Nail down what to build**
`/scope-review` locks the scope: who's the user, what's in, what's out, how do we measure success.

**3. Decide how to build it**
`/plan-review` scores the architecture. Data flow, error handling, performance, security, testability. If anything scores below a 3/5, fix the plan before coding.

**4. (Optional) Research unknowns**
If the plan uses a library or pattern that's new to you, run `/research`. It gathers evidence with citations and gives an honest confidence level.

**5. (Optional) Attack your own plan**
Before committing, run `/challenge`. Claude plays devil's advocate — pre-mortem, hidden assumptions, "what could go wrong." Better to find the hole now than in production.

**6. Design the UI (if there is one)**
`/design-system` sets up the tokens. `/design-review` audits the screens.

**7. Build it**
Pick the platform: `/ios-dev`, `/android-dev`, `/web-dev`, or `/backend-dev`. Each one writes the code, runs tests, and hands off.

**8. Review**
`/code-review` catches structural issues. Then run the deep platform audit: `/ios-audit`, `/android-audit`, `/web-audit`, or `/backend-audit`.

**9. Find and fix bugs**
`/qa` hunts bugs and writes regression tests. Or use `/auto-fix` to sweep everything (code + bugs + security + a11y + deps) in one shot.

**10. Extra gates before release**
For mobile: `/mobile-quality` (lifecycle, devices, permissions, store readiness). For any app: `/security-audit`, `/benchmark`, `/a11y-audit`, `/dependencies`.

**11. Ship**
Turn on safety mode (`/safe-mode-on`), create the PR (`/ship`), deploy (`/deploy` / `/release-ios` / `/release-android`), turn off safety (`/safe-mode-off`).

**12. Learn**
`/retro` — what worked, what didn't, what to change next time.

**The shortcut:** type `/deliver`. It asks three questions (task type, platform, scope) and runs the right subset of the above for you.

---

## Skills, grouped by when you use them

### 💭 Thinking
These come **before** any code.

- **`/brainstorm`** — "I have an idea, help me think through it."
- **`/scope-review`** — "What exactly should we build?"
- **`/plan-review`** — "Is the architecture sound?"
- **`/research`** — "Should we use X or Y? Help me compare."
- **`/challenge`** — "Find the holes in my plan before I start coding."

### 🎨 Designing
These come when UI or API shape is in play.

- **`/design-system`** — Colors, typography, spacing tokens. Do this once per project.
- **`/design-review`** — Look at the Figma + implementation, flag mismatches.
- **`/api-review`** — Check API contract compatibility (OpenAPI, GraphQL, gRPC).

### 🛠️ Building
These write the actual code.

- **`/ios-dev`** — Swift, SwiftUI, UIKit.
- **`/android-dev`** — Kotlin, Jetpack Compose.
- **`/web-dev`** — React, Vue, Next.js, SvelteKit.
- **`/backend-dev`** — Node, Python, Go, Ruby, Rust.

Each one follows the same approach: read the plan, read the codebase, plan the file tree, implement bottom-up, compile often, write tests, hand off.

### 🔍 Reviewing
These check the code without running your app.

- **`/code-review`** — Every PR, always. Catches the common stuff.
- **`/ios-audit`** / **`/android-audit`** / **`/web-audit`** / **`/backend-audit`** — Deeper, platform-specific. Run these for non-trivial changes.
- **`/qa`** — Finds bugs, fixes them, writes regression tests. Pick a mode: Quick / Standard / Exhaustive / Report-Only.
- **`/auto-fix`** — Runs every review skill AND applies fixes. Use when you want one command to sweep everything.
- **`/full-review`** — Runs every review skill, report only (no fixes). Use when you want to see everything before deciding how to fix.

### ✅ Verifying
These are the gates before release.

- **`/mobile-quality`** — The big one for mobile apps. Lifecycle, network, device matrix, permissions, perf, reliability, SDK audit, store readiness.
- **`/security-audit`** — OWASP, secrets, auth, input validation. Any time auth or data changes.
- **`/benchmark`** — Performance: startup time, memory, bundle size, jank.
- **`/a11y-audit`** — WCAG 2.1 AA compliance. Screen reader + keyboard + contrast + zoom.
- **`/dependencies`** — Scan for CVEs, outdated libraries, bad licenses, abandoned projects.

### 🔧 Operating
These are for when something's live (or broken).

- **`/investigate`** — "Something's wrong, help me find the cause."
- **`/monitor`** — Set up SLOs, dashboards, alerts, runbooks.
- **`/hotfix`** — Production is broken NOW. Fast-lane with strict safety.
- **`/rollback`** — Undo a bad release.
- **`/incident`** — Running the active outage + writing the post-mortem.
- **`/migrate`** — Database / schema changes done safely (expand → backfill → contract).

### 🔒 Safety
These fence off risky operations.

- **`/safe-mode-on`** — "Claude, only edit these files. Don't touch anything else."
- **`/safe-mode-off`** — "OK, safe to edit normally again."

### 🚢 Shipping
These push code out the door.

- **`/ship`** — Make the PR.
- **`/deploy`** — Push to servers (backend / web).
- **`/release-ios`** — TestFlight or App Store.
- **`/release-android`** — Play Store.

### 📚 Documenting + Learning
These keep the team healthy long-term.

- **`/docs`** — Write README, API ref, ADRs, runbooks, useful comments.
- **`/retro`** — What did we ship this week / sprint / release? What worked? What didn't?

### 🧭 Orchestrators
These run multiple skills for you.

- **`/deliver`** — The end-to-end driver. Ask 3 questions, run the right flow.
- **`/team-agents`** — The full "engineering org" treatment. Roles, gates, the works.

---

## Common scenarios, copy-paste

### "I need to add a small feature to our iOS app"
```
/plan-review
/ios-dev
/code-review
/ios-audit
/qa
/ship
/release-ios
```

### "I'm adding a new API endpoint"
```
/plan-review
/api-review
/backend-dev
/code-review
/backend-audit
/security-audit
/qa
/ship
/deploy
```

### "Production is broken right now"
```
/hotfix
```
(It handles the whole flow — stabilize, fix, test, ship, monitor. If rollback is faster, it hands off to `/rollback`.)

### "I want to redesign all the UI"
```
/scope-review
/plan-review
/design-system
/design-review
/ios-dev (or /android-dev, /web-dev)
/code-review
/platform-audit
/qa
/a11y-audit
/ship
```

### "We need to check if we're ready to submit to App Store"
```
/mobile-quality
```
(That's it. It runs the whole pre-submission gate.)

### "I don't know, just do the right thing"
```
/deliver
```
(It asks you 3 questions and picks the flow.)

---

## FAQ

**Do I have to type the slash?**
No. Both work:
- `/ios-dev`
- "implement this iOS feature"

Claude picks the skill either way.

**What if I pick the wrong one?**
Just say so. Claude switches mid-conversation. Or ask "which skill should I use?"

**Can I skip phases?**
Yes. For small tasks, just run `/code-review` → `/qa` → `/ship`. Skip planning.

**What's the difference between `/qa` and `/auto-fix`?**
`/qa` focuses on bugs (finds them, fixes them, writes tests).
`/auto-fix` does everything `/qa` does PLUS security, a11y, dependencies, code quality — one sweep.
If you just want bug-hunting, use `/qa`. If you want a full sweep before merging, use `/auto-fix`.

**What's the difference between `/full-review` and `/auto-fix`?**
`/full-review` reports only (no edits). `/auto-fix` also applies fixes.

**What's the difference between `/investigate` and `/challenge`?**
`/investigate` asks "why is this broken?" (reactive).
`/challenge` asks "what could break?" (proactive).

**What's the difference between `/retro` and `/incident`?**
`/retro` is a regular reflection on recent work.
`/incident` is specifically for an active outage → its post-mortem is immediate and blameless.

**What about naming?**
- Skills ending in `-dev` write code.
- Skills ending in `-audit` review code.
- Skills ending in `-review` review plans/designs/contracts.

---

## Tips

1. **Start with the task, not the skill.** Describe what you need — Claude picks the right tool.
2. **Small tasks don't need `/deliver`.** A 5-minute fix just needs `/code-review` + `/ship`.
3. **`/plan-review` saves real time.** 30 minutes of planning can save 3 days of rework.
4. **Always run `/code-review` before merging.** Non-negotiable.
5. **`/auto-fix` is powerful but opinionated.** If you want granular control, run reviews individually.
6. **Gates exist to help you.** If `/mobile-quality` finds a P0, don't skip it. Fix it.
7. **Use `/safe-mode-on` before hotfixes and production deploys.** It prevents accidental edits.
8. **Write a `/retro` for every big thing shipped.** Even if no one reads it, the act of writing it makes you smarter next time.

---

## If a skill feels missing

You might want these later:
- `/feature-flag` — rollout strategy
- `/onboard` — codebase tour for new engineers
- `/changelog` — auto-generate changelog
- `/backport` — port a fix from main to a release branch
- `/i18n` — localization audit
- `/cleanup` — dead code removal

Don't build them until you hit the need twice. Overbuilding the toolkit wastes the same time it saves.

---

## The mental model, one line each

- **THINK** → decide what and how
- **DESIGN** → set up the look
- **BUILD** → write the code
- **REVIEW** → check the code
- **VERIFY** → check the release
- **SHIP** → push it out
- **OPERATE** → keep it alive (or revive it)
- **DOC** → explain it
- **ORCHESTRATE** → run all of the above in the right order

That's the whole system. Forty-one skills, nine phases, one mental model.
