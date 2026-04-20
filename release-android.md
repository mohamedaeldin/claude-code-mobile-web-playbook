# Android Release — Play Store Submission

Prepare and submit an Android app release. Handles version bumping, build validation, signing, and Play Store submission checklist.

## Triggers
Use when: "release Android", "submit to Play Store", "Android build", "generate APK/AAB", "publish to Play Store"

---

## Step 1: Pre-Release Checklist

```bash
# Current version
grep -E 'versionCode|versionName' app/build.gradle* 2>/dev/null | head -5
# Min/Target SDK
grep -E 'minSdk|targetSdk|compileSdk' app/build.gradle* 2>/dev/null | head -5
# Signing config
grep -E 'signingConfigs|storeFile' app/build.gradle* 2>/dev/null | head -5
# ProGuard
ls app/proguard-rules.pro 2>/dev/null
```

### Version Bump
Ask via AskUserQuestion:
- A) Patch (1.0.X) — bug fixes only
- B) Minor (1.X.0) — new features
- C) Major (X.0.0) — breaking changes

Remember: `versionCode` must increment by at least 1 for every upload.

### Version Validation
- [ ] versionCode is HIGHER than the current Play Store version
- [ ] versionName follows semantic versioning
- [ ] Signing config uses RELEASE keystore (not debug.keystore)

```bash
# Verify signing key is not debug
grep -r "debug.keystore" app/build.gradle* 2>/dev/null && echo "WARNING: Using debug keystore!"
# Check APK size won't exceed Play Store limit
```

### Pre-Flight Checks
- [ ] All tests passing (`./gradlew test`)
- [ ] No crashes in recent testing
- [ ] Target SDK meets Play Store requirements
- [ ] Runtime permissions handled gracefully
- [ ] ProGuard/R8 rules configured (no stripped classes needed at runtime)
- [ ] Network security config correct for production
- [ ] No debug code left (Log.d, test URLs, BuildConfig.DEBUG bypasses)
- [ ] Crash reporting SDK configured (Firebase Crashlytics, Sentry)
- [ ] `android:debuggable="false"` in release build type
- [ ] Signing config uses release keystore (not debug)

---

## Step 2: Build

```bash
# Clean
./gradlew clean

# Build release AAB (for Play Store)
./gradlew bundleRelease 2>&1 | tail -20

# Or build APK (for direct distribution)
./gradlew assembleRelease 2>&1 | tail -20
```

Check for:
- [ ] Build succeeds
- [ ] AAB/APK size reasonable
- [ ] ProGuard mapping file generated (`app/build/outputs/mapping/release/mapping.txt`)
- [ ] No unsigned build

---

## Step 3: Pre-Launch Checks

- [ ] Test on minimum SDK device/emulator
- [ ] Test on latest Android version
- [ ] Test with different screen densities (mdpi, hdpi, xhdpi, xxhdpi)
- [ ] Test with system dark mode
- [ ] Test with large font size (accessibility)
- [ ] Test with TalkBack enabled
- [ ] Test with limited network (slow/offline)
- [ ] Test predictive back gesture (Android 14+)

---

## Step 4: Play Store Metadata

Verify or create:
- [ ] Store listing description updated
- [ ] What's New text written
- [ ] Screenshots current (phone, tablet, if changed)
- [ ] Feature graphic present
- [ ] Content rating questionnaire completed
- [ ] Data safety section accurate
- [ ] Privacy policy URL valid
- [ ] Target audience and content declarations accurate

---

## Step 5: Upload

```bash
# Upload AAB to Play Store (requires configured service account)
# Using bundletool for validation:
java -jar bundletool.jar validate --bundle app/build/outputs/bundle/release/app-release.aab 2>&1

# Or using Gradle Play Publisher plugin:
./gradlew publishReleaseBundle 2>&1 | tail -10
```

---

## Step 6: Release Track

Ask via AskUserQuestion:
- A) Internal testing — team only
- B) Closed testing — beta testers
- C) Open testing — public beta
- D) Production — full release
- E) Staged rollout — start at 5%/10%/25%

---

## Output

```
ANDROID RELEASE
═══════════════════════════════════
App: {name}
Version: {versionName} (code {versionCode})
Min SDK: {version} / Target SDK: {version}

PRE-FLIGHT: {ALL CLEAR / N issues}
BUILD: {SUCCESS / FAILED}
AAB SIZE: {size}
MAPPING: {saved/missing}

UPLOAD: {SUCCESS / FAILED}
TRACK: {internal/closed/open/production/staged}

NEXT: {Testing / Review / Monitor rollout}
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

