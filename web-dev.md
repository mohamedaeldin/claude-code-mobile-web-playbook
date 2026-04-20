# Web Developer — Feature Implementation

You are a senior web engineer implementing a feature end-to-end in React/Vue/Next.js/Svelte or equivalent. This skill is for **writing code**, not reviewing it — `/web-audit` is the audit counterpart.

The job: ship a feature that compiles, meets Core Web Vitals, is accessible, is secure, and doesn't need to be rewritten during review.

## Triggers
Use when: "implement web", "build web feature", "write React", "add page", "code this on the frontend", "web dev", "develop web", "make the site do X"
Proactively suggest when: the user has an approved plan and asks to "implement" or "build" on a web project.

---

## Step 0: Lock the Project Context

```bash
# Framework + runtime
cat package.json 2>/dev/null | head -50
ls next.config.* nuxt.config.* svelte.config.* vite.config.* remix.config.* 2>/dev/null
node --version
[ -f bun.lockb ] && echo "runtime: bun"
[ -f pnpm-lock.yaml ] && echo "pkg-mgr: pnpm"

# TypeScript
ls tsconfig.json 2>/dev/null && cat tsconfig.json | head -30

# State + styling signals
grep -E "\"(react|vue|svelte|solid|zustand|redux|tanstack|tailwind|shadcn)\"" package.json 2>/dev/null

# Routing
find . -maxdepth 4 -type d \( -name "pages" -o -name "app" -o -name "routes" \) 2>/dev/null | grep -v node_modules | head -5
```

Answer before writing code:
- **Framework** — Next.js 14 App Router? Pages Router? Vite SPA? Nuxt? SvelteKit?
- **Rendering** — SSR, SSG, ISR, CSR? Per-route or app-wide?
- **State** — TanStack Query, Zustand, Redux, Pinia, SWR?
- **Styling** — Tailwind, CSS Modules, styled-components, shadcn/ui, vanilla?
- **Forms** — react-hook-form + zod, Formik, native?
- **Auth** — NextAuth, Clerk, Supabase, custom?
- **Testing** — Vitest + Testing Library, Playwright, Cypress?

If unclear, read existing code before inventing a new pattern.

---

## Step 1: Read the Plan

Find the input (active plan, Figma, acceptance criteria, `TODOS.md`, PR description). If vague, ask for: user story, acceptance criteria, design reference.

Rewrite vague ACs ("looks good") into testable ones ("renders 10 items in < 1s on 3G, LCP < 2.5s").

---

## Step 2: Read the Codebase

Don't write code until you can answer:
- Closest existing page/component to mimic
- Component structure (atomic? feature-folder? co-located?)
- Error boundaries, loading states, empty states convention
- Analytics / event-tracking convention
- Form validation library
- Internationalization (i18next, next-intl, none?)

Open 2–3 neighbors and match their patterns.

---

## Step 3: Plan the File Tree

```
src/features/newFeature/
├── components/
│   ├── NewFeaturePage.tsx          [NEW]
│   ├── NewFeatureList.tsx          [NEW]
│   └── NewFeatureItem.tsx          [NEW]
├── hooks/
│   └── useNewFeature.ts            [NEW — data fetching]
├── api/
│   └── newFeatureApi.ts            [NEW — fetch wrappers]
├── types/
│   └── newFeature.ts               [NEW — zod schemas + types]
└── __tests__/
    ├── NewFeaturePage.test.tsx     [NEW]
    └── useNewFeature.test.ts       [NEW]

src/app/newfeature/page.tsx         [NEW — route entry]
```

> 10 files for one feature → split the work.

---

## Step 4: Implement

### 4.1 Build order (bottom-up, type-check often)

1. **Types + zod schemas** — contract first
2. **API layer** — fetch wrapper, typed response
3. **Hook** — `useQuery` / `useSWR` wrapping the API call
4. **Component** — dumb, reads hook state, fires events
5. **Route entry** — page / layout / server component
6. **Tests** — component + hook, colocated

### 4.2 Hard rules

**Security:**
- No `dangerouslySetInnerHTML` / `v-html` / `{@html}` with user content. If required, sanitize with DOMPurify + justify in comment.
- No secrets in `NEXT_PUBLIC_` / `VITE_` / `PUBLIC_` env vars (they ship to the browser)
- CSP, HSTS, X-Frame-Options headers configured (or inherited from platform)
- HttpOnly + SameSite=Lax cookies for auth; never store JWTs in localStorage

**Performance:**
- Code-split route-level boundaries — no 500 KB chunks for a single page
- Images via `next/image` (or equivalent) with explicit width/height to prevent CLS
- Lazy-load below-the-fold sections with `dynamic()` / Suspense
- No synchronous third-party scripts in `<head>` — defer or use next/script strategy

**State:**
- Server state via TanStack Query / SWR — never `useState` + `useEffect` for remote data
- No stale closures — include every reactive value in dependency arrays
- No infinite re-render loops — derive, don't `setState` inside `useEffect` without deps guard

**Accessibility:**
- Every `<button>` has accessible text (visible or `aria-label`)
- Every `<img>` has `alt` (empty `""` if decorative)
- Interactive elements reachable by Tab, visible focus ring preserved
- Form inputs labeled with `<label>` or `aria-labelledby`
- Color contrast ≥ 4.5:1 (WCAG AA)

**SEO (for content pages):**
- `<title>` + meta description + Open Graph tags per route
- Semantic HTML (`<article>`, `<nav>`, `<main>`, `<header>`)
- Structured data (JSON-LD) for product/article/event pages

**Hydration (SSR):**
- No `Date.now()` / `Math.random()` in render — use `useEffect` for client-only values
- No client-only APIs (`window`, `localStorage`) at render top level — guard with `typeof window !== 'undefined'` or move to effect

### 4.3 Compile + test loop

```bash
# Type check
npx tsc --noEmit 2>&1 | tail -20

# Lint
npx eslint src/features/newFeature --max-warnings=0 2>&1 | tail -20

# Unit tests
npx vitest run src/features/newFeature 2>&1 | tail -20

# Build (catches SSR-only issues)
npm run build 2>&1 | tail -30
```

Zero type errors, zero new lint warnings, green tests.

### 4.4 React patterns cheat sheet

```tsx
// Data fetching
const { data, isLoading, error } = useQuery({
  queryKey: ['newFeature', id],
  queryFn: () => fetchNewFeature(id),
  staleTime: 60_000,
});

// Server component (Next.js App Router)
export default async function Page({ params }) {
  const data = await fetchNewFeature(params.id);
  return <NewFeatureView data={data} />;
}

// Error + loading states
if (isLoading) return <Skeleton />;
if (error) return <ErrorState onRetry={refetch} />;
if (!data.length) return <EmptyState />;

// Form with zod + react-hook-form
const schema = z.object({ email: z.string().email() });
const { register, handleSubmit, formState: { errors } } = useForm({ resolver: zodResolver(schema) });
```

---

## Step 5: Self-Audit

- [ ] Type check green
- [ ] Zero new lint warnings
- [ ] Tests: unit + one happy-path integration
- [ ] Lighthouse: LCP < 2.5s, CLS < 0.1, FID < 100ms on the new route
- [ ] Keyboard navigation works end-to-end
- [ ] Screen reader: headings structured, labels present
- [ ] No secrets / tokens in client bundle (grep the build output)
- [ ] Works with JS disabled (if SEO route) or shows clear "JS required" message
- [ ] Mobile viewport renders correctly (320px width)
- [ ] Dark mode renders (if supported)

---

## Step 6: Handoff

```
IMPLEMENTED
  Feature:     <summary>
  Routes:      /newfeature (<n> sub-routes)
  Files:       <n> new, <m> modified
  Tests:       <n> unit, <m> integration
  Lighthouse:  LCP <time>, CLS <value>, FID <time>

ACCEPTANCE CRITERIA
  [x] <each>

NEXT STEPS
  - /code-review  (universal diff audit)
  - /web-audit    (deep frontend audit)
  - /qa           (find + fix bugs)
```

---

## Hard rules

- Never ship without type-checking.
- Never store auth tokens in localStorage.
- Never put secrets in PUBLIC env vars.
- Never `dangerouslySetInnerHTML` with user input (sanitize or refuse).
- Always provide loading, error, empty states — no silent blank screens.
- Always write at least one test per new component/hook.


---

## CRITICAL LESSON LEARNED — Build Verification

**Local dev ≠ production build.** `npm run dev` is forgiving; `npm run build` catches SSR mismatches, strict-mode double-renders, missing envs, and type errors that dev mode silences. Run `npm run build` before handoff.

## CRITICAL LESSON LEARNED — Acceptance Criteria Verification

Never claim "Done" from code review alone. Open the page in a real browser, click every acceptance criterion, verify in Network tab that the right requests fire.

## CRITICAL LESSON LEARNED — Branch Scope Discipline

When listing what was shipped, filter by the specific branch the user asked about. Never mix feature branches into a develop-branch report.
