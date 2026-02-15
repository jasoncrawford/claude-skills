---
name: vercel-supabase-stack
description: Use when starting a new project and choosing a hosting and database stack, when setting up Vercel serverless functions with Supabase, or when debugging issues with Vercel dev, Supabase CLI, or the interaction between them
---

# Vercel + Supabase Stack

## Overview

Vercel for hosting (static frontend + serverless API functions) and Supabase for database (Postgres + realtime subscriptions). Zero infrastructure to manage, generous free tiers, local development with full parity. Good default choice for small-to-medium web apps with a database.

## When to Choose This Stack

**Good fit:**
- SPA or SSR frontend with a handful of API endpoints
- Postgres is the right database (relational data, JSONB, full-text search)
- You want realtime subscriptions (Supabase has this built in)
- You want to go from zero to deployed in an afternoon
- Team of 1-5 people, no dedicated ops

**Poor fit:**
- Long-running server processes (Vercel functions have a 10s/60s timeout)
- WebSocket server you control (Vercel doesn't support persistent connections — Supabase Realtime is the workaround)
- High-volume writes that need connection pooling tuning (Supabase free tier has limits)
- You need a non-Postgres database
- Complex backend with background jobs, queues, cron (you'll outgrow serverless quickly)

**When to look elsewhere:**
- Fly.io or Railway if you need a long-running server
- Cloudflare Workers + D1/Turso if you want edge-first with SQLite
- AWS/GCP if you need full infrastructure control

## How the Pieces Fit Together

```
┌──────────────────────────────────────────────┐
│  Vercel                                      │
│  ┌────────────────┐  ┌────────────────────┐  │
│  │  Static files   │  │  /api/* functions   │  │
│  │  (Vite build)   │  │  (Node.js)         │  │
│  └────────────────┘  └────────┬───────────┘  │
└───────────────────────────────┼───────────────┘
                                │ service key
                                ▼
                    ┌───────────────────────┐
                    │  Supabase             │
                    │  Postgres + Realtime  │
                    └───────────────────────┘
                                ▲
                                │ anon key (RLS)
                    ┌───────────────────────┐
                    │  Browser client       │
                    │  (for realtime only)  │
                    └───────────────────────┘
```

**Two Supabase keys, two purposes:**
- **Service key** — used by API functions on the server. Bypasses Row Level Security. Never exposed to the browser.
- **Anon key** — used by the browser client for realtime subscriptions. Subject to RLS. Safe to expose (it's in the JS bundle).

## Project Structure

```
project/
├── index.html
├── src/                    # Frontend (Vite)
├── api/                    # Vercel serverless functions
│   ├── events/index.ts     # Each folder = one endpoint
│   ├── state/index.ts
│   └── tsconfig.json       # SEPARATE tsconfig (CommonJS)
├── lib/                    # Shared server utilities
│   ├── supabase.ts         # Supabase client singleton
│   └── auth.ts
├── supabase/
│   ├── config.toml         # Local Supabase configuration
│   └── migrations/         # SQL migration files
├── vercel.json
├── vite.config.js
├── tsconfig.json           # Frontend tsconfig (ESNext)
├── .env                    # Local dev credentials
└── .env.test               # Test credentials
```

## Critical Setup Details

### Two tsconfigs

The root `tsconfig.json` uses ESNext modules (for Vite/browser). Vercel serverless functions need CommonJS. Without a separate `api/tsconfig.json`, your functions will crash in production with module format errors.

```jsonc
// api/tsconfig.json
{
  "compilerOptions": {
    "module": "CommonJS",
    "moduleResolution": "node",
    "target": "ES2020",
    "esModuleInterop": true
  },
  "include": ["./**/*.ts", "../lib/**/*.ts"]
}
```

The `include` path pulls in shared `lib/` utilities so both frontend and API can import from the same helpers.

### vercel.json routing

```json
{
  "framework": "vite",
  "routes": [
    { "src": "/api/(.*)", "dest": "/api/$1" },
    { "handle": "filesystem" }
  ]
}
```

This routes `/api/*` to serverless functions and everything else to the Vite build output.

### Local dev: two servers

Vite serves the frontend. `vercel dev` runs the API functions locally. Vite proxies `/api/*` to the Vercel dev server so the browser sees a single origin.

```jsonc
// package.json scripts
"dev": "vite",                              // Frontend on :3000
"dev:api": "vercel dev --listen 3001 --yes", // API on :3001
```

```javascript
// vite.config.js — proxy API calls to vercel dev
server: {
  port: 3000,
  proxy: {
    '/api': { target: 'http://localhost:3001', changeOrigin: true }
  }
}
```

Run both in separate terminals during development. The frontend works without the API server (for offline-first apps, the API is optional during dev).

### Supabase CLI for local database

```bash
supabase init          # Creates supabase/ directory
supabase start         # Starts local Postgres, Realtime, Auth, Studio
supabase db reset      # Reapply all migrations from scratch
supabase link          # Connect to cloud project for push/pull
supabase db push       # Apply local migrations to cloud
```

Local Supabase runs on `http://127.0.0.1:54321`. Credentials are deterministic (same for everyone) — safe to check into `.env`.

### Supabase client singleton (server-side)

```typescript
// lib/supabase.ts
import { createClient } from '@supabase/supabase-js';

let _supabase = null;

export function getSupabase() {
  if (_supabase) return _supabase;
  const schema = process.env.SUPABASE_SCHEMA || 'public';
  _supabase = createClient(process.env.SUPABASE_URL, process.env.SUPABASE_SERVICE_KEY, {
    db: { schema }
  });
  return _supabase;
}
```

The `schema` parameter enables test isolation — test env sets `SUPABASE_SCHEMA=test`, dev uses the default `public`.

### Test isolation via Postgres schemas

Create a `test` schema that mirrors your `public` schema tables. Tests use `SUPABASE_SCHEMA=test` so they hit `test.events` instead of `public.events`. Same database instance, fully isolated data.

```sql
-- migration: create test schema
CREATE SCHEMA test;
CREATE TABLE test.events ( /* same columns as public.events */ );
GRANT USAGE ON SCHEMA test TO anon, authenticated, service_role;
GRANT ALL ON ALL TABLES IN SCHEMA test TO anon, authenticated, service_role;
```

Don't forget to expose the `test` schema in `supabase/config.toml`:
```toml
[api]
schemas = ["public", "graphql_public", "test"]
```

### Environment variables for production

Set these in the Vercel dashboard (Settings > Environment Variables), never in a file:

- `SUPABASE_URL` — your cloud project URL (`https://xxxx.supabase.co`)
- `SUPABASE_ANON_KEY` — public key for browser client
- `SUPABASE_SERVICE_KEY` — secret key for API functions
- Any app-specific secrets (API tokens, etc.)

**Gotcha:** When using `vercel env add` via CLI, pipe with `printf` not `echo` — `echo` adds a trailing newline that becomes part of the value.

### Vite `define` for frontend config

Inject build-time config into the frontend bundle:

```javascript
// vite.config.js
define: {
  __SUPABASE_URL__: JSON.stringify(process.env.SUPABASE_URL || ''),
  __SUPABASE_ANON_KEY__: JSON.stringify(process.env.SUPABASE_ANON_KEY || ''),
}
```

These become global constants replaced at build time. The frontend never reads `process.env` directly.

## Gotchas

| Issue | Cause | Fix |
|-------|-------|-----|
| API functions crash with module errors in production | Root tsconfig uses ESNext, Vercel needs CommonJS | Add `api/tsconfig.json` with `"module": "CommonJS"` |
| `vercel dev` overrides env vars from `.env` | Vercel CLI auto-loads `.env` into process.env for serverless functions | Don't put environment-differentiating vars (like `SUPABASE_SCHEMA`) in `.env` — only put them in `.env.test` |
| Test data shows up in dev database | Test and dev share the same Postgres schema | Use `SUPABASE_SCHEMA=test` with a separate schema |
| `supabase start` fails | Docker not running, or port conflict | Start Docker Desktop; check nothing else is on port 54321 |
| Realtime not working locally | Table not added to realtime publication | Add `ALTER PUBLICATION supabase_realtime ADD TABLE your_table;` to migration |
| Deploy works but API returns 500 | Missing env vars in Vercel dashboard | Check all required vars are set for the Production environment |
| Supabase project pauses on free tier | No activity for 7 days | Visit the dashboard to unpause; set a reminder if tests depend on cloud |

## Deployment Checklist

- [ ] Vercel project created and linked to git repo
- [ ] All env vars set in Vercel dashboard (production)
- [ ] `api/tsconfig.json` with CommonJS module format
- [ ] `vercel.json` with API routing
- [ ] Supabase cloud project created and linked (`supabase link`)
- [ ] Migrations pushed to cloud (`supabase db push`)
- [ ] Realtime enabled for tables that need subscriptions
- [ ] Test schema created (if using schema-based test isolation)
- [ ] Auto-deploy on push to main configured
