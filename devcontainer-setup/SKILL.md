---
name: devcontainer-setup
description: Use when setting up a new project for containerized Claude Code development, when the user wants to run Claude with --dangerously-skip-permissions safely, or when adding a .devcontainer to an existing project
---

# Devcontainer Setup for Claude Code

## Overview

Set up a Docker devcontainer that lets Claude Code run with `--dangerously-skip-permissions` safely. The container provides filesystem isolation and a network firewall while sharing skills, settings, and plugins bidirectionally with the host.

## When to Use

- Starting a new project that will use Claude Code
- Adding containerized development to an existing project
- User wants to skip permission prompts safely

**Not for:** iOS/macOS native development (requires Xcode/macOS toolchain, can't run in Linux containers).

## Quick Start

Copy the template and customize the firewall:

```bash
cp -r ~/.devcontainer-template /path/to/project/.devcontainer
# Edit .devcontainer/init-firewall.sh — add project-specific domains
```

Then:

```bash
devcontainer up --workspace-folder /path/to/project
devcontainer exec --workspace-folder /path/to/project claude --dangerously-skip-permissions
```

Install `devcontainer` CLI with `npm install -g @devcontainers/cli` if needed.

## Template Location

`~/.devcontainer-template/` contains three files:

| File | Purpose | Customize? |
|---|---|---|
| `devcontainer.json` | Container config, mounts, env vars | Rarely — only the project name |
| `Dockerfile` | Node 20, git, gh, zsh, Claude Code, firewall tools | No |
| `init-firewall.sh` | Domain allowlist firewall | Yes — add project-specific domains |

## Mount Strategy

This is the critical design decision. **Never bind-mount all of `~/.claude` read-write** — Claude Code overwrites `.claude.json` on startup when it can't find macOS Keychain auth, which corrupts the host config.

Instead, use selective mounts:

| What | Mount type | Direction | Why |
|---|---|---|---|
| `~/.claude/skills` | bind (rw) | Bidirectional | Edit skills from either side |
| `~/.claude/settings.json` | bind (rw) | Bidirectional | Settings stay in sync |
| `~/.claude/projects` | bind (rw) | Bidirectional | Memory files sync |
| `~/.claude/plugins` | bind (rw) | Bidirectional | Plugins available in container |
| `~/.gitconfig` | bind (ro) | Host to container | Git identity for commits |
| `~/.claude` | bind (ro) | Host to container | Seeds `.claude.json` on first start |
| `.claude.json` state | Docker volume | Container only | Isolates runtime state from host |

The `postStartCommand` uses `cp -n` to seed `.claude.json` from the read-only host mount into the volume on first start. The `-n` flag means it won't overwrite once the user logs in.

## Authentication

### Claude Code
- User must log in once per project (per Docker volume)
- Auth persists in the volume across container restarts
- Volumes survive `docker rm` — only lost with `docker volume rm`

### GitHub CLI
- macOS Keychain tokens can't be forwarded to containers
- Use `GH_TOKEN` env var via `${localEnv:GH_TOKEN}` in `containerEnv`
- Store PAT in macOS Keychain, extract in `~/.zshrc`:

```bash
# Store once:
security add-generic-password -a "$USER" -s "github-pat" -w "ghp_xxx"
# In ~/.zshrc:
export GH_TOKEN=$(security find-generic-password -a "$USER" -s "github-pat" -w 2>/dev/null)
```

PAT needs: `repo`, `read:org` scopes (fine-grained, scoped to specific repos).

### Git Push via HTTPS

Even with `GH_TOKEN` set and `gh auth status` showing logged in, `git push` to `https://github.com/...` may fail with "could not read Username for 'https://github.com'". The container has no TTY for interactive credential prompts, and `GIT_ASKPASS=gh` also fails.

**Workaround:** Embed the token temporarily in the remote URL:

```bash
TOKEN=$(gh auth token)
git remote set-url origin "https://${TOKEN}@github.com/owner/repo.git"
git push -u origin <branch>
git remote set-url origin "https://github.com/owner/repo.git"  # restore clean URL
```

Also applies to `git pull` and `git fetch` when remote access is needed.

## Firewall Customization

Edit `init-firewall.sh` and add project-specific domains in the `for domain in` loop. Core domains (GitHub, npm, Claude API, Sentry, Statsig, VS Code) are already included.

Example additions for a Supabase + Vercel + Resend project:

```bash
    "your-project.supabase.co" \
    "supabase.co" \
    "supabase.com" \
    "api.supabase.com" \
    "vercel.com" \
    "api.vercel.com" \
    "vercel.live" \
    "api.resend.com" \
```

## Container Lifecycle

```bash
# Start (or restart existing)
devcontainer up --workspace-folder .

# Run Claude
devcontainer exec --workspace-folder . claude --dangerously-skip-permissions

# Drop into shell
devcontainer exec --workspace-folder . zsh

# Stop
docker ps  # find container ID
docker stop <id>

# Full reset (loses auth, must re-login)
docker stop <id> && docker rm <id>
docker volume ls | grep claude-state  # find volume
docker volume rm <volume-name>
devcontainer up --workspace-folder .
```

Idle containers use negligible resources. No need to stop/start routinely.

## Common Mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| Bind-mount all of `~/.claude` read-write | Host config corrupted on first start | Use selective mounts (see Mount Strategy) |
| `ipset add` without `2>/dev/null \|\| true` | Firewall script fails on duplicate IPs | Add error suppression to both ipset loops |
| Forget `--cap-add=NET_ADMIN,NET_RAW` | Firewall script can't set iptables rules | Include in `runArgs` |
| Mount `~/.config/gh` for GitHub auth | Fails — token is in macOS Keychain, not file | Use `GH_TOKEN` env var instead |
| Expect `docker rm` to clear volume | Volume persists, stale `.claude.json` blocks seeding | Use `docker volume rm` for full reset |
| Skip `cp -n` seeding of `.claude.json` | Onboarding flow shown on every new project | Keep read-only host mount + `cp -n` in postStartCommand |
