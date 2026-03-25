---
name: infrastructure-as-code
description: Use when about to perform infrastructure setup manually — running SQL in a web UI, clicking through a dashboard to configure a service, or telling the user to do something by hand that could instead be expressed as a file committed to the repo
---

# Infrastructure as Code

## Overview

All infrastructure configuration belongs in files checked into the repository. If you can click it in a web UI, you should also be able to reproduce it by running a command against the repo. Manual steps are a liability: they're undocumented, unrepeatable, and invisible to code review.

## The Rule

**If it configures infrastructure, it lives in the repo.**

This means:
- Database schema changes → migration files (`supabase/migrations/`, `db/migrate/`, etc.)
- Service configuration → config files (`railway.json`, `vercel.json`, `fly.toml`, etc.)
- CI/CD pipelines → workflow files (`.github/workflows/`)
- Secrets and credentials → **exception**: values stay out of the repo, but the *names* and *structure* are documented (`.env.example`, deployment runbook in README)

## Common Violations

| Temptation | Correct approach |
|------------|-----------------|
| "Run this SQL in the Supabase dashboard" | Add a migration file; run `supabase db push` |
| "Go to Railway and set these env vars" | Document required vars in `.env.example`; set values via platform UI (values are secret, but names aren't) |
| "Enable RLS in the Supabase table editor" | Add `alter table ... enable row level security;` to the migration |
| "Create the table in the web UI" | Write a migration file; apply with CLI |
| "Click 'Add webhook' in GitHub settings" | Automate with `gh api` or document exactly in a runbook |

## Migration Files

Every schema change is a new migration file, never an edit to an existing one:

```
supabase/migrations/
  20260321000000_create_logging_tables.sql   ← initial schema
  20260401120000_add_indexes.sql             ← new change
```

Apply with: `supabase db push` (or equivalent for your stack).

Auto-deploy on merge: add a CI workflow that runs `supabase db push` on push to `main` when migration files change.

## Service Config Files

Most platforms support a config file that Railway/Vercel/Fly picks up automatically:

- **Railway:** `railway.json`
- **Vercel:** `vercel.json`
- **Fly.io:** `fly.toml`
- **Render:** `render.yaml`

Commit these files. They define build commands, start commands, health checks, and restart policies — all things you'd otherwise click through a dashboard to set.

## Secrets Are the Exception

Secret *values* (API keys, tokens, passwords) must not be committed. But:
- Document which secrets are needed in `.env.example`
- Set values through the platform's secrets UI or CLI
- Treat the secret *names* as part of the infrastructure definition

## Red Flags

Stop and find the file-based equivalent if you're about to:
- Tell the user to paste SQL into a web UI
- Suggest clicking through a dashboard to configure something
- Write setup steps that can't be replayed by cloning the repo and running a command
- Create infrastructure that only exists in a platform's UI with no corresponding file

## Checklist

- [ ] Database migrations in versioned files, not applied manually
- [ ] Migrations auto-applied in CI on merge to main
- [ ] Service config (build, start, health check) in a committed config file
- [ ] CI/CD pipelines in `.github/workflows/` (or equivalent)
- [ ] `.env.example` documents all required env var names
- [ ] A fresh clone + one setup command produces a working environment
