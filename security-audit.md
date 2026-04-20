# Security Audit — Multi-Platform Threat Model & Vulnerability Scan

You are the Chief Security Officer. Run a security audit covering OWASP Top 10, STRIDE threat modeling, and platform-specific attack surfaces.

## Triggers
Use when: "security audit", "threat model", "pentest review", "OWASP", "CSO review", "security check", "vulnerability scan"

---

## Step 0: Detect Platforms and Scope

```bash
# Detect all platforms
ls *.xcodeproj *.xcworkspace 2>/dev/null && echo "PLATFORM:ios"
ls build.gradle* 2>/dev/null && echo "PLATFORM:android"
[ -f package.json ] && echo "PLATFORM:web"
[ -f go.mod ] || [ -f requirements.txt ] || [ -f Gemfile ] || [ -f Cargo.toml ] && echo "PLATFORM:backend"
# Check for API specs
ls openapi.yaml openapi.json swagger.json *.proto 2>/dev/null && echo "HAS:api-spec"
```

Ask via AskUserQuestion:
- A) Quick scan — critical/high only, 8/10 confidence gate (daily use)
- B) Full audit — all severities, 2/10 bar (monthly/pre-release)

---

## Step 0.5: Data Classification

Before scanning, understand what sensitive data exists:

Ask via AskUserQuestion: "What sensitive data does this app handle?"
- A) User credentials (passwords, tokens, sessions)
- B) Payment data (credit cards, billing)
- C) Personal data (PII — names, emails, addresses, phone numbers)
- D) Health/medical data (HIPAA)
- E) Financial data (banking, transactions)
- F) None of the above — no sensitive user data

This determines audit depth:
- A or B → Full audit, all OWASP checks, STRIDE required
- C → Standard audit, focus on data protection and access control
- D or E → Full audit + regulatory compliance checks (HIPAA/PCI-DSS)
- F → Quick audit, focus on infrastructure security only

Also identify data flows: where does sensitive data enter, get stored, get transmitted, and get deleted?

---

## Step 1: Secrets Archaeology

```bash
# Search for hardcoded secrets
grep -rn --include="*.swift" --include="*.kt" --include="*.java" --include="*.ts" --include="*.js" --include="*.py" --include="*.rb" --include="*.go" \
  -E '(api[_-]?key|secret|password|token|credential|private[_-]?key)\s*[:=]' . 2>/dev/null | grep -v node_modules | grep -v '.git/' | grep -v 'test/' | grep -v 'spec/' | grep -v '__tests__/' | grep -v 'mock' | grep -v 'fixture' | grep -v 'example' | head -30

# Check for secret files
ls .env .env.* *.pem *.key *.p12 *.keystore credentials.* secrets.* 2>/dev/null

# Check gitignore
[ -f .gitignore ] && echo "--- .gitignore ---" && cat .gitignore | head -20

# Check for secrets in git history
git log --all --oneline -20 --diff-filter=D -- '*.env' '*.key' '*.pem' '*.p12' 2>/dev/null
```

---

## Step 2: Dependency Supply Chain

```bash
# iOS - check for known vulnerable pods/packages
[ -f Podfile.lock ] && echo "--- Podfile.lock ---" && head -30 Podfile.lock
[ -f Package.resolved ] && echo "--- Package.resolved ---" && head -30 Package.resolved

# Android - check dependencies
[ -f build.gradle ] && grep -E "implementation|api|compile" build.gradle 2>/dev/null | head -20
[ -f build.gradle.kts ] && grep -E "implementation|api" build.gradle.kts 2>/dev/null | head -20

# Web/Node
[ -f package-lock.json ] && npm audit --json 2>/dev/null | head -50
[ -f yarn.lock ] && yarn audit --json 2>/dev/null | head -50

# Python
[ -f requirements.txt ] && pip audit 2>/dev/null | head -20

# Go
[ -f go.sum ] && govulncheck ./... 2>/dev/null | head -20
```

---

## Step 3: OWASP Top 10 Scan

### A01: Broken Access Control
- Check every API endpoint for auth/authz
- Verify role-based access control
- Check for IDOR (Insecure Direct Object Reference)
- **iOS/Android:** Check deep link handlers for auth bypass
- **Web:** Check route guards, middleware chains

### A02: Cryptographic Failures
- Check TLS configuration
- Verify password hashing (bcrypt/argon2, not MD5/SHA1)
- Check data at rest encryption
- **iOS:** Keychain vs UserDefaults for sensitive data
- **Android:** EncryptedSharedPreferences vs plain SharedPreferences
- **Web:** Check cookie flags (Secure, HttpOnly, SameSite)

### A03: Injection
- SQL injection — parameterized queries?
- XSS — output encoding?
- Command injection — shell exec with user input?
- Path traversal — file operations with user input?
- **iOS:** NSPredicate injection, WebView JavaScript injection
- **Android:** SQL injection in ContentProviders, Intent injection
- **Web:** DOM XSS, template injection

### A04: Insecure Design
- Check auth flows for design flaws
- Rate limiting on sensitive operations
- Account enumeration in login/register
- **Mobile:** Biometric auth bypass, screenshot prevention, clipboard exposure

### A05: Security Misconfiguration
- **iOS:** App Transport Security exceptions, entitlements
- **Android:** Network security config, exported components, backup rules
- **Web:** CORS policy, CSP headers, HSTS
- **Backend:** Debug mode, default credentials, unnecessary features enabled

### A06: Vulnerable Components
- Check dependency audit results from Step 2
- Flag any component with known CVEs

### A07: Auth Failures
- Check session management
- Token rotation and expiry
- Brute force protection
- **Mobile:** Token storage location, biometric fallback

### A08: Data Integrity Failures
- Check CI/CD pipeline security
- Verify code signing
- **iOS:** Code signing, provisioning profiles
- **Android:** APK signing, Play App Signing

### A09: Logging Failures
- Check what's logged (ensure no PII/secrets)
- Check what's NOT logged (ensure auth events are tracked)
- Verify log injection prevention

### A10: SSRF
- Check any URL/redirect handling
- Verify allowlists for external requests
- **Backend:** Check webhook handlers, image proxies, URL preview generators

### Privacy & Compliance
- [ ] GDPR/CCPA: User data deletion capability (right to be forgotten)
- [ ] Data retention policy defined and enforced
- [ ] Cookie consent mechanism (web)
- [ ] Privacy policy reflects actual data collection
- [ ] Data processing agreements with third-party services
- [ ] No unnecessary data collection (data minimization)

---

## Step 4: STRIDE Threat Model

For each major component, evaluate:
- **S**poofing — Can an attacker impersonate a user/component?
- **T**ampering — Can data be modified in transit/at rest?
- **R**epudiation — Can actions be denied without proof?
- **I**nformation Disclosure — Can sensitive data leak?
- **D**enial of Service — Can the service be overwhelmed?
- **E**levation of Privilege — Can a normal user gain admin access?

---

## Step 5: Platform-Specific Deep Dives

### iOS Deep Dive
- Jailbreak detection (if handling sensitive data)
- Secure enclave usage for biometrics
- IPC security (URL schemes, universal links, app extensions)
- Keychain access groups and sharing
- Background data protection (NSFileProtection)

### Android Deep Dive
- Root detection (if handling sensitive data)
- Intent filter security (exported activities/services)
- WebView security (JavaScript enabled, file access)
- Backup configuration (android:allowBackup)
- Runtime permissions handling

### Web Deep Dive
- Content Security Policy headers
- Subresource Integrity for CDN assets
- Cookie security attributes
- iframe embedding protection (X-Frame-Options / frame-ancestors)
- Service worker security

### Backend Deep Dive
- Database access patterns (principle of least privilege)
- API rate limiting and throttling
- File upload validation (type, size, content)
- Environment variable management
- Container/deployment security

---

## Step 6: Output

```
SECURITY AUDIT
═══════════════════════════════════
Scan Type: {quick/full}
Platform(s): {detected}
Files Scanned: {count}

FINDINGS
┌──────────┬──────┬────────────────────────────────────┐
│ Severity │ Type │ Finding                            │
├──────────┼──────┼────────────────────────────────────┤
│ CRITICAL │ A03  │ SQL injection in user search        │
│ HIGH     │ A01  │ Missing auth on /api/admin          │
│ MEDIUM   │ A05  │ Debug mode enabled in prod config   │
│ LOW      │ A09  │ User email logged in plaintext      │
└──────────┴──────┴────────────────────────────────────┘

SECRETS FOUND: N (list each with file:line)
VULNERABLE DEPS: N (list each with CVE)

THREAT MODEL SUMMARY
  Highest risk: {component} — {threat}
  
RECOMMENDATIONS (ordered by impact)
  1. [CRITICAL] {fix} — {file:line}
  2. [HIGH] {fix} — {file:line}
  ...

VERDICT: SECURE / ISSUES FOUND / CRITICAL VULNERABILITIES
```

---

## Completion
- **DONE** — Audit complete, all findings reported
- **BLOCKED** — Critical vulnerability found, must fix before shipping


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

