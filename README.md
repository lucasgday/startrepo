# startrepo

Two Claude Code skills for Next.js projects: one to bootstrap, one to audit.

Most projects accumulate invisible gaps — a cron that passes when the secret is empty, a query without `.limit()`, a sitemap that blocks the wrong paths. `reviewrepo` surfaces those. `startrepo` avoids them from day one.

---

- **`startrepo.md`** — scaffold a new project with production patterns built in
- **`reviewrepo.md`** — audit an existing project against ~40 production patterns

## How to install

### 1. Install Claude Code

```bash
npm install -g @anthropic-ai/claude-code
```

### 2. Add the skills

```bash
mkdir -p ~/.claude/skills
cp startrepo.md ~/.claude/skills/startrepo.md
cp reviewrepo.md ~/.claude/skills/reviewrepo.md
```

---

## startrepo — new projects

Open Claude Code in the directory where you want to create the project:

```bash
cd ~/code
claude
/startrepo
```

Claude asks for project name, modules, and GitHub visibility — then scaffolds everything automatically.

### What gets created

```
my-app/
├── src/
│   ├── app/
│   │   ├── api/          # REST endpoints
│   │   ├── auth/         # Login, signup, reset, callback
│   │   ├── dev/          # Dev tools (gated)
│   │   └── sitemap.ts    # Dynamic sitemap
│   ├── components/       # Shared UI components
│   ├── lib/
│   │   ├── auth/         # Session, feature access, rate limiting
│   │   ├── db/           # supabase-repository.ts (all queries)
│   │   ├── email/        # Resend client + batch sending
│   │   └── llm/          # OpenAI / Anthropic streaming
│   └── styles/           # CSS variables + component styles
├── e2e/                  # Playwright tests with auth
├── BACKLOG.md            # Structured backlog with scoring
├── CLAUDE.md             # Claude project instructions
└── .env.local            # Env vars template (never committed)
```

### Modules (all opt-in)

- TypeScript, Tailwind CSS, custom CSS architecture
- Supabase auth (login, signup, password reset, Turnstile captcha)
- Email via Resend (batch sending, unsubscribe, audit logs)
- LLM streaming (OpenAI or Anthropic)
- Sentry observability
- Dev tools (`/dev/metrics`, `/dev/backlog`, `/mailing`)
- SEO baseline (sitemap, robots.txt, canonicals, schema.org)
- Security baseline (headers, cron protection, RLS, CSRF)
- Playwright E2E auth (captcha bypass for tests)

---

## reviewrepo — existing projects

Open Claude Code inside any existing Next.js project:

```bash
cd ~/my-project
claude
/reviewrepo
```

Claude reads the codebase and checks it against ~40 patterns across 11 areas: CSS architecture, layout shell, middleware, database, auth, security, SEO, email, LLM streaming, observability, and TypeScript/build.

### What the output looks like

```
## reviewrepo — my-project

### Summary
38 items checked · 24 ✅ · 8 ⚠️ · 6 ❌

### Critical (fix before next deploy)
- ❌ CRON_SECRET fail-closed: crons authorize when env var is empty
  Fix: add `if (!process.env.CRON_SECRET) return NextResponse.json({}, { status: 401 })` at top of each cron handler
- ❌ Service role client used in user-facing endpoint: src/app/api/items/route.ts:12
  Fix: replace with user-scoped Supabase client so RLS applies

### High (fix this week)
- ⚠️ No explicit .limit() on src/lib/db/supabase-repository.ts:87 — PostgREST silently truncates at 1000 rows
  Fix: add .limit(500) or paginate

### Medium (worth doing)
- ⚠️ Canonical missing on /about and /pricing pages
...
```

After the report, Claude offers to fix issues one by one — Critical first, or whatever you choose.

### What it checks

| Area | Items |
|------|-------|
| Security | Headers, CRON_SECRET fail-closed, timing-safe comparisons, CSRF, RLS |
| Database | Single repository file, `.limit()`, UTC dates, service role scope |
| Auth | Session helpers, password recovery flow, `?next=` sanitization |
| SEO | sitemap, robots.txt, canonicals, schema.org, apex domain |
| Email | Batch sending, unsubscribe headers, two-phase crons |
| Middleware | Allowlist vs denylist, open redirect protection, feature gating |
| LLM | Runtime, auth before stream, structured events |
| TypeScript | tsconfig excludes, 0 type errors, nodejs runtime |
| Dev tools | Gating, metrics dashboard |
| Observability | Sentry setup, error filtering |
| Playwright | Auth fixture, cached storage state, mobile tests |

---

## Patterns included in both skills

- All DB queries in a single `supabase-repository.ts` — never inline
- Service role client only in crons/admin — never in user-facing endpoints
- Cron endpoints fail-closed when `CRON_SECRET` is unset
- UTC date arithmetic in all API routes
- Batch email sending (never loop individual sends)
- `isDevOrAdminAccess()` for feature gating — never `NODE_ENV`
- Apex domain as Vercel primary — www redirects to apex

---

## Requirements

- [Claude Code](https://claude.ai/code)
- [Supabase](https://supabase.com) account (for startrepo)
- [Vercel](https://vercel.com) account (for startrepo)
- [GitHub CLI](https://cli.github.com) (`gh`) installed and authenticated
- [pnpm](https://pnpm.io) installed
