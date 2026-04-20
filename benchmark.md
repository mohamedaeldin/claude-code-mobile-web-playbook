# Benchmark — Performance Audit Across Platforms

Run performance analysis and establish baselines. Covers build times, runtime performance, bundle sizes, and platform-specific metrics.

## Triggers
Use when: "benchmark", "performance audit", "speed check", "optimize", "why is it slow", "perf review"

---

## Step 1: Detect Platform

```bash
ls *.xcodeproj 2>/dev/null && echo "PLATFORM:ios"
ls build.gradle* 2>/dev/null && echo "PLATFORM:android"
[ -f package.json ] && echo "PLATFORM:web"
[ -f go.mod ] || [ -f requirements.txt ] || [ -f Gemfile ] && echo "PLATFORM:backend"
```

---

## Step 2: Build Performance

### iOS
```bash
# Clean build time
time xcodebuild build -scheme <scheme> -destination 'platform=iOS Simulator,name=iPhone 16' -quiet 2>&1 | tail -5
# App size
ls -lh build/Build/Products/Debug-iphonesimulator/*.app 2>/dev/null
```

### Android
```bash
# Clean build time
time ./gradlew assembleDebug 2>&1 | tail -5
# APK/AAB size
ls -lh app/build/outputs/apk/debug/*.apk 2>/dev/null
ls -lh app/build/outputs/bundle/release/*.aab 2>/dev/null
```

### Web
```bash
# Build time
time npm run build 2>&1 | tail -10
# Bundle analysis
npx vite-bundle-visualizer 2>/dev/null || npx webpack-bundle-analyzer 2>/dev/null
# Bundle size
ls -lh dist/ build/ .next/ out/ 2>/dev/null
du -sh dist/ build/ .next/ out/ 2>/dev/null
```

### Backend
```bash
# Build time (compiled languages)
[ -f go.mod ] && time go build ./... 2>&1
[ -f Cargo.toml ] && time cargo build --release 2>&1 | tail -5
# Binary size
ls -lh target/release/ 2>/dev/null | head -5
```

---

## Step 3: Runtime Performance

### iOS
- **Launch time:** Check for heavy `didFinishLaunching` work
- **Scroll performance:** Check for main thread blocking in cell/row setup
- **Memory:** Check for unbounded caches, image memory
- **Network:** Check for redundant API calls, missing caching
- **Battery:** Check for unnecessary background work, location tracking, timers

### Android
- **Startup:** Check `Application.onCreate()`, lazy vs eager initialization
- **ANR risk:** Find main thread blocking (>5s operations)
- **Memory:** Check for leaks, large bitmap allocations
- **Network:** Check for redundant calls, missing OkHttp caching
- **Battery:** Check for WakeLock usage, frequent alarms, location updates

### Web
- **Core Web Vitals:**
  - LCP (Largest Contentful Paint) — target <2.5s
  - FID (First Input Delay) — target <100ms  
  - CLS (Cumulative Layout Shift) — target <0.1
- **Bundle size:** Per-route code splitting, tree shaking effectiveness
- **Render performance:** Check for unnecessary re-renders, expensive computations
- **Network:** Waterfall analysis, critical path optimization

### Backend
- **Response time:** Identify slow endpoints
- **Database queries:** Find N+1 queries, missing indexes, slow queries
- **Memory:** Check for unbounded caches, connection leaks
- **Throughput:** Requests per second capacity

---

## Step 4: Optimization Opportunities

For each finding, report:
```
OPPORTUNITY: {description}
  Current: {metric}
  Target: {expected after fix}
  Effort: {low/medium/high}
  Impact: {low/medium/high}
  Fix: {specific recommendation}
```

Prioritize by impact/effort ratio.

---

## Step 5: Output

```
PERFORMANCE BENCHMARK
═══════════════════════════════════
Platform: {detected}

BUILD
  Clean build: {time}
  Incremental: {time}
  Output size: {size}

RUNTIME
  {platform-specific metrics}

TOP OPPORTUNITIES (by impact/effort)
  1. {high impact, low effort} — {recommendation}
  2. {high impact, medium effort} — {recommendation}
  3. {medium impact, low effort} — {recommendation}

BOTTLENECK: {the single biggest performance issue}
FIX: {the one thing to do first}
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

