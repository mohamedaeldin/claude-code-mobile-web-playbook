# Mobile Quality & Delivery Layer

The end-to-end quality gate for shipping a mobile release. Combines functional QA, device fragmentation, performance, reliability, SDK audit, and store-submission readiness into one final validation pass. Run this BEFORE submitting to App Store or Play Store.

This skill does not replace `/qa`, `/ios-audit`, `/android-audit`, `/code-review`, `/security-audit`, or `/benchmark` — it is the final pre-release orchestrator that consumes their outputs and enforces the hard gate for production.

## Triggers
Use when: "mobile quality", "pre release mobile check", "are we ready for app store", "are we ready for play store", "final mobile validation", "ship mobile", "mobile release gate", "is the app shippable"
Proactively suggest when: the user says "ready to ship", "submit to store", "tag release", or asks for a final sign-off on a mobile app.

---

## Step 0: Detect Mobile Platform(s)

```bash
ls *.xcodeproj *.xcworkspace 2>/dev/null && echo "PLATFORM:ios"
ls build.gradle* 2>/dev/null && echo "PLATFORM:android"
[ -f pubspec.yaml ] && echo "PLATFORM:flutter"
```

If neither iOS nor Android is detected, STOP. This skill only applies to mobile projects.

---

## Step 1: Scope the Release

1. Identify the release candidate:
   - iOS: `agvtool what-marketing-version` or `Info.plist` `CFBundleShortVersionString`
   - Android: `versionName` in `app/build.gradle(.kts)`
2. Read `git log` since the last release tag.
3. Read `TODOS.md`, `README.md`, recent PR descriptions.
4. Confirm with the user: "Release candidate is `{version}` containing `{N}` commits since `{last tag}`. Proceed?"

---

## Step 2: Pick Mode

Ask via AskUserQuestion:
- A) **Quick Gate** — run HARD GATE checks only (crash/ANR/nav/data). Skip optional depth.
- B) **Standard Gate (Recommended)** — HARD GATE + device matrix + network conditions + permissions.
- C) **Full Release Audit** — everything including SDK audit, performance benchmark, sync/offline, store-submission readiness.
- D) **Report Only** — find gaps, no edits.

If mode D: read-only for the rest of the session.

---

## Step 3: Functional QA Pass

Delegate to `/qa` with the appropriate intensity:
- Quick Gate → `/qa` Quick mode
- Standard Gate → `/qa` Standard mode
- Full Release Audit → `/qa` Exhaustive mode

Record:
- Total bugs found by severity
- Bugs fixed
- Regression tests added

---

## Step 4: Mobile Quality Gate Checks

Run each check explicitly. Record PASS / FAIL / N/A per row.

### 4.1 Lifecycle Testing
- [ ] App background → foreground: state preserved, no crash
- [ ] App killed (swipe away) → reopen: auth persists, deeplink survives
- [ ] Interrupted flows (incoming call, notification, system alert, Control Center)
- [ ] Memory warning / low memory: views rebuild without state loss
- [ ] Locale change at runtime: UI re-lays out, no Arabic/English mixing
- [ ] Theme change (light ↔ dark): all screens adapt without restart

### 4.2 Network Conditions
- [ ] Offline mode: user sees clear empty/offline UI, no silent stalls
- [ ] Slow network (3G/edge): no ANR, loading states visible
- [ ] Intermittent connection: retries succeed, no duplicate writes
- [ ] Airplane mode toggle mid-request: request fails gracefully
- [ ] Token expires mid-request: refresh fires, request replays with fresh token

### 4.3 Navigation & Deep Links
- [ ] Every deeplink opens the correct screen from cold launch
- [ ] Every deeplink opens the correct screen from warm resume
- [ ] Push notification tap routes correctly when app is backgrounded
- [ ] Push notification tap routes correctly when app is killed
- [ ] Deeplink while logged out → redirect to login, then to deeplink target
- [ ] Back-navigation from deeplink target lands on a sensible parent

### 4.4 Device Matrix (Device Compatibility Engineer)
- [ ] Smallest supported screen (iPhone SE / 5.5" Android): no clipping
- [ ] Largest screen (iPhone 17 Pro Max / tablet): no wasted space
- [ ] iPad / Android tablet: layout adapts or is explicitly blocked
- [ ] RTL (Arabic): all layouts mirror correctly, no fixed-LTR screens
- [ ] Accessibility text sizes (Dynamic Type XXL / fontScale 2.0): no overflow
- [ ] Dark mode: every screen has a dark variant, no inherited white backgrounds

**Min OS versions**
- iOS: build succeeds and key flows tested on deployment target version (e.g. iOS 17)
- Android: minSdk device tested (emulator OK), compileSdk matches target

### 4.5 Permissions (Mobile QA Lead)
For each permission the app uses (location, notifications, camera, photos, contacts, calendar, microphone, background refresh):
- [ ] Denied → feature degrades gracefully, app explains what's missing
- [ ] Limited (iOS photo library "Selected Photos", Android "While Using") → works in reduced mode
- [ ] Allowed → full feature
- [ ] Re-request flow works after user flipped the switch in Settings
- [ ] Permission rationale copy matches the platform (no Android wording in iOS prompts or vice versa)

---

## Step 5: Performance & Reliability Checks

Skip to Step 6 if mode = Quick Gate.

### 5.1 Performance (Mobile Performance Engineer)
Delegate to `/benchmark` if available, or run manual checks:
- [ ] Cold launch time ≤ 3s on mid-tier device
- [ ] No frame drops on primary scroll views
- [ ] Images loaded downsampled (no full-resolution decoding for thumbnails)
- [ ] Memory steady-state < 200 MB on main flow
- [ ] No ANR (Android) or main-thread stalls > 250ms (iOS)

### 5.2 Reliability (Mobile Reliability Engineer)
- [ ] Crash reporter (Crashlytics / Sentry / MetricKit) wired up and reporting in debug
- [ ] Non-fatal error reporting in place for network/decoding failures
- [ ] Logs do not contain PII, tokens, or full request/response bodies
- [ ] No `try!`, `!!`, or force-unwrap on network or user-input code paths
- [ ] Auth session recovery tested (token expired, refresh token expired, both)

### 5.3 Sync / Offline State (Sync & Offline State Engineer)
If the app writes data while offline:
- [ ] Queued writes replay on reconnect, exactly once
- [ ] Conflict resolution documented (last-write-wins, server-wins, merge)
- [ ] Cached reads have a TTL and a manual refresh path
- [ ] App upgrade does not invalidate cached data silently

---

## Step 6: SDK & Dependency Audit

Skip if mode = Quick Gate or Standard Gate.

### 6.1 SDK Integration Audit (Mobile SDK Integration Auditor)
For each third-party SDK:
- [ ] Version pinned, not `latest` or `+`
- [ ] Release notes reviewed for regressions / API changes
- [ ] Privacy manifest (iOS) declares data collected by this SDK
- [ ] Play Data Safety (Android) declares data collected by this SDK
- [ ] License is compatible with distribution
- [ ] SDK not adding main-thread work, background wake-ups, or network chatter

### 6.2 Size Impact
- [ ] iOS IPA size delta vs previous release reviewed
- [ ] Android AAB size delta vs previous release reviewed
- [ ] No single dependency > 5 MB without justification

---

## Step 7: Store Submission Readiness

Skip if mode = Quick Gate.

### 7.1 iOS (Mobile Release Operations Manager)
- [ ] Xcode scheme set to Release
- [ ] Signing identity + provisioning profile valid and not about to expire
- [ ] Privacy manifest (`PrivacyInfo.xcprivacy`) present and accurate
- [ ] Required reason APIs declared (file timestamp, user defaults, disk space, active keyboards, system boot time)
- [ ] Encryption declaration (`ITSAppUsesNonExemptEncryption`) set correctly
- [ ] `NSUserTrackingUsageDescription` + every other `NS*UsageDescription` present if the capability is used
- [ ] App icon + launch screen present at all required sizes
- [ ] Screenshots updated for current UI
- [ ] Release notes drafted

### 7.2 Android (Mobile Release Operations Manager)
- [ ] Release build uses release keystore (NOT `debug.keystore`)
- [ ] `compileSdk` and `targetSdk` meet current Play Store minimum
- [ ] ProGuard/R8 rules tested against release build
- [ ] `mapping.txt` archived and uploaded to Play Console
- [ ] AAB (not APK) built for production
- [ ] Data Safety form matches actual data collection
- [ ] Adaptive icon provided
- [ ] Edge-to-edge rendering verified (Android 15+)
- [ ] Release notes drafted

### 7.3 Mobile Build & CI Engineer
- [ ] CI pipeline green on release branch
- [ ] Build is reproducible from a clean checkout
- [ ] No secrets committed to git or embedded in bundle
- [ ] Dependency lockfiles (`Podfile.lock`, `Package.resolved`, `gradle.lockfile`) committed

---

## Step 8: HARD GATE

No release proceeds with any of these open:

**P0 — Block release:**
- Any crash reproducible on the golden path
- Any ANR (Android) or main-thread stall > 2s
- Broken navigation — any tap that dead-ends or navigates to the wrong screen
- Data inconsistency — writes lost, duplicated, or going to the wrong user
- Auth regression — logged-out user reaches protected screens, or logged-in user forced out unexpectedly
- PII in logs or in any build that leaves the device

If any P0 is open, STOP. Do not proceed to Step 9.

---

## Step 9: Final Report

```
MOBILE QUALITY & DELIVERY REPORT
════════════════════════════════════════════
Platform(s):      {iOS / Android / both}
Release:          {version} ({build})
Mode:             {Quick / Standard / Full / Report-Only}
Branch:           {branch}

QA SUMMARY (from /qa)
  Bugs found:      {n}
  Bugs fixed:      {n}
  Regression tests added: {n}

MOBILE QUALITY GATE
  Lifecycle:       PASS / FAIL  ({n} issues)
  Network:         PASS / FAIL
  Navigation:      PASS / FAIL
  Device Matrix:   PASS / FAIL
  Permissions:     PASS / FAIL

PERFORMANCE & RELIABILITY
  Cold launch:     {time}
  ANR / stalls:    {count}
  Crash reporter:  WIRED / MISSING
  Sync / Offline:  PASS / FAIL / N/A

SDK & DEPENDENCY AUDIT
  SDKs reviewed:   {n}
  High-risk:       {n}
  Size delta:      {±MB}

STORE SUBMISSION
  iOS:             READY / NOT READY
  Android:         READY / NOT READY
  CI green:        YES / NO

P0 ISSUES: {n}
  {list each, file:line, user impact}

P1 ISSUES: {n}
P2 ISSUES: {n}

VERDICT: SHIP / HOLD / BLOCKED
{1-line summary}

NEXT STEPS
  - {if P0}: fix, re-run /mobile-quality
  - {if ready}: /release-ios OR /release-android
```

---

## Completion

- **DONE — SHIP** — All gates PASS, no P0, no P1 (or P1s accepted).
- **DONE_WITH_CONCERNS** — Gates PASS, but user accepted known risks (record them).
- **BLOCKED** — P0 open or any hard gate FAIL. Cannot ship until resolved.

---

## CRITICAL LESSON LEARNED — Build Verification

**xcodebuild CLI ≠ Xcode IDE.** The CLI with `CODE_SIGNING_ALLOWED=NO` and `generic/platform=iOS` MISSES real compiler errors that Xcode IDE catches — specifically Swift access control (`public` exposing `internal` types), module resolution, and signing-dependent validation.

**Rules:**
1. NEVER claim "zero errors" based solely on CLI output — always add: "CLI build passed — verify in Xcode IDE for full validation"
2. During code review, MANUALLY check Swift access control: `public` functions must NOT expose `internal` types in parameters or return types
3. For iOS, prefer `xcodebuild` with a specific simulator destination (e.g., `platform=iOS Simulator,name=iPhone 16`) over `generic/platform=iOS`
4. For Android, `./gradlew compileDebugKotlin` is reliable — Gradle catches all errors consistently
5. If the user reports build errors that CLI missed, acknowledge the gap immediately and fix


---

## CRITICAL LESSON LEARNED — Acceptance Criteria Verification

Acceptance criteria are pass/fail checkboxes, NOT suggestions. NEVER claim "Done" based on code review alone.

**Rules:**
1. After implementing a feature, walk through EVERY acceptance criterion and verify it works end-to-end
2. For deep links: verify BOTH the sender (widget/button URL) AND the receiver (app URL handler) are wired and functional
3. For Figma designs: download and use the ACTUAL assets from Figma immediately — never use placeholder icons with TODO comments
4. For "tapping opens X": the feature is NOT done until the tap actually opens the correct screen
5. If you cannot test (no simulator access), explicitly say "NOT VERIFIED — needs manual testing" instead of claiming "Done"
6. Never mark acceptance criteria as "Done" in delivery summaries unless the actual behavior has been verified or explicitly flagged as untested


---

## CRITICAL LESSON LEARNED — Branch Scope Discipline

When comparing branches, listing changes, or reporting what was done, ONLY include commits from the SPECIFIC branch requested. NEVER use `git log --all` or mix commits from other branches into the report.

**Rules:**
1. When the user asks about `develop`, ONLY look at `develop` commits — never include `feature/*` or other branches
2. When listing "what was done today", filter by the exact branch the user is working on
3. NEVER list items from other branches as "Not Applicable" — if they're not on the requested branch, they don't exist in the context of that request
4. Always verify which branch a commit belongs to before including it in any comparison, report, or checklist
5. Do exactly what was requested — no more, no less. Adding unrequested context from other branches creates confusion and erodes trust
