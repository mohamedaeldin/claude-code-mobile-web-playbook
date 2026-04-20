# Accessibility Audit — Multi-Platform WCAG Compliance

Deep accessibility audit of iOS, Android, or Web. Goes beyond the one-line checks in `/ios-audit`, `/android-audit`, `/web-audit` — this skill verifies real WCAG 2.1 AA compliance and produces a prioritized remediation list.

The rule: **accessibility is pass/fail, not opinion. Test with the actual assistive tech; don't just read the code.**

## Triggers
Use when: "accessibility audit", "a11y", "WCAG", "screen reader test", "VoiceOver check", "TalkBack check", "accessibility review", "keyboard navigation check"
Proactively suggest when: the user asks to "make it accessible", "pass WCAG", "support screen readers", or is preparing for government / enterprise procurement.

---

## Step 0: Detect Platform

```bash
ls *.xcodeproj *.xcworkspace 2>/dev/null && echo "PLATFORM:ios"
ls build.gradle* 2>/dev/null && echo "PLATFORM:android"
[ -f package.json ] && echo "PLATFORM:web"
```

---

## Step 1: Scope + Standard

Ask:
- **Target standard** — WCAG 2.1 AA (default)? WCAG 2.2? Section 508?
- **Scope** — whole app? Specific flows? Newly added screens only?
- **Certification needed?** — internal bar vs formal audit by third party.

---

## Step 2: Automated Scan First

Automated tooling catches ~30% of real issues. Do it first to clear the easy wins, but NEVER declare accessible based only on automation.

### iOS
```bash
# Xcode Accessibility Inspector (GUI)
# Or run the Accessibility audit via XCUIAutomation / axe-ios
```

### Android
```bash
# Accessibility Scanner app (install from Play Store on the test device)
# Or Espresso with AccessibilityChecks
./gradlew connectedAndroidTest -Pandroid.testInstrumentationRunnerArguments.accessibility=true
```

### Web
```bash
npx @axe-core/cli <url> 2>&1 | tail -50
npx lighthouse <url> --only-categories=accessibility 2>&1 | tail -30
# Or programmatic in tests:
npm install --save-dev @axe-core/react jest-axe
```

Record findings with severity (critical / serious / moderate / minor).

---

## Step 3: Manual Screen Reader Test

This is what matters. Automation misses ambiguous labels, bad reading order, trapped focus.

### iOS — VoiceOver
1. Settings → Accessibility → VoiceOver → ON (or triple-click side button)
2. Swipe right through every screen. Each element should:
   - Be reachable (not skipped)
   - Announce a meaningful label (not "button" or "image")
   - Announce its state (selected, disabled, loading)
   - Announce its role ("button", "heading", "link")
3. Two-finger swipe up reads whole screen top-to-bottom — reading order correct?
4. Rotor (two-finger rotate) — headings, form controls, landmarks all navigable?

### Android — TalkBack
1. Settings → Accessibility → TalkBack → ON
2. Swipe right through every screen — same checks as iOS.
3. Check touch exploration — tap anywhere and listen.
4. Three-finger swipe down → reading controls menu works.

### Web — NVDA / VoiceOver / JAWS
1. Tab through the entire page — can you reach every control?
2. Shift+Tab — reverse order makes sense?
3. Screen reader navigation keys:
   - H / 1–6 — jump between headings
   - L — jump between lists
   - F — jump between form controls
   - D / landmarks — jump between regions
4. Forms mode vs browse mode transitions correctly?
5. Modal dialogs: focus trapped inside while open, returns to trigger on close?

---

## Step 4: Keyboard-Only Test (Web)

Unplug your mouse (or ignore it). Can you:
- [ ] Navigate every interactive element with Tab / Shift+Tab
- [ ] See a visible focus indicator at all times
- [ ] Activate buttons with Enter and Space
- [ ] Escape modals with Esc
- [ ] Reach everything in dropdowns with arrow keys
- [ ] Skip repetitive content with a "skip to main" link

If any no → fail.

---

## Step 5: Visual + Motor Tests

### Zoom / Dynamic Type
- iOS: Settings → Accessibility → Display & Text Size → Larger Text → XXXL
- Android: Settings → Accessibility → Font size → Largest
- Web: Cmd/Ctrl + multiple times (200%, 400%)

Every screen should remain usable — no text clipped, no buttons offscreen.

### Color contrast
- Use a contrast checker (axe, WebAIM Contrast Checker, Xcode Accessibility Inspector)
- Text ≥ 4.5:1 against its background
- Large text (18pt+ regular or 14pt+ bold) ≥ 3:1
- UI controls + state indicators ≥ 3:1

### Color alone
- No "click the red button" / "errors shown in red" — also needs text, icon, or shape.

### Motion
- `prefers-reduced-motion` respected (web + mobile): no auto-playing animations, no parallax, no bouncy transitions for users who opt out.
- No flashing content (3+ flashes per second = seizure risk).

### Touch targets
- Minimum 44×44 pt (iOS) or 48×48 dp (Android). Web: ≥ 44×44 CSS pixels.

---

## Step 6: Cognitive + Language

- [ ] Form errors are specific ("Email is required" not "Invalid input")
- [ ] Errors associated with the field via `aria-describedby` / `accessibilityLabel`
- [ ] Language of the page declared (`<html lang="en">`, `UIAccessibilityTraits.language`)
- [ ] Time limits can be extended, paused, or turned off
- [ ] Auto-playing audio/video has a pause control
- [ ] Text is plain language where possible; jargon has definitions

---

## Step 7: Platform-Specific Deep Dives

### iOS
- [ ] Every custom control has correct `accessibilityTraits` (button, header, selected, etc.)
- [ ] `accessibilityElementsHidden` / `shouldGroupAccessibilityChildren` used correctly for grouping
- [ ] Custom actions registered via `accessibilityCustomActions` for swipe-only features
- [ ] Hit areas ≥ 44×44 pt
- [ ] Images have `accessibilityLabel` (or `.isAccessibilityElement = false` if decorative)

### Android
- [ ] `contentDescription` on every non-text interactive Composable
- [ ] Decorative images marked with `Modifier.semantics { invisibleToUser() }`
- [ ] `semantics { heading() }` on headings
- [ ] `semantics { liveRegion = ... }` on dynamically updating text
- [ ] Touch targets ≥ 48dp (use `Modifier.minimumInteractiveComponentSize()`)
- [ ] Focus order correct via `Modifier.semantics { traversalIndex = ... }`

### Web
- [ ] Landmarks present: `<main>`, `<nav>`, `<header>`, `<footer>`, `<aside>`
- [ ] Heading hierarchy: one `<h1>`, logical nesting, no skipped levels
- [ ] Form labels associated via `for` / `htmlFor` or `aria-labelledby`
- [ ] Buttons use `<button>` not `<div onClick>`
- [ ] Links use `<a href>` not `<button>` for navigation
- [ ] `aria-live` on status messages that appear dynamically
- [ ] `aria-hidden="true"` on decorative elements only — never on focusable elements
- [ ] Modal: `role="dialog"` + `aria-modal="true"` + focus trap
- [ ] Input types set correctly (`type="email"`, `type="tel"`, `autocomplete` attributes)

---

## Step 8: Report

```
ACCESSIBILITY AUDIT
══════════════════════════════════════
Platform:   {iOS / Android / Web}
Standard:   WCAG 2.1 AA
Scope:      {flows / screens}

AUTOMATED FINDINGS: {n}
  [critical] {n}
  [serious]  {n}
  [moderate] {n}

MANUAL FINDINGS: {n}
  [P0 — blocking] {n}    ← must fix
  [P1 — major]    {n}    ← should fix
  [P2 — minor]    {n}    ← nice to fix

KEY ISSUES:
  1. {location} — {issue} — WCAG {criterion} — {suggested fix}
  2. ...

NOT TESTED (acknowledge + defer):
  - {scope gap}

VERDICT: PASS / NEEDS WORK / FAIL
```

Attach screenshots or screen recordings for non-obvious issues.

---

## Hard rules

- **Never declare accessible based on automated scan alone.** Automation catches 30%.
- **Never skip manual screen reader testing.** 10 minutes per flow is non-negotiable.
- **Never ship a screen without testing Dynamic Type / zoom at max.**
- **Never use color as the only indicator of state.**
- **Never trap focus in a modal without a way out.**

---

## Completion

- **PASS** — all P0 + P1 issues fixed, documented deferrals reviewed
- **NEEDS WORK** — P0/P1 list produced, remediation plan drafted
- **FAIL** — systemic issues, redesign needed before fixes


---

## CRITICAL LESSON LEARNED — Build Verification

Accessibility can't be verified in an IDE or linter. It requires actual assistive tech on actual devices. No substitute.

## CRITICAL LESSON LEARNED — Acceptance Criteria Verification

"Passes axe" is NOT "accessible." Walk every user flow with a screen reader + keyboard only. Record findings with severity. Commit to a fix date per P0/P1.

## CRITICAL LESSON LEARNED — Branch Scope Discipline

When auditing, audit the actual shipping code (release branch, main, or tag) — not a work-in-progress feature branch. Users interact with shipped code.
