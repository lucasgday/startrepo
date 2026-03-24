Bootstrap a new project from scratch using proven patterns. Do not copy files from other repos — recreate everything from the guide below.

## Step 0 — Gather inputs

Ask the user (all at once, one message):
1. **Project name** (e.g. "My App") and **slug** (e.g. `my-app`, used for dir and GitHub repo name)
2. **One-line description** — what does it do?
3. **Which modules to include?** (user can say "all" or list them)
   - `auth` — Supabase auth, login/signup/reset, Turnstile captcha
   - `email` — Resend, batch sending, email_log, cron_log, unsubscribe
   - `dev-tools` — /dev/metrics, /dev/backlog, /mailing
   - `llm` — OpenAI or Anthropic streaming pattern
   - `landing` — public landing page with hero + features block
   - `observability` — Sentry error tracking + SWR error capture
4. **GitHub visibility**: public or private?
5. **Package manager**: pnpm (default) or npm?

Wait for answers before proceeding.

---

## Step 1 — Scaffold

```bash
npx create-next-app@latest <slug> \
  --typescript --tailwind --eslint --app --src-dir \
  --import-alias "@/*" --no-turbopack
cd <slug>
```

Then install base dependencies:
```bash
pnpm add @supabase/supabase-js @supabase/ssr zod
pnpm add -D @types/node
```

Add per selected module:
- `auth`: no extra deps (Supabase already added)
- `email`: `pnpm add resend`
- `llm` (OpenAI): `pnpm add openai`
- `llm` (Anthropic): `pnpm add @anthropic-ai/sdk`
- Turnstile (if auth): no npm package needed — plain fetch to Cloudflare API

---

## Step 2 — Git + GitHub

```bash
git init
git add -A
git commit -m "chore: initial scaffold"
gh repo create <slug> --<public|private> --source=. --remote=origin --push
```

---

## Step 3 — Core structure (always)

### CSS architecture
Create `src/styles/globals.css` with CSS variables. Use CSS Modules (`.module.css`) colocated with components — **not** a single global file.

Core tokens to define:
```css
:root {
  --bg: #f8fafc;
  --card: #ffffff;
  --ink: #0f172a;
  --muted: #64748b;
  --line: #e2e8f0;
  --accent: /* project accent color, ask user */;
  --accent-soft: /* light tint of accent */;
  --danger: #dc2626;
  --danger-soft: #fef2f2;
  --radius-sm: 0.5rem;
  --radius-md: 0.75rem;
  --radius-lg: 1rem;
}
```

Sidebar (if applicable):
```css
:root {
  --sb-bg: #1a2332;
  --sb-accent: #34d399;
  --sb-width: 200px;
}
```

### Layout shell
Two variants:
- **Authenticated**: `app-layout` (flex row) + `app-sidebar` (200px, `var(--sb-bg)`) + `app-content` (flex 1, overflow-y scroll)
- **Public**: `app-header` (top bar) + `container app-main`

Create `src/components/app-sidebar.tsx` as a client component with nav links. Use `usePathname` for active state. Gate dev links by role check (not `NODE_ENV`).

### Middleware (`src/middleware.ts`)
Public paths: `/`, `/auth/*`, `/privacy`.
All other paths require session. Redirect to `/auth/login?next=<path>` if unauthenticated.
Inject `x-pathname` header for server components to read current path.

### `src/lib/db/supabase-repository.ts`
Single file for all DB queries. Export typed functions. Never write inline Supabase queries in components or route handlers.

### `src/lib/auth/session.ts`
Export:
- `requireSessionUser()` — throws redirect to login if no session
- `getSessionUserOrNull()` — returns user or null, no redirect

### `src/lib/auth/feature-access.ts`
Export:
- `isDevOrAdminAccess(user)` — returns true for admin/dev emails. Use for gating dev tools and beta features. Never use `NODE_ENV` for gating.

### Icon button tooltips
For any icon-only button (no visible text), use `data-tooltip="label"` + CSS pseudo-elements instead of the browser-native `title` attribute (which can't be styled and has a delay):
```css
[data-tooltip] { position: relative; }
[data-tooltip]::after {
  content: attr(data-tooltip);
  position: absolute; bottom: calc(100% + 6px); left: 50%;
  transform: translateX(-50%);
  background: #1e293b; color: #f8fafc;
  font-size: 0.7rem; white-space: nowrap;
  padding: 0.22rem 0.5rem; border-radius: 0.3rem;
  pointer-events: none; opacity: 0; transition: opacity 0.12s; z-index: 9999;
}
[data-tooltip]:hover::after { opacity: 1; }
@media (pointer: coarse) { [data-tooltip]::after { display: none; } }
```
Always keep `aria-label` separately — `data-tooltip` is visual only.

### SEO baseline (always — do this from day 1)

Add `src/app/sitemap.ts` and `src/app/robots.ts` before the first deploy. Next.js App Router serves them automatically at `/sitemap.xml` and `/robots.txt`.

**`src/app/sitemap.ts`** — list all public routes. For dynamic routes (e.g. `/blog/[slug]`), call the data function directly:
```typescript
import type { MetadataRoute } from "next";

const BASE_URL = "https://myapp.com";

export default function sitemap(): MetadataRoute.Sitemap {
  return [
    { url: BASE_URL, changeFrequency: "weekly", priority: 1.0 },
    { url: `${BASE_URL}/about`, changeFrequency: "monthly", priority: 0.7 },
    // dynamic: ...getSlugs().map(slug => ({ url: `${BASE_URL}/blog/${slug}`, ... }))
  ];
}
```

**`src/app/robots.ts`** — block auth, dev, and app routes. Allow only public pages:
```typescript
import type { MetadataRoute } from "next";

export default function robots(): MetadataRoute.Robots {
  return {
    rules: [{
      userAgent: "*",
      allow: ["/", "/about", "/privacy"],
      disallow: ["/api/", "/dev/", "/auth/", "/app/"],
    }],
    sitemap: "https://myapp.com/sitemap.xml",
  };
}
```

**Canonicals** — add `alternates.canonical` to every public page's metadata export:
```typescript
export const metadata: Metadata = {
  title: "...",
  alternates: { canonical: "https://myapp.com/about" },
  openGraph: { ... },
};
```

**Vercel domain setup** — always set the apex domain as primary (not www):
1. In Vercel → Settings → Domains, add both `myapp.com` and `www.myapp.com`
2. Set `myapp.com` as primary ("Production"). Let `www.myapp.com` redirect 307 to it.
3. Hardcode apex everywhere in code (`metadataBase`, `sitemap`, `robots`, email templates, `APP_BASE_URL`). Never hardcode www.
4. Verify: `curl -sI https://myapp.com/sitemap.xml` should return 200, not 307.

**Search Console** — verify via DNS (domain property covers all variants):
1. Add a domain property (`myapp.com`, not URL prefix `https://www.myapp.com/`) in Search Console
2. Verify via DNS provider — no code change needed
3. After first production deploy, submit sitemap: `myapp.com/sitemap.xml`
4. If sitemap shows error "No se ha podido obtener": check that the Vercel primary domain matches the sitemap URL (www mismatch is the most common cause)

**Search Console verification** — support via optional env var in `layout.tsx`:
```typescript
export const metadata: Metadata = {
  metadataBase: new URL("https://myapp.com"),
  ...(process.env.NEXT_PUBLIC_GOOGLE_SITE_VERIFICATION
    ? { verification: { google: process.env.NEXT_PUBLIC_GOOGLE_SITE_VERIFICATION } }
    : {}),
};
```
Set `NEXT_PUBLIC_GOOGLE_SITE_VERIFICATION` in Vercel env vars when verifying Search Console. If unset, renders nothing.

**Schema.org** — add `WebSite` + `Organization` JSON-LD on the homepage:
```tsx
const websiteSchema = {
  "@context": "https://schema.org",
  "@type": "WebSite",
  name: "My App",
  url: "https://myapp.com",
  description: "...",
};

// In the page component:
<script type="application/ld+json"
  dangerouslySetInnerHTML={{ __html: JSON.stringify(websiteSchema) }}
/>
```
Define schema objects as module-level consts (not inside components) to avoid re-creation on every render.

### Security baseline (always — do this from day 1)

This is the difference between a vibe-coded app and a production app. Add these before writing any feature code.

**1. Security headers (`next.config.ts`)**
```typescript
const securityHeaders = [
  { key: "X-Frame-Options", value: "DENY" },
  { key: "X-Content-Type-Options", value: "nosniff" },
  { key: "Referrer-Policy", value: "strict-origin-when-cross-origin" },
  { key: "Permissions-Policy", value: "camera=(), microphone=(), geolocation=()" },
];

// In nextConfig:
async headers() {
  return [{ source: "/(.*)", headers: securityHeaders }];
}
```

**2. Cron endpoints — fail-closed when secret is missing**
```typescript
// WRONG — if CRON_SECRET is unset, "Bearer " === "Bearer " passes
const expected = `Bearer ${process.env.CRON_SECRET ?? ""}`;

// CORRECT — fail-closed if env var is absent
if (!process.env.CRON_SECRET) {
  return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
}
const expected = `Bearer ${process.env.CRON_SECRET}`;
if (!crypto.timingSafeEqual(Buffer.from(authHeader), Buffer.from(expected))) {
  return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
}
```
Same pattern for any webhook endpoint that uses a shared secret.

**3. Never trust client-supplied `user_id`**
```typescript
// WRONG — attacker sets userId in request body
const { userId, habitId } = await req.json();

// CORRECT — always read user_id from the verified session
const user = await requireSessionUser();
const habitId = (await req.json()).habitId; // only take non-identity fields from body
```
Rule: only `userId` comes from session. Everything else can come from the request.

**4. Service role client — server-only**
```typescript
// Service role bypasses RLS. Only use in:
// - Cron endpoints (no user session)
// - Admin-only endpoints
// - DB migrations

// User-facing endpoints: always use the user-scoped Supabase client
// getSupabaseServerClient() — reads session cookies, respects RLS
// NEVER: getSupabaseServiceRoleClient() in a user-facing endpoint
```

**5. RLS on every table**
Enable Row Level Security on every table in Supabase. Add at minimum:
```sql
ALTER TABLE my_table ENABLE ROW LEVEL SECURITY;
CREATE POLICY "users own their rows" ON my_table
  USING (user_id = auth.uid());
```
RLS is the last line of defense if application code has a bug.

**6. UTC date arithmetic in API routes**
```typescript
// WRONG — setMonth() uses local time, overflows in negative-offset timezones
// e.g. "2026-02-01T00:00:00Z" in UTC-3 is Jan 31 local → setMonth(Feb) → Feb 31 → Mar 3
const d = new Date("2026-02-01T00:00:00Z");
d.setMonth(d.getMonth() + 1); // BUG: gives Mar 3-4 depending on timezone

// CORRECT — always use UTC methods for date arithmetic on the server
d.setUTCMonth(d.getUTCMonth() + 1); // gives Mar 1 ✓
```
Rule: any Date arithmetic in API routes must use `setUTCMonth`/`getUTCMonth`/`setUTCDate`/etc.

### Integration test safety (if Supabase)
When tests create/delete users in a real Supabase instance:
- **Env guard**: `assertNotProduction()` — block tests against non-local Supabase unless `SUPABASE_INTEGRATION_TESTS_ENABLED=true`
- **Distinctive emails**: use prefix `integration-test--` + domain `@test.integration.invalid` — impossible for real users to match
- **Delete guard**: before `admin.deleteUser(id)`, verify the user's email starts with the test prefix
- **Scoped cleanup**: `cleanupOrphanedIntegrationUsers(client, labelPrefix)` — each test suite cleans only its own label prefix, not all test users (avoids race conditions in parallel test files)

---

## Step 4 — Module: `auth`

### Pages to create
- `src/app/auth/login/page.tsx`
- `src/app/auth/signup/page.tsx`
- `src/app/auth/forgot-password/page.tsx`
- `src/app/auth/reset-password/page.tsx`
- `src/app/auth/callback/route.ts` — PKCE exchange (`exchangeCodeForSession`)
- `src/app/auth/verify/page.tsx` — post-signup "check your email" page

### Patterns
- All forms: client components with controlled inputs + Zod validation before submit
- Password fields: show/hide toggle (`.password-input-wrapper` pattern)
- `?next=` redirect after login: always sanitize with `sanitizeNextPath()` — strip external URLs, allow only relative paths starting with `/`
- Turnstile widget: render client-side, send token to server, validate with `https://challenges.cloudflare.com/turnstile/v0/siteverify`
- Auth errors: use an enum/union of error codes (`verify_email`, `link_expired`, `invalid_credentials`, etc.). Map to user-facing copy. Show "Resend verification" where applicable.

### API routes
- `POST /api/auth/login`
- `POST /api/auth/signup`
- `POST /api/auth/logout`
- `PATCH /api/auth/password` — change password (requires current password)
- `PATCH /api/auth/password/recovery` — reset password via recovery token (no current password needed)

### Supabase setup
- Custom sending domain (not `supabase.co`). Configure DKIM + SPF + DMARC.
- Customize "Confirm signup" and "Reset password" email templates.
- Set Site URL + Redirect URLs in Supabase dashboard.

---

## Step 5 — Module: `landing`

### Structure
```
src/app/page.tsx           ← server component, renders hero
src/app/landing.tsx        ← client component with full page
src/styles/landing.css     ← landing-specific styles
```

### Sections
1. **Nav** — logo + app name + CTA button (Sign up)
2. **Hero** — H1 headline + tagline + sub-tagline + primary CTA
3. **How it works** — 3-step visual or feature grid
4. **Features block** — cards with icon + title + description
5. **Footer** — links: Privacy · Terms · Contact

### Patterns
- Authenticated users redirected to `/dashboard` or app home at page load (server-side check)
- Hero supports A/B variants — use a `?v=` query param or random assignment stored in cookie
- No fake testimonials or social proof until you have real data (LND-006 lesson)

---

## Step 6 — Module: `email`

### Files to create
- `src/lib/email/resend.ts` — singleton Resend client, `getResendClient()`
- `src/lib/email/marketing.ts` — `sendMarketingEmail()` + `sendMarketingEmailBatch()`
- `src/lib/emailTokens.ts` — HMAC-SHA256 token sign/verify

### Key patterns

**Batch sending** (mandatory for crons — never loop individual sends):
```typescript
// Collect all pending emails → single resend.batch.send() call
// Chunks of 100 max (Resend limit)
// Always check { error } on response — SDK doesn't throw
```

**Email headers** (all marketing emails):
```typescript
headers: {
  "List-Unsubscribe": `<${unsubUrl}>`,
  "List-Unsubscribe-Post": "List-Unsubscribe=One-Click",
}
```

**ALLOWED_TEST_EMAIL guard**:
```typescript
// If env var is set, skip send for any recipient that doesn't match.
// Must be DELETED (not just empty) in production.
```

**Unsubscribe endpoint** (`src/app/unsubscribe/route.ts`):
- `GET` — verify token, update DB, redirect to confirmation page
- `POST` — one-click RFC 8058, return `{ ok: true }`
- Scopes: `marketing_unsub`, `daily_unsub` (keep separate for granular opt-out)

**Token structure** (HMAC-SHA256):
```
base64url(payload) + "." + base64url(HMAC-SHA256(payloadB64, EMAIL_LINK_SECRET))
payload = { user_id, scope, exp }
```
Verify with `crypto.timingSafeEqual`.

### DB tables (Supabase migrations)
```sql
-- email_log: audit trail of every email sent
create table email_log (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references auth.users,
  type text not null,  -- 'activation_e1', 'daily_checkin', etc.
  to_email text,
  subject text,
  sent_at timestamptz default now()
);

-- cron_log: audit trail of every cron run
create table cron_log (
  id uuid primary key default gen_random_uuid(),
  cron_name text not null,
  sent int default 0,
  skipped int default 0,
  errors int default 0,
  details jsonb,
  ran_at timestamptz default now()
);

**Cron catchup (Vercel Hobby):** Vercel Hobby can miss cron jobs during deploys. Add a `/api/health` endpoint pinged every 5min by an external service (cron-job.org). The health endpoint checks `cron_log` — if a cron didn't run today and it's 1h past its scheduled UTC hour, trigger it via internal fetch with `CRON_SECRET`. Use `hasCronRunToday(client, cronName)` helper in the repository.

-- user_email_prefs: opt-out preferences
create table user_email_prefs (
  user_id uuid primary key references auth.users,
  marketing_enabled boolean default true,
  daily_enabled boolean default true,
  updated_at timestamptz default now()
);
```

Add `logEmailSent()` and `logCronRun()` to `supabase-repository.ts`.

### Cron pattern (two-phase)
```
Phase 1: loop users → collect PendingEmail[] (no sends yet)
Phase 2: sendMarketingEmailBatch(pending) → then update DB per user
```
Never update DB before send succeeds. Never send inside the collection loop.

---

## Step 7 — Module: `llm`

### OpenAI streaming (NDJSON)

```typescript
// src/lib/llm/openai-stream.ts
// Uses Responses API with stream: true, stream_options: { include_usage: true }
// AsyncGenerator that yields parsed NDJSON lines starting with "{"
// Returns { usage, model, latencyMs } on completion
```

SSE endpoint pattern (`/api/stream/route.ts`):
```typescript
export const runtime = "nodejs";
export const maxDuration = 120;

// Auth + quota check first
// Emit type:"item" per result, type:"done" at end, type:"error" on failure
// ReadableStream with encoder — never use edge runtime for this
```

Prompt design:
- Return compact NDJSON (one object per line) — reduces output tokens ~80% vs JSON blobs
- Include `idx` field to map back to input lines
- Don't return fields the server can reconstruct

### Anthropic streaming

```typescript
// src/lib/llm/anthropic-stream.ts
// Uses claude-sonnet-4-6 (or latest) with stream: true
// Same SSE pattern as above
// Default to latest Sonnet; override with env var ANTHROPIC_MODEL
```

### Env vars
- OpenAI: `OPENAI_API_KEY`, `OPENAI_MODEL` (e.g. `gpt-4o-mini`)
- Anthropic: `ANTHROPIC_API_KEY`, `ANTHROPIC_MODEL` (e.g. `claude-sonnet-4-6`)

---

## Step 7b — Module: `observability`

Install: `pnpm add @sentry/nextjs`

### Setup files

- `src/lib/observability/sentry.ts` — pure helper functions (no Sentry import): `getSentryDsn`, `getSentryEnvironment`, `getSentryRelease`, `getSentryTracesSampleRate`, `getSentryReplaySessionSampleRate`, `getSentryReplayOnErrorSampleRate`, `getSentryInitOptions`, `shouldUploadSentrySourceMaps`, `shouldEnableSentryTunnel`. Use separate `Client` variants that read `process.env` directly (for tree-shaking in client bundles) vs server variants that accept `env` param.
- `src/lib/observability/swr.ts` — `describeSwrKey(key)` for Sentry context, `shouldCaptureSwrError(error)` to filter AuthError/AbortError from Sentry.
- `src/components/observability-provider.tsx` — client component wrapping `<SWRConfig>` with `onError` that captures to Sentry. Also sets `Sentry.setUser({ id })` on mount.
- `src/instrumentation.ts` — `Sentry.init(getSentryInitOptions("server"))` for server runtime
- `src/instrumentation-client.ts` — `Sentry.init(...)` with replaysSessionSampleRate + replaysOnErrorSampleRate + Sentry.replayIntegration()
- `src/sentry.server.config.ts` + `src/sentry.edge.config.ts` — thin init files
- `src/app/global-error.tsx` — Next.js global error boundary with `Sentry.captureException`

### next.config.ts

Wrap with `withSentryConfig()`:
- `tunnelRoute: "/monitoring"` — avoids ad blockers
- `disableLogger: true`
- `sourcemaps.deleteSourcemapsAfterUpload: true`
- `silent: !shouldUploadSentrySourceMaps()` — skip upload when no auth token

### Layout integration

Wrap app children with `<ObservabilityProvider user={user}>` in root layout (server component passes user from session).

### Env vars

```
NEXT_PUBLIC_SENTRY_DSN=            # from sentry.io project
SENTRY_AUTH_TOKEN=                 # for source map upload (build only)
SENTRY_ORG=                        # sentry org slug
SENTRY_PROJECT=                    # sentry project slug
SENTRY_ENVIRONMENT=                # optional override (defaults to VERCEL_ENV)
NEXT_PUBLIC_SENTRY_TRACES_SAMPLE_RATE=0.1
NEXT_PUBLIC_SENTRY_REPLAYS_SESSION_SAMPLE_RATE=0.1
NEXT_PUBLIC_SENTRY_REPLAYS_ON_ERROR_SAMPLE_RATE=1.0
```

### Middleware

Add `/monitoring` to public paths (Sentry tunnel).

---

## Step 8 — Module: `dev-tools`

All dev routes gated by `isDevOrAdminAccess()`. Accessible to admin in production.

### `/dev/metrics`
- Stat cards: total users, active users (30d), key entity counts
- Bar chart (Recharts): signups over time
- Table: per-user breakdown
- Email funnels section (if `email` module included): sent/clicked per email type
- Cron log table (if `email` module included): last 20 runs
- **Always pass `?date=YYYY-MM-DD` (client local date) to the metrics API** — the server uses it for "today" calculations to avoid timezone mismatch

### `/dev/backlog`
- Reads `BACKLOG.md` bundled at build time (GET)
- PUT only in development (not production)
- Parser: markdown → `BacklogItem[]` with ID, area, priority, I/F/R/E columns, score = `I*2 + F + R - E`
- Display: solved chart + scoring panel + master table grouped by milestone

### `/mailing`
- Preview all email templates side by side
- Tabs: HTML / plain text
- Width toggle: mobile (375px) / desktop (600px)
- Sticky header for navigation between templates

---

## Step 9 — BACKLOG.md

Create a `BACKLOG.md` at repo root with this structure:

```markdown
# Backlog — <Project Name>

## Scoring
Items scored as `I*2 + F + R - E` (Impact × 2 + Frequency + Reach − Effort). All 1–5.

## Active — M1: Foundation

| ID | Area | Title | Status | Priority | Branch | PR | Notes | Done | I | F | R | E |
|----|------|-------|--------|----------|--------|----|-------|------|---|---|---|---|
| CORE-001 | Core | Initial setup | done | P0 | - | - | Scaffold + auth + deploy | <today> | 5 | 5 | 5 | 1 |

## Backlog

| ID | Area | Title | Status | Priority | Branch | PR | Notes | Done | I | F | R | E |
|----|------|-------|--------|----------|--------|----|-------|------|---|---|---|---|
```

Use sequential IDs by area (e.g. `AUTH-001`, `FEAT-001`, `INFRA-001`).

---

## Step 10 — CLAUDE.md

Create `CLAUDE.md` at repo root:

```markdown
# <Project Name> — Claude instructions

## Language
- Reply to user in [Spanish/English — ask user].
- Code, commits, and comments in English.

## Skills available

| Skill | When to use |
|---|---|
| `/feature <ID>` | Starting a backlog item. Reads context, enters plan mode, implements. |
| `/check` | Before every commit. TypeScript + lint + tests + diff review. |
| `/simplify` | After finishing a feature or refactor. Quality, reuse, efficiency. Not for small fixes. |
| `/security-review [file]` | When touching endpoints, auth flows, or user data logic. |
| `/backend-review [file]` | When adding/changing API routes or DB queries. Auth, limits, error handling. |
| `/design-review [file]` | When creating or modifying UI components. |
| `/dailycheckin` | Start of session. Shows current objective and P0/P1 priorities. |
| `/dailycheckout` | End of session. Updates memory, reviews priorities, confirms next objective. |

## Before closing any task
1. `/check` — tsc + lint + tests. Fix everything that fails.
2. `/security-review` — on modified files. Fix Critical and High.
3. `/backend-review` — on modified API routes. Fix Critical and High.
4. `/design-review` — on modified UI files. Fix Blockers.
5. Mark item as `done` in BACKLOG.md and commit + push.

## CSS
- CSS variables for all design tokens. Use CSS Modules colocated with components.
- Core variables: `--accent`, `--card`, `--muted`, etc. Never hardcode colors.

## API / DB
- All Supabase queries in `src/lib/db/supabase-repository.ts`.
- REST endpoints in `src/app/api/`.
- Runtime: `nodejs` (not `edge`) in all route handlers.
- Always use explicit `.limit()` on Supabase queries that may return >100 rows. PostgREST silently truncates at 1000 rows by default — no error, just missing data.

## Backlog
- Lives in `BACKLOG.md` at repo root.
- Sequential IDs by area: FEAT-XXX, AUTH-XXX, INFRA-XXX, etc.
- Status: `next` → `in_progress` → `done`.

## Commits
- Format: `feat|fix|docs|refactor(scope): concise message`
- Always add: `Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>`
- When closing a backlog item: commit + push automatically, without waiting for the user to ask.

## Global best practices
- `~/.claude/BEST_PRACTICES.md` contains cross-repo patterns. Read at session start when relevant.
```

---

## Step 11 — Environment variables

After scaffolding, create `.env.local` with all required vars (leave values blank for the user to fill):

### Always required
```env
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=
APP_BASE_URL=http://localhost:3000
```

### If `auth` module
```env
NEXT_PUBLIC_TURNSTILE_SITE_KEY=
TURNSTILE_SECRET_KEY=
```

### If `email` module
```env
RESEND_API_KEY=
EMAIL_FROM=no-reply@mail.yourdomain.com
EMAIL_LINK_SECRET=   # random 32+ char secret for HMAC tokens
ALLOWED_TEST_EMAIL=  # your email — DELETE this var before sending to real users
CRON_SECRET=         # random secret to protect cron endpoints
```

### If `llm` (OpenAI)
```env
OPENAI_API_KEY=
OPENAI_MODEL=gpt-4o-mini
```

### If `llm` (Anthropic)
```env
ANTHROPIC_API_KEY=
ANTHROPIC_MODEL=claude-sonnet-4-6
```

Print the full list of vars needed based on selected modules, and remind the user to:
1. Add them to Vercel (Settings → Environment Variables)
2. Never commit `.env.local` (verify it's in `.gitignore`)
3. Delete `ALLOWED_TEST_EMAIL` before rolling out emails to real users

---

## Step 11b — Playwright E2E auth (if auth module)

When adding Playwright e2e tests to a project with Supabase auth + captcha, use this pattern to bypass captcha in tests:

**`/api/auth/test-exchange/route.ts`** (dev-only endpoint):
```typescript
// Returns 404 in production — only active in development
if (process.env.NODE_ENV !== "development") {
  return NextResponse.json({ error: "Not found" }, { status: 404 });
}
// Accepts a Supabase admin-generated magic link token, exchanges for session, sets SSR cookies
```

**`e2e/global-setup.ts`**: generate token via Supabase admin API (`auth.admin.generateLink`), POST to `/api/auth/test-exchange`, save `storageState` to `e2e/.auth/user.json` (cache 50min TTL).

**`e2e/fixtures/auth.ts`**: export `authenticatedPage` fixture that restores storageState.

**`playwright.config.ts`**: set `globalSetup`, `storageState` in projects, and add `playwright.config.ts`, `e2e/**`, `scripts/**` to `tsconfig.json` excludes to prevent Next.js from compiling them (avoids build-time dep pollution like `dotenv`).

---

## Step 12 — Initial commit and push

```bash
git add -A
git commit -m "feat: initial boilerplate — <selected modules>

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>"
git push -u origin main
```

Then print a summary:
- Repo URL on GitHub
- Modules included
- Next steps: fill env vars → deploy to Vercel → configure Supabase auth redirect URLs
