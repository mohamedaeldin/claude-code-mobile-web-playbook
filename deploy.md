# Deploy — Backend & Web Deployment

Deploy backend services and web applications. Handles pre-deploy checks, deployment execution, and post-deploy verification.

## Triggers
Use when: "deploy", "deploy to production", "push to staging", "release the backend", "deploy the site"

---

## Step 0: Detect Infrastructure

```bash
# CI/CD
ls .github/workflows/ 2>/dev/null && echo "CI:github-actions"
[ -f .gitlab-ci.yml ] && echo "CI:gitlab-ci"
[ -f Jenkinsfile ] && echo "CI:jenkins"
[ -f bitbucket-pipelines.yml ] && echo "CI:bitbucket"

# Hosting
[ -f vercel.json ] || [ -f .vercel ] && echo "HOST:vercel"
[ -f netlify.toml ] && echo "HOST:netlify"
[ -f fly.toml ] && echo "HOST:fly"
[ -f Dockerfile ] && echo "HOST:docker"
[ -f render.yaml ] && echo "HOST:render"
[ -f railway.json ] || [ -f railway.toml ] && echo "HOST:railway"
[ -f app.yaml ] && echo "HOST:gcp-app-engine"
ls serverless.yml serverless.ts 2>/dev/null && echo "HOST:serverless"
[ -f Procfile ] && echo "HOST:heroku"
ls terraform/ *.tf 2>/dev/null && echo "INFRA:terraform"
ls pulumi/ Pulumi.yaml 2>/dev/null && echo "INFRA:pulumi"

# Database migrations
ls db/migrate/ migrations/ alembic/ 2>/dev/null && echo "HAS:migrations"
```

Ask via AskUserQuestion:
- A) Staging/Preview
- B) Production
- C) Both (staging first, then promote)

**Production Safety Gate:** If deploying to production:
1. Verify you're on the correct branch: `git branch --show-current`
2. Verify all tests pass: run test suite for the platform
3. Verify no pending migrations that haven't been tested: check migration status
4. Ask via AskUserQuestion: "Confirm production deployment of branch {branch} to {platform}? This will affect live users."
   - A) Yes, deploy to production
   - B) Actually, deploy to staging first
   - C) Abort

**Database backup before migration:** If migrations detected:
```bash
# Suggest backup command based on detected database
echo "IMPORTANT: Back up your database before running migrations in production."
echo "Run your backup command now, then confirm to proceed."
```
Ask via AskUserQuestion: "Database migrations detected. Have you backed up the production database?"
- A) Yes, backup complete
- B) No migrations needed (data-safe changes only)
- C) Abort — I need to backup first

---

## Step 1: Pre-Deploy Checks

- [ ] All tests passing on current branch
- [ ] No uncommitted changes
- [ ] Branch is up to date with base branch
- [ ] Environment variables configured for target environment
- [ ] Database migrations reviewed and tested
- [ ] No breaking API changes (or clients are updated)
- [ ] Monitoring/alerting configured
- [ ] Rollback plan exists

### Migration Safety
If database migrations detected:
```bash
# List pending migrations
# Rails: bin/rails db:migrate:status
# Django: python manage.py showmigrations
# Node: npx prisma migrate status
# Go: goose status
```

- [ ] Migrations are reversible (can roll back)
- [ ] No destructive migrations (dropping columns/tables) without data backup
- [ ] Large table migrations won't lock tables
- [ ] Migration tested on production-like data volume

---

## Step 2: Deploy

### Vercel
```bash
vercel --prod 2>&1 | tail -10
# Or: git push (if auto-deploy configured)
```

### Netlify
```bash
netlify deploy --prod 2>&1 | tail -10
```

### Fly.io
```bash
fly deploy 2>&1 | tail -20
```

### Docker / Container
```bash
docker build -t <image> . 2>&1 | tail -10
docker push <image> 2>&1 | tail -5
# Then trigger deployment in orchestrator
```

### Railway / Render
```bash
# Usually auto-deploys on git push
git push origin <branch>
```

### GitHub Actions
```bash
# Trigger deployment workflow
gh workflow run deploy.yml -f environment=production
# Or push to deploy branch
git push origin main
```

### Custom
Read deployment docs in `README.md`, `DEPLOY.md`, or CI/CD config.

---

## Step 3: Post-Deploy Verification

```bash
# Health check with timeout and retry
for i in 1 2 3; do
  STATUS=$(curl -s -o /dev/null -w "%{http_code}" --max-time 10 <health-endpoint>)
  echo "Health check attempt $i: HTTP $STATUS"
  [ "$STATUS" = "200" ] && break
  sleep 5
done

# Check deployment status
# Vercel: vercel ls
# Fly: fly status
# Docker: docker ps
```

### Smoke Tests
- [ ] Health endpoint returns 200
- [ ] Homepage loads correctly
- [ ] Auth flow works (login/register)
- [ ] Critical API endpoints respond
- [ ] Database queries work
- [ ] Background jobs processing

### Monitoring Check
- [ ] Error tracking shows no new errors (Sentry, Bugsnag)
- [ ] Response times normal
- [ ] No memory/CPU spikes
- [ ] No increase in 5xx errors

---

## Step 4: Output

```
DEPLOYMENT
════════════════��══════════════════
Target: {staging/production}
Platform: {Vercel/Fly/Docker/etc}
URL: {deployment URL}

PRE-DEPLOY: {ALL CLEAR / N issues}
MIGRATIONS: {N applied / none needed}
DEPLOY: {SUCCESS / FAILED}
HEALTH: {HEALTHY / DEGRADED / DOWN}

POST-DEPLOY CHECKS
  Health: {pass/fail}
  Auth: {pass/fail}
  API: {pass/fail}
  Errors: {none / N new errors}

STATUS: DEPLOYED / ROLLED BACK / FAILED
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

