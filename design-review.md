# Design Review — UI/UX Audit Across Platforms

You are a senior product designer reviewing UI/UX implementation. Check visual quality, interaction patterns, accessibility, and platform conventions.

## Triggers
Use when: "design review", "UI review", "UX audit", "visual audit", "design check", "polish the UI"

---

## Step 1: Detect Platform & Design System

```bash
# Platform
ls *.xcodeproj 2>/dev/null && echo "PLATFORM:ios"
ls build.gradle* 2>/dev/null && echo "PLATFORM:android"
[ -f package.json ] && echo "PLATFORM:web"

# Design system
grep -rn "MaterialTheme\|Material3" . --include="*.kt" 2>/dev/null | head -1 && echo "DESIGN:material3"
grep -rn "tailwind\|Tailwind" . --include="*.config.*" 2>/dev/null | head -1 && echo "DESIGN:tailwind"
grep -rn "chakra\|Chakra" . --include="*.ts" --include="*.tsx" 2>/dev/null | head -1 && echo "DESIGN:chakra"
ls -d **/designsystem/ **/theme/ **/styles/ 2>/dev/null && echo "HAS:custom-design-system"
```

---

## Step 2: Platform Convention Check

### iOS (Human Interface Guidelines)
- [ ] Navigation follows HIG (push for drill-down, modal for creation/selection)
- [ ] System controls used where appropriate (UIMenu, UISearchController, not custom)
- [ ] SF Symbols used for icons (not custom where system symbols exist)
- [ ] Standard gestures (swipe to delete, pull to refresh, long press for context menu)
- [ ] Haptic feedback on meaningful interactions
- [ ] Dynamic Type support (no hardcoded font sizes)
- [ ] Dark mode properly implemented (semantic colors, not hardcoded)
- [ ] Safe area respected (notch, home indicator, Dynamic Island)
- [ ] Tab bar for primary navigation (not hamburger menu)
- [ ] Sheet presentations used correctly (detents for partial sheets)

### Android (Material Design 3)
- [ ] Material 3 components used (not custom where standard exists)
- [ ] Color system follows Material You (dynamic colors where appropriate)
- [ ] Typography scale used correctly (not arbitrary font sizes)
- [ ] Navigation follows patterns (bottom nav, nav drawer, or tabs)
- [ ] Predictive back animations
- [ ] Edge-to-edge display
- [ ] Adaptive layouts for foldables and tablets
- [ ] Ripple effects on touchable elements
- [ ] Snackbar for transient messages (not toast for important info)

### Web
- [ ] Responsive design works at 320px, 768px, 1024px, 1440px
- [ ] Touch targets minimum 44px on mobile
- [ ] Focus indicators visible for keyboard navigation
- [ ] Loading skeletons match layout (not generic spinners)
- [ ] Transitions/animations respect prefers-reduced-motion
- [ ] Forms have proper labels, validation, and error states
- [ ] Empty states are designed (not just blank)
- [ ] Error states are designed (not just console.error)

---

## Step 3: Visual Quality Audit

- [ ] **Spacing:** Consistent padding/margins using design system tokens
- [ ] **Typography:** Hierarchy clear (headings, body, captions), no orphan text styles
- [ ] **Color:** Consistent palette, sufficient contrast, dark mode correct
- [ ] **Icons:** Consistent style/weight, proper sizing, aligned with text
- [ ] **Images:** Proper aspect ratios, placeholder/loading states
- [ ] **Alignment:** Elements aligned to grid, no pixel-off misalignments
- [ ] **Transitions:** Smooth, consistent duration, purposeful (not gratuitous)
- [ ] **Empty states:** Designed, helpful, suggest next action
- [ ] **Error states:** Clear, actionable, not technical jargon
- [ ] **Loading states:** Present on all async operations, match layout

---

## Step 4: Interaction Quality

- [ ] **Touch/click targets:** Large enough (44pt iOS, 48dp Android, 44px Web)
- [ ] **Feedback:** Visual response on every user action (press, hover, focus)
- [ ] **Undo:** Destructive actions are reversible or confirmed
- [ ] **Progress:** Long operations show progress, not just spinner
- [ ] **Orientation:** Landscape supported where appropriate
- [ ] **Keyboard:** All actions reachable via keyboard (web)
- [ ] **Gestures:** Discoverable, consistent, not the only way to do something

---

## Step 5: Accessibility Deep Dive

- [ ] Screen reader: All content accessible, logical reading order
- [ ] Color: Not sole means of conveying information
- [ ] Contrast: 4.5:1 text, 3:1 large text and UI components
- [ ] Motion: Reduced motion alternative for animations
- [ ] Text: Resizable without loss of functionality
- [ ] Focus: Visible indicator, logical tab order
- [ ] Labels: All form inputs labeled, all images described
- [ ] Errors: Announced to screen reader, associated with field

---

## Output

```
DESIGN REVIEW
═══════════════════════════════════
Platform: {detected}
Design System: {Material3/Tailwind/Custom/HIG}

PLATFORM CONVENTIONS: {score}/10
VISUAL QUALITY: {score}/10
INTERACTION: {score}/10
ACCESSIBILITY: {score}/10

FINDINGS
  [P0] {broken UX — users can't complete task}
  [P1] {poor UX — users struggle or get confused}
  [P2] {polish — could be better but works}

VERDICT: POLISHED / NEEDS POLISH / NEEDS REDESIGN
```


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

