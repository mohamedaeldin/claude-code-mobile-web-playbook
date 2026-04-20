# Design System — Create or Audit Design Tokens & Components

Create or audit a design system with tokens, components, and documentation. Works across iOS (SwiftUI), Android (Compose), and Web (CSS/Tailwind).

## Triggers
Use when: "design system", "create theme", "setup colors", "design tokens", "component library", "brand setup"

---

## Step 1: Inventory Existing System

```bash
# Check for existing design system
find . -name "Colors*" -o -name "Theme*" -o -name "Typography*" -o -name "Spacing*" -o -name "tokens*" 2>/dev/null | head -10
find . -name "*.swift" -path "*Theme*" -o -name "*.kt" -path "*Theme*" 2>/dev/null | head -5
ls tailwind.config.* theme.ts theme.js tokens.json 2>/dev/null
```

If existing: audit for completeness. If missing: create from scratch.

---

## Step 2: Define Tokens

### Color Tokens
```
Primary: {brand color}
Secondary: {accent color}
Background: {surface colors — light + dark}
Text: {primary, secondary, disabled — light + dark}
Error/Warning/Success/Info: {semantic colors}
```

### Typography Tokens
```
Heading 1-6: {font, weight, size, line height}
Body: {regular, bold}
Caption: {small text}
Code: {monospace}
```

### Spacing Tokens
```
xs: 4pt    sm: 8pt    md: 16pt    lg: 24pt    xl: 32pt    xxl: 48pt
```

### Other Tokens
```
Border radius: sm/md/lg/full
Shadow: sm/md/lg
Animation duration: fast/normal/slow
```

---

## Step 3: Platform Implementation

### iOS (SwiftUI)
Create:
- `Theme/Colors.swift` — semantic colors with light/dark variants
- `Theme/Typography.swift` — font styles as ViewModifiers
- `Theme/Spacing.swift` — spacing constants
- `Theme/Components/` — reusable SwiftUI components

### Android (Compose)
Create:
- `ui/theme/Color.kt` — MaterialTheme color scheme
- `ui/theme/Type.kt` — Typography definition
- `ui/theme/Theme.kt` — App theme composable
- `ui/components/` — reusable composables

### Web (Tailwind/CSS)
Create or update:
- `tailwind.config.ts` — custom colors, fonts, spacing
- `styles/tokens.css` — CSS custom properties
- `components/` — shared UI components

---

## Step 4: Component Checklist

For each platform, ensure these base components exist:
- [ ] Button (primary, secondary, outlined, text, icon, loading, disabled)
- [ ] Text input (single-line, multiline, password, with validation)
- [ ] Card / Surface
- [ ] List item (with icon, subtitle, action)
- [ ] Dialog / Alert
- [ ] Bottom sheet / Modal
- [ ] Loading indicator (spinner, skeleton, progress bar)
- [ ] Toast / Snackbar
- [ ] Badge / Chip / Tag
- [ ] Avatar / Image placeholder
- [ ] Empty state
- [ ] Error state

---

## Output

```
DESIGN SYSTEM
═══════════════════════════════════
Platform(s): {detected}
Status: {CREATED / AUDITED / UPDATED}

TOKENS
  Colors: {count} defined
  Typography: {count} styles
  Spacing: {count} values

COMPONENTS: {count} / {expected}
  Missing: {list}

DARK MODE: {COMPLETE / PARTIAL / MISSING}
ACCESSIBILITY: {COMPLIANT / N issues}

FILES CREATED/MODIFIED: {list}
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

