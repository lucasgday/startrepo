# startrepo

Two Claude Code skills for Next.js projects:

- **`startrepo.md`** — bootstrap a new project from scratch
- **`reviewrepo.md`** — audit an existing project against production patterns

## What it does

`startrepo.md` is a prompt skill for [Claude Code](https://claude.ai/code) that scaffolds a full Next.js App Router project with:

- TypeScript, Tailwind CSS, custom CSS architecture
- Supabase auth (login, signup, password reset, Turnstile captcha)
- Email via Resend (batch sending, unsubscribe, email/cron audit logs)
- LLM streaming (OpenAI or Anthropic)
- Sentry observability
- Dev tools (`/dev/metrics`, `/dev/backlog`, `/mailing`)
- SEO baseline (sitemap, robots.txt, canonicals, schema.org)
- Security baseline (headers, cron protection, RLS, CSRF)
- Playwright E2E auth (captcha bypass for tests)
- `BACKLOG.md` structure + `CLAUDE.md` project instructions

Each module is opt-in. You can include all or just what you need.

## How to use

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

### 3. Use them

**New project** — open Claude Code in the directory where you want to create it:

```bash
cd ~/code
claude
/startrepo
```

Claude asks for project name, modules, and GitHub visibility — then scaffolds everything automatically.

**Existing project** — open Claude Code inside the project:

```bash
cd ~/my-project
claude
/reviewrepo
```

Claude reads the codebase, checks it against ~40 patterns across security, SEO, auth, email, DB, and more, and reports: ✅ done / ⚠️ partial / ❌ missing. Then offers to fix issues one by one.

## What gets created

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

## Patterns included

- All DB queries in a single `supabase-repository.ts` — never inline
- Service role client only in crons/admin — never in user-facing endpoints
- Cron endpoints fail-closed when `CRON_SECRET` is unset
- UTC date arithmetic in all API routes
- Batch email sending (never loop individual sends)
- `isDevOrAdminAccess()` for feature gating — never `NODE_ENV`
- Apex domain as Vercel primary — www redirects to apex

## Requirements

- [Claude Code](https://claude.ai/code)
- [Supabase](https://supabase.com) account
- [Vercel](https://vercel.com) account
- [GitHub CLI](https://cli.github.com) (`gh`) installed and authenticated
- [pnpm](https://pnpm.io) installed
