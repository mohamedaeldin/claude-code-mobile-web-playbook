# iOS Release — TestFlight & App Store Submission

Prepare and submit an iOS app release. Handles version bumping, build validation, TestFlight upload, and App Store submission checklist.

## Triggers
Use when: "release iOS", "submit to App Store", "TestFlight", "iOS build", "archive and upload"

---

## Step 1: Pre-Release Checklist

```bash
# Find Xcode project
ls *.xcodeproj *.xcworkspace 2>/dev/null
# Current version
grep -r 'MARKETING_VERSION' *.xcodeproj/project.pbxproj 2>/dev/null | head -1
grep -r 'CURRENT_PROJECT_VERSION' *.xcodeproj/project.pbxproj 2>/dev/null | head -1
# Check for required files
ls PrivacyInfo.xcprivacy 2>/dev/null || echo "MISSING: Privacy manifest"
```

### Version Bump
Ask via AskUserQuestion:
- A) Patch (1.0.X) — bug fixes only
- B) Minor (1.X.0) — new features, backward compatible
- C) Major (X.0.0) — breaking changes

### Pre-Flight Checks
- [ ] All tests passing
- [ ] No crashes in recent testing
- [ ] Privacy manifest (`PrivacyInfo.xcprivacy`) present and accurate
- [ ] Required reason APIs declared
- [ ] App Transport Security configured
- [ ] Info.plist usage descriptions present (camera, photos, location, etc.)
- [ ] Launch screen configured
- [ ] App icons at all required sizes
- [ ] Minimum deployment target correct
- [ ] No debug code left (print statements, test data, development URLs)
- [ ] Analytics/crash reporting SDK configured for production

---

## Step 2: Build Validation

### Pre-Build Validation
```bash
# Verify Xcode command-line tools
xcode-select -p 2>/dev/null || echo "ERROR: Xcode command-line tools not installed"
# Verify code signing
security find-identity -v -p codesigning 2>/dev/null | head -5
# Verify provisioning profiles
ls ~/Library/MobileDevice/Provisioning\ Profiles/*.mobileprovision 2>/dev/null | wc -l
```
- [ ] Code signing certificate is not expired
- [ ] Provisioning profile matches bundle ID
- [ ] ExportOptions.plist exists and is configured

```bash
# Clean build
xcodebuild clean -scheme <scheme> -quiet

# Build for release
xcodebuild archive \
  -scheme <scheme> \
  -archivePath build/<scheme>.xcarchive \
  -destination 'generic/platform=iOS' \
  -quiet 2>&1 | tail -20
```

Check for:
- [ ] Build succeeds without warnings (or warnings are acceptable)
- [ ] Archive size reasonable
- [ ] No missing entitlements
- [ ] Code signing configured correctly

---

## Step 3: App Store Metadata

Verify or create:
- [ ] App description updated for this version
- [ ] What's New text written
- [ ] Screenshots current (if UI changed)
- [ ] Keywords optimized
- [ ] Support URL valid
- [ ] Privacy policy URL valid
- [ ] Age rating accurate
- [ ] Content rights declarations correct

---

## Step 4: TestFlight Upload

```bash
# Export for App Store
xcodebuild -exportArchive \
  -archivePath build/<scheme>.xcarchive \
  -exportPath build/export \
  -exportOptionsPlist ExportOptions.plist \
  -quiet 2>&1 | tail -10

# Upload to App Store Connect
xcrun altool --upload-app \
  -f build/export/<scheme>.ipa \
  -t ios \
  --apiKey <key-id> \
  --apiIssuer <issuer-id> 2>&1 | tail -10
```

Or if using `xcodebuild -allowProvisioningUpdates`:
```bash
xcodebuild -exportArchive -archivePath build/<scheme>.xcarchive -exportPath build/export -exportOptionsPlist ExportOptions.plist -allowProvisioningUpdates
```

---

## Step 5: Post-Upload

- [ ] Build processing complete in App Store Connect
- [ ] TestFlight build available for internal testing
- [ ] Export compliance questionnaire answered
- [ ] Beta groups assigned (if applicable)
- [ ] Release notes written for testers

---

## Step 6: App Store Submission (when ready)

Ask via AskUserQuestion:
- A) Submit for review now
- B) TestFlight only — submit later
- C) Skip — just wanted the build

If A:
- [ ] Verify all metadata is final
- [ ] Select build in App Store Connect
- [ ] Choose release method (automatic after approval / manual)
- [ ] Submit for review

---

## Output

```
iOS RELEASE
═══════════════════════════════════
App: {name}
Version: {version} (build {build})
Target: iOS {min-version}+

PRE-FLIGHT: {ALL CLEAR / N issues}
BUILD: {SUCCESS / FAILED}
UPLOAD: {SUCCESS / FAILED}
TESTFLIGHT: {AVAILABLE / PROCESSING}

NEXT: {TestFlight testing / Submit for review / Fix issues}
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

