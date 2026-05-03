---
name: devcontainer-setup
description: Use when setting up a new project for containerized Claude Code development, adding .devcontainer to an existing project, or wanting to run Claude with --dangerously-skip-permissions safely
---

# Devcontainer Setup for Claude Code

## Overview

Set up a Docker devcontainer for Claude Code development. The container provides filesystem isolation, network firewall (restricting outbound traffic to approved domains), and bidirectional sync of skills, settings, and plugins. Supports running Claude with `--dangerously-skip-permissions` safely.

## When to Use

- Starting a new project that will use Claude Code
- Adding containerized development to an existing project
- Wanting filesystem and network isolation for safe credential handling
- Wanting faster startup with pre-built base image (vs local builds)

**Not for:** iOS/macOS native development (requires Xcode/macOS toolchain, can't run in Linux containers).

## Quick Start

Create `.devcontainer/devcontainer.json`:

```json
{
  "name": "Your Project Name",
  "image": "ghcr.io/jasoncrawford/devcontainer-claude:latest",
  "remoteUser": "node",
  "features": {
    "ghcr.io/jasoncrawford/devcontainer-claude/setup:1": {}
  },
  "workspaceMount": "source=${localWorkspaceFolder},target=/workspace,type=bind,consistency=delegated",
  "workspaceFolder": "/workspace",
  "remoteEnv": {
    "GH_TOKEN": "${localEnv:GH_TOKEN}",
    "VERCEL_TOKEN": "${localEnv:VERCEL_TOKEN}",
    "CLAUDE_CODE_OAUTH_TOKEN": "${localEnv:CLAUDE_CODE_OAUTH_TOKEN}"
  },
  "waitFor": "postStartCommand"
}
```

Then start:

```bash
devcontainer up --workspace-folder .
devcontainer exec --workspace-folder . claude --dangerously-skip-permissions
```

Install `devcontainer` CLI with `npm install -g @devcontainers/cli` if needed.

## Architecture

Uses `jasoncrawford/devcontainer-claude` (published base image):
- Pre-built Node 20 image — faster startup, no local build
- Includes git, gh, zsh, Claude Code, brunel
- Feature (`setup:1`) automatically provides:
  - Network firewall (allowlist: GitHub, npm, Anthropic, VS Code, and custom domains)
  - Mount strategy (skills, settings, projects, plugins, gitconfig)
  - Container lifecycle (`post-create.sh`, `post-start.sh`)
  - Auth seeding and isolation

## For Projects Needing Extra System Packages

Create `.devcontainer/Dockerfile` extending the base:

```dockerfile
FROM ghcr.io/jasoncrawford/devcontainer-claude:latest
RUN apt-get update && apt-get install -y --no-install-recommends my-tool \
  && apt-get clean && rm -rf /var/lib/apt/lists/*
```

Then in `devcontainer.json`, replace `image` with:

```json
"build": {
  "dockerfile": "Dockerfile"
}
```

## Firewall Customization

For project-specific domains beyond GitHub/npm/Anthropic/VS Code defaults, create `.devcontainer/firewall-extras.txt`:

```
# Supabase
your-project.supabase.co
supabase.co
api.supabase.com

# Other services
api.stripe.com
```

The `setup` feature reads this file and adds domains to the allowlist.

## Authentication

### Claude Code
- User logs in once per devcontainer (per Docker volume)
- Auth persists across container restarts
- Volume survives `docker rm` — only cleared with `docker volume rm`

### GitHub CLI
- Store PAT in macOS Keychain:

```bash
security add-generic-password -a "$USER" -s "github-pat" -w "ghp_xxx"
```

- Extract in shell profile (`~/.zshrc`):

```bash
export GH_TOKEN=$(security find-generic-password -a "$USER" -s "github-pat" -w 2>/dev/null)
```

- Pass to container via `remoteEnv.GH_TOKEN` (already in quick-start config above)

## Container Lifecycle

```bash
# Start (pulls pre-built image)
devcontainer up --workspace-folder .

# Run Claude
devcontainer exec --workspace-folder . claude --dangerously-skip-permissions

# Shell
devcontainer exec --workspace-folder . zsh

# Full reset (loses auth, requires re-login)
docker ps  # find container ID
docker stop <id> && docker rm <id>
docker volume rm <name>-claude-state-<id>
devcontainer up --workspace-folder .
```

## Common Mistakes

| Mistake | Fix |
|---|---|
| Using local Dockerfile when base image works | Use published image + features. Add Dockerfile only if extra packages needed. |
| Forgetting `remoteEnv` for auth tokens | Include all three: `GH_TOKEN`, `VERCEL_TOKEN`, `CLAUDE_CODE_OAUTH_TOKEN` |
| Binding mount all of `~/.claude` read-write | Feature manages mounts automatically. Don't add custom mounts. |
| Trying to hardcode firewall domains | Use `.devcontainer/firewall-extras.txt`. Feature reads it automatically. |
| Expect `docker rm` to clear volume | Use `docker volume rm` for full reset. |
