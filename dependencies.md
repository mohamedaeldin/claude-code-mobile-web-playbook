# Dependencies — Audit, Upgrade, Secure

Audit every third-party dependency for version health, CVEs, abandonment, license compatibility, and size impact. Produce a prioritized upgrade / removal plan.

The rule: **every unpinned dep, every outdated major, every open CVE is a ticking clock. Know what you owe.**

## Triggers
Use when: "dependency audit", "update dependencies", "upgrade libraries", "CVE check", "security patches", "outdated packages", "what needs upgrading", "dependabot", "renovate"
Proactively suggest when: the user mentions a CVE, a deprecated library, or asks to update.

---

## Step 0: Detect Package Manager

```bash
# Node
ls package.json package-lock.json yarn.lock pnpm-lock.yaml bun.lockb 2>/dev/null

# iOS
ls Podfile Podfile.lock Package.swift Package.resolved Cartfile 2>/dev/null

# Android
ls build.gradle build.gradle.kts app/build.gradle app/build.gradle.kts gradle.lockfile 2>/dev/null
ls gradle/libs.versions.toml 2>/dev/null  # version catalog

# Python
ls requirements.txt pyproject.toml Pipfile poetry.lock uv.lock 2>/dev/null

# Go
ls go.mod go.sum 2>/dev/null

# Ruby
ls Gemfile Gemfile.lock 2>/dev/null

# Rust
ls Cargo.toml Cargo.lock 2>/dev/null
```

---

## Step 1: Inventory

List every direct dependency with:
- Name
- Declared version (exact or range)
- Installed (locked) version
- Latest available
- Age of declared version

```bash
# Node
npm outdated --json
# Or: npx npm-check-updates --format group

# iOS (SPM)
swift package show-dependencies --format json
# CocoaPods
pod outdated

# Android (Gradle)
./gradlew dependencyUpdates -Drevision=release

# Python
pip list --outdated --format=json
# Or: uv tree

# Go
go list -m -u all

# Ruby
bundle outdated
```

Save the output — it's the basis of the report.

---

## Step 2: CVE / Security Check

Security is separate from "out of date." A dep can be current AND vulnerable.

```bash
# Node
npm audit --audit-level=moderate
# Or: npx audit-ci --moderate

# iOS (SPM + CocoaPods)
# No native CVE tool — use GitHub Dependabot alerts on the repo

# Android
./gradlew dependencyCheckAnalyze   # OWASP Dependency Check plugin
# Or: ./gradlew :app:dependencyInsight --dependency <name>

# Python
pip install pip-audit && pip-audit
# Or: safety check

# Go
govulncheck ./...

# Ruby
bundle audit check

# Multi-language SBOM
syft <project> -o json > sbom.json
grype sbom:sbom.json   # CVE check against SBOM
```

Record every CVE with: CVSS score, affected version, fixed version, exploitability (is the vulnerable code path reachable in your app?).

---

## Step 3: License Check

Unapproved licenses can block distribution.

```bash
# Node
npx license-checker --json

# iOS
# CocoaPods: pod-license-plist
# SPM: manually inspect each dep's LICENSE

# Android
./gradlew :app:licenseDebugReport

# Python
pip-licenses --format=json

# Go
go-licenses report ./...
```

Flag: GPL, AGPL, proprietary with uncertain terms, "unlicense," or missing license.

---

## Step 4: Maintenance Health

An up-to-date dep from an abandoned project is a time bomb. For each direct dep, check:
- Last release date (> 18 months = yellow, > 3 years = red)
- Open vs closed issues ratio
- Maintainer count (bus factor of 1 = risk)
- Recent commits to main

GitHub Insights tab or `npm info <pkg> time` gives most of this.

---

## Step 5: Size Impact (for client-side bundles)

Client bundle size affects every user.

```bash
# Web
npx vite-bundle-visualizer
# Or: npm run build && npx source-map-explorer 'dist/**/*.js'

# iOS
xcrun size -m Employee.app/Employee

# Android
./gradlew :app:bundleRelease
# Then inspect app/build/outputs/bundle/release/*.aab with bundletool
```

Flag any single dep > 100 KB gzipped (web) or > 5 MB (mobile).

---

## Step 6: Classify

For each dependency, assign:

| Status | Meaning | Action |
|---|---|---|
| **CRITICAL** | CVE with public exploit, CVSS ≥ 7 | Patch THIS WEEK |
| **HIGH** | Major upgrade available, current is EOL, moderate CVE | Patch within 30 days |
| **MEDIUM** | Minor / patch version behind, no known CVE | Patch next planned cycle |
| **LOW** | Current, maintained, no action needed | Review in next audit |
| **ABANDONED** | Last release > 2 years + open CVEs + no response | Plan replacement |
| **UNUSED** | In lockfile but not imported | Remove |

Find unused deps:
```bash
npx depcheck                  # Node
./gradlew :app:assembleRelease  # Android — unused deps shown in build log
```

---

## Step 7: Upgrade Plan (not a free-for-all)

Rules for planning:
1. **Patch versions** (1.2.3 → 1.2.4) — batch them, merge quickly
2. **Minor versions** (1.2.x → 1.3.0) — one at a time, run tests
3. **Major versions** (1.x → 2.x) — individual PRs with migration notes, test thoroughly
4. **Security patches** — out-of-band, ASAP, isolated commit

Never "upgrade everything in one PR." Every upgrade is a potential breaking change; bisecting a break is hard when 20 things changed.

### Lockfile hygiene
- Commit lockfiles — always (`package-lock.json`, `Package.resolved`, `Gemfile.lock`, `go.sum`, etc.)
- Range pins (`^1.2.3`, `~1.2.3`) allowed in manifest; lockfile keeps builds reproducible
- Exception: critical deps with known-stable APIs can be exact-pinned

---

## Step 8: Report

```
DEPENDENCY AUDIT
══════════════════════════════════════
Project:       {name}
Ecosystem:     {npm / pip / gradle / spm / ...}
Total deps:    {n} direct, {n} transitive
Locked?        YES / NO

SECURITY
  Critical CVEs:  {n}   (list each: CVE-ID, pkg, version, fix)
  High CVEs:      {n}
  Moderate CVEs:  {n}

VERSION HEALTH
  On latest:       {n}%
  Behind patch:    {n}
  Behind minor:    {n}
  Behind major:    {n}
  Abandoned:       {n}   (list each)

LICENSE
  Approved:        {n}
  Flagged:         {n}   (list each with license type)

SIZE (if applicable)
  Heaviest deps:   {top 5 with sizes}

UNUSED DEPS: {n}   (list each)

RECOMMENDED ACTIONS (prioritized):
  1. PATCH THIS WEEK — {critical CVE fixes}
  2. REMOVE NOW — {unused deps}
  3. UPGRADE MAJOR — {list with breaking-change notes}
  4. REPLACE — {abandoned deps + suggested replacements}
  5. MONITOR — {deps showing signs of abandonment}

VERDICT: HEALTHY / NEEDS ATTENTION / CRITICAL
```

---

## Step 9: Automate

To keep this from becoming a quarterly fire drill:
- **Dependabot / Renovate** — auto-PR for patch + minor updates
- **CI job** — `npm audit` / `pip-audit` / `govulncheck` on every PR; fail on critical
- **Scheduled job** — weekly `dependencies` audit report to the team
- **License allowlist** — CI fails on disallowed licenses

---

## Hard rules

- **Never ignore a critical CVE.** Patch or document the mitigation.
- **Never upgrade majors without reading the migration guide + testing.**
- **Never keep a dep you don't use.** Remove unused deps on sight.
- **Never allow unpinned versions in production builds.** Lockfile in git.
- **Never skip license checks.** A wrong license can kill a release.

---

## Completion

- **HEALTHY** — no critical issues, all deps maintained
- **NEEDS ATTENTION** — fix plan produced, owners + dates assigned
- **CRITICAL** — stop-the-world: active CVEs need patching before next ship


---

## CRITICAL LESSON LEARNED — Build Verification

"npm audit" showing 0 issues doesn't mean you're safe — it only scans registered CVEs. Review your SBOM against actual usage; reachability matters.

## CRITICAL LESSON LEARNED — Acceptance Criteria Verification

After any upgrade, run the full test suite + smoke test the app manually. Transitive dep breakage often bypasses unit tests.

## CRITICAL LESSON LEARNED — Branch Scope Discipline

Dependency updates land on a dedicated branch, reviewed independently. Never bundle a dep upgrade into a feature PR — it hides the change and breaks bisecting.
