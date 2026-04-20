# Claude Code — Mobile & Web Playbook

A curated set of Claude Code slash commands (skills) for building and shipping mobile (iOS, Android) and web apps following proven best practices.

## Install

Copy the `.md` files into your Claude Code commands directory:

```bash
git clone https://github.com/mohamedaeldin/claude-code-mobile-web-playbook.git
cp claude-code-mobile-web-playbook/*.md ~/.claude/commands/
```

Each command becomes available in Claude Code as `/<filename>` (e.g. `ios-dev.md` → `/ios-dev`).

## What's inside

**Develop**
- `/ios-dev` — iOS feature implementation (Swift, SwiftUI, UIKit)
- `/android-dev` — Android feature implementation (Kotlin, Jetpack Compose)
- `/web-dev` — Web feature implementation (React, Vue, Next.js, Svelte)
- `/backend-dev` — Backend feature implementation

**Review & Audit**
- `/code-review` — Pre-landing diff audit
- `/ios-audit`, `/android-audit`, `/web-audit`, `/backend-audit` — Deep platform reviews
- `/design-review` — UI/UX audit
- `/design-system` — Design tokens and components
- `/api-review` — API contract review
- `/security-audit` — Threat model and vulnerability scan
- `/benchmark` — Performance audit
- `/dependencies` — Dependency audit and CVE scan
- `/plan-review` — Architecture review
- `/scope-review` — Scope and vision check
- `/full-review` — Run every review back-to-back

**Ship**
- `/ship` — Automated PR workflow
- `/deploy` — Backend and web deployment
- `/release-ios` — TestFlight and App Store submission
- `/release-android` — Play Store submission
- `/mobile-quality` — Mobile store-readiness check
- `/rollback` — Revert a broken release
- `/hotfix` — Emergency production fix

**Operate**
- `/monitor` — Observability, logs, metrics, traces, alerts
- `/incident` — Coordinate outages and post-mortems
- `/investigate` — Root-cause debugging
- `/qa` — Multi-platform testing and bug fixing
- `/auto-fix` — Review and apply fixes
- `/migrate` — Database and schema migration

**Plan & Think**
- `/brainstorm` — Product thinking before code
- `/research` — Deep technical investigation with sources
- `/challenge` — Devil's advocate / critical thinker
- `/docs` — Write and generate documentation
- `/retro` — Retrospective

**Workflow**
- `/deliver` — End-to-end task lifecycle orchestrator
- `/team-agents` — Multi-agent engineering system
- `/safe-mode-on` / `/safe-mode-off` — Scope locking for sensitive operations

## License

MIT — use freely, adapt to your team.
