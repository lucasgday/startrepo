Audit an existing Next.js project against the patterns in startrepo.md. Do NOT make changes unless the user confirms. Be direct and specific — no filler.

## Step 0 — Read the project

Run these in parallel to understand the codebase:

```bash
find src -type f | sort
cat next.config.ts 2>/dev/null || cat next.config.js 2>/dev/null
cat package.json
cat .env.local 2>/dev/null | grep -v "=" | head -30  # show keys, not values
```

Also read:
- `src/middleware.ts` or `src/proxy.ts`
- `src/app/layout.tsx`
- `src/lib/db/` (first file found)
- `src/app/api/` (list only, no content yet)

Do NOT read all files. Use this to map what exists.

---

## Step 1 — Run the checklist

Go through each area below. For each item, mark:
- ✅ **Done** — implemented correctly
- ⚠️ **Partial** — exists but incomplete or has a known issue
- ❌ **Missing** — not implemented
- `—` **N/A** — not applicable (module not selected)

Read only the files needed to verify each item. Don't guess.

---

### CI / Quality gates
- [ ] `.github/workflows/ci.yml` exists with `pnpm tsc --noEmit`, `pnpm lint`, `pnpm test --run`
- [ ] `pnpm lint` is a hard blocker (not just a warning) — catches `react-hooks/rules-of-hooks`
- [ ] GitHub Secrets configured for Supabase keys

### React hooks
- [ ] No hook (`useEffect`, `useSWR`, `useState`, etc.) placed after an early `return` — would cause hook count mismatch crash
- [ ] Verify with `pnpm lint`: zero `react-hooks/rules-of-hooks` errors

### Error boundaries
- [ ] `src/app/global-error.tsx` exists with `Sentry.captureException` + `error.digest` shown to user
- [ ] Main authenticated routes have a route-level `error.tsx` (renders inside shell, not full-page takeover)

### CSS architecture
- [ ] CSS variables defined (`--bg`, `--card`, `--ink`, `--muted`, `--accent`, etc.)
- [ ] No hardcoded colors in components (spot-check 3 files)
- [ ] Type scale uses vars (`--text-sm`, `--text-base`, etc.) — not hardcoded `font-size`

### Layout shell
- [ ] Authenticated layout: sidebar + content area
- [ ] Public layout: top header + main
- [ ] Mobile breakpoint defined and consistent across shell files
- [ ] Sidebar hidden by default on mobile (no flash on PWA cold start)

### Middleware
- [ ] Public paths explicitly listed (allowlist, not denylist)
- [ ] `?next=` redirect sanitized (`sanitizeNextPath` or equivalent — no open redirect)
- [ ] Gated routes return 403 JSON for API paths, redirect for pages
- [ ] Feature gating uses role check, not `NODE_ENV`

### Database
- [ ] All queries in a single `supabase-repository.ts` (no inline queries in components/routes)
- [ ] Explicit `.limit()` on queries that may return >100 rows
- [ ] UTC date methods (`setUTCMonth`, `getUTCDate`) in API routes — not local time methods
- [ ] Service role client only in crons/admin — not in user-facing endpoints

### Auth
- [ ] `requireSessionUser()` and `getSessionUserOrNull()` in `src/lib/auth/session.ts`
- [ ] Password recovery uses separate endpoint (no `currentPassword` required)
- [ ] `?next=` param sanitized on login redirect
- [ ] User ID always from session, never from request body

### Security baseline
- [ ] Security headers in `next.config.ts` (`X-Frame-Options`, `X-Content-Type-Options`, `Referrer-Policy`, `Permissions-Policy`)
- [ ] Cron endpoints fail-closed when `CRON_SECRET` is unset (check: `if (!process.env.CRON_SECRET) return 401`)
- [ ] Timing-safe comparison for secrets (`crypto.timingSafeEqual`)
- [ ] CSRF protection on mutating endpoints (Origin header check or equivalent)
- [ ] RLS enabled on all Supabase tables (check migration files)
- [ ] No `window.prompt` / `window.confirm` — all UI inline

### SEO baseline
- [ ] `src/app/sitemap.ts` exists and covers all public routes
- [ ] `src/app/robots.ts` exists — blocks auth/api/dev, allows public pages
- [ ] `alternates.canonical` on every public page metadata
- [ ] `metadataBase` set in root layout
- [ ] WebSite + Organization JSON-LD schema on homepage
- [ ] Vercel primary domain is apex (not www) — check if code hardcodes www anywhere

### Email (if Resend is in package.json)
- [ ] Batch sending for crons (`resend.batch.send`) — no per-user loops
- [ ] `List-Unsubscribe` + `List-Unsubscribe-Post` headers on all marketing emails
- [ ] `ALLOWED_TEST_EMAIL` guard in sending functions
- [ ] Unsubscribe endpoint handles GET + POST (RFC 8058)
- [ ] `email_log` and `cron_log` tables exist
- [ ] Cron catchup mechanism (health endpoint or equivalent)
- [ ] Two-phase cron: collect first, send after — never update DB before send

### LLM streaming (if OpenAI/Anthropic in package.json)
- [ ] SSE endpoint uses `nodejs` runtime, not `edge`
- [ ] `maxDuration` set (60–120s)
- [ ] Auth + quota check before streaming starts
- [ ] Emits `type:"done"` and `type:"error"` — not just raw text

### Observability (if Sentry in package.json)
- [ ] `src/instrumentation.ts` and `src/instrumentation-client.ts` exist
- [ ] SWR errors captured (with AuthError/AbortError filtered out)
- [ ] Sentry tunnel route (`/monitoring`) in public paths
- [ ] Source maps uploaded on build (not committed to repo)

### Dev tools
- [ ] All `/dev/*` routes gated by `isDevOrAdminAccess()` (not `NODE_ENV`)
- [ ] `/dev/backlog` or equivalent exists
- [ ] Metrics dashboard exists with at minimum: signups, active users, key entity counts

### TypeScript / Build
- [ ] `tsconfig.json` excludes `playwright.config.ts`, `e2e/**`, `scripts/**`
- [ ] `pnpm tsc --noEmit` passes with 0 errors
- [ ] Runtime explicitly set to `nodejs` in all route handlers (no `edge`)

### Playwright E2E (if e2e/ dir exists)
- [ ] Auth fixture bypasses captcha via admin token exchange
- [ ] `storageState` cached (not re-authenticated on every test run)
- [ ] Mobile viewport tests for key user flows

---

## Step 2 — Report

Output in this format:

```
## reviewrepo — [project name]

### Summary
X items checked · Y ✅ · Z ⚠️ · W ❌

### Critical (fix before next deploy)
- ❌ [item]: [what's wrong + file:line if known]
  Fix: [1-line suggestion]

### High (fix this week)
- ⚠️ [item]: [what's wrong]
  Fix: [1-line suggestion]

### Medium (worth doing)
- [items]

### Low / Nice to have
- [items]

### N/A (modules not present)
- [items skipped]
```

Keep each finding to 2 lines max: what's wrong + how to fix it.

---

## Step 3 — Offer to fix

After the report, ask:

> "¿Querés que arregle alguno de estos? Puedo empezar por los Críticos o decime cuál."

Wait for the user to choose. Do NOT start fixing without confirmation.
If the user says "fix all Critical", fix them one by one, running `/check` after each.
