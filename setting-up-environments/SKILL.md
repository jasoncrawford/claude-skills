---
name: setting-up-environments
description: Use when starting a new project that needs a database or external service, when adding a test suite that touches shared state, or when production credentials need to be separated from development
---

# Setting Up Environments

## Overview

Every project needs at least three isolated environments: **test**, **dev**, and **production**. All differences between them live in environment variables. Environments are flat — no inheritance, no "dev extends base" — each is a complete, independent configuration.

## When to Use

- Starting a new project with a database or external service
- Adding automated tests that touch a database or API
- Deploying to production for the first time
- Noticing test data appearing in your dev database (too late — but fix it now)

## The Three Environments

| Environment | Purpose | Checked into git? | Where config lives |
|-------------|---------|-------------------|-------------------|
| **test** | Automated tests (CI + local) | Yes | `.env.test` |
| **dev** | Local development | Yes | `.env` |
| **production** | Live users | **Never** | Hosting platform (Vercel, Fly, etc.) |

### Why test and dev are checked in

Test and dev configs contain local/throwaway credentials (local database, test API keys). Checking them in means any developer can clone and run immediately — no setup doc to follow, no secrets to request. These credentials have no security value outside the developer's machine.

### Why production is never checked in

Production credentials grant access to real user data. They belong in your hosting platform's environment variable UI, not in any file on any developer's laptop. If your `.env` contains production database URLs, you have a security problem.

## Core Principles

### 1. Flat, not hierarchical

Each environment file is a complete, standalone configuration. No `.env.base` that others extend. No `dotenv-flow` or inheritance chains.

**Why:** Inheritance creates invisible coupling. When you change a "base" value, you may break an environment you weren't thinking about. Flat files are boring and repetitive — that's the point. You can read any single file and know exactly what that environment will use.

### 2. Isolation through data, not code

Environments run the same code. The only differences are environment variable values. Typically this means:
- Different database schemas, databases, or database instances
- Different API keys (test keys vs. live keys)
- Different service URLs (localhost vs. cloud)

**Never** use `if (process.env.NODE_ENV === 'test')` to change behavior. If test and production need different behavior, that's a feature flag — also controlled by an environment variable.

### 3. Local services for test and dev

Tests must hit local services — local database, local auth, local storage. No network calls to cloud services. Tests that depend on a remote database are slow, flaky, and break when the service has an outage or someone pauses the free tier.

Dev should also be local whenever practical. Most databases and queues have a local option (Postgres, Redis, Supabase CLI, LocalStack for AWS). Running locally means you can develop offline, you're not sharing state with other developers, and you're not paying for cloud resources during development.

**Cloud is for production.** If your dev environment points at a cloud database, every developer shares one instance — migrations collide, test data leaks between people, and an outage blocks the whole team. If your test environment points at a cloud database, your CI is one expired credit card away from failing.

When a service genuinely can't run locally (e.g., a third-party API with no emulator), use a dedicated test/sandbox account with throwaway credentials, and accept that those tests are integration tests — slower and less reliable by nature.

### 4. Test isolation is non-negotiable

Tests must never read or write the dev database. Common isolation strategies:

- **Separate schema** (e.g., Postgres `search_path`): Same database instance, different namespace. Cheapest option for local dev.
- **Separate database**: Different database name on the same server.
- **Separate instance**: Entirely different server. Most isolated but highest overhead.

Pick the lightest option that gives you full isolation. For most projects, a separate schema or database on localhost is sufficient.

### 5. Loading the right config

Your test runner loads `.env.test`. Your dev server loads `.env`. Production reads from the hosting platform. Keep the loading mechanism simple and explicit.

```bash
# Test command loads .env.test explicitly
"test": "env-cmd -f .env.test playwright test"

# Dev server loads .env by default (most frameworks do this)
"dev": "vite"

# Production: hosting platform injects env vars — no file needed
```

**Watch out for tools that merge env files.** Some frameworks (e.g., Vercel CLI's `vercel dev`) auto-load `.env` and override `process.env`. If your test runner sets `DATABASE_SCHEMA=test` via `env-cmd` but your API server subprocess loads `.env` which doesn't set `DATABASE_SCHEMA`, the API server falls back to the default — and your test writes hit the dev database. Keep environment-specific variables out of `.env` if they need to differ between test and dev and your tooling merges aggressively.

## Setting Up a New Project

1. **Create `.env`** with local dev credentials (localhost URLs, local DB, dev API keys)
2. **Create `.env.test`** with test credentials (same localhost, but different schema/database + test-specific values)
3. **Add `.env.local` and `.env.production` to `.gitignore`** — these are for local overrides and must never be committed
4. **Configure your hosting platform** with production env vars (never create a `.env.production` file)
5. **Update test scripts** to explicitly load `.env.test` (e.g., `env-cmd -f .env.test`)
6. **Verify isolation** — run tests, then check your dev database. If test data leaked, your isolation is broken.

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Production credentials in `.env` | Move to hosting platform. `.env` is for local dev only. |
| `.env.test` not checked in | Check it in. Test credentials are local/throwaway. |
| Test and dev share a database schema | Add a `DATABASE_SCHEMA` or equivalent env var. Use a separate schema for tests. |
| `if (NODE_ENV === 'test')` in application code | Remove it. Use an environment variable for the behavioral difference. |
| `.env.local` checked into git | Add to `.gitignore`. Local overrides are personal. |
| Environment inheritance (`dotenv-flow`, `.env.base`) | Replace with flat, standalone files. Repetition is fine. |
| Tests pointing at a cloud database | Run the database locally. Cloud = slow, flaky, shared state. |
| Dev pointing at a shared cloud database | Run locally. Use Supabase CLI, Docker Compose, etc. Cloud is for production. |
| Tool auto-loading `.env` clobbers test config | Keep differing variables out of `.env`, or use explicit loading that doesn't merge. |

## Checklist

- [ ] `.env` exists with local dev credentials (checked in)
- [ ] `.env.test` exists with test credentials (checked in)
- [ ] `.env.local` gitignored (for personal overrides)
- [ ] No `.env.production` file exists anywhere
- [ ] Production env vars set in hosting platform
- [ ] Test scripts explicitly load `.env.test`
- [ ] Test and dev databases/schemas are isolated
- [ ] Test and dev services run locally (no cloud dependencies)
- [ ] No `if (NODE_ENV === ...)` branching in application code
- [ ] Running tests leaves dev database unchanged
