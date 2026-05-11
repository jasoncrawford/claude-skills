---
name: npm-publish
description: Use when setting up automated npm publishing from GitHub Actions, adding a publish workflow to a repo, or deciding how to authenticate npm publish in CI
---

# npm Publish from GitHub Actions

Use OIDC trusted publishing — not a stored `NPM_TOKEN`. No long-lived secret to manage; the token is scoped to this repo + workflow file and each publish gets a provenance attestation.

## Setup

**One-time on npmjs.com:** package → Settings → Publishing → add trusted publisher:

| Field | Value |
|-------|-------|
| Repository owner | your GitHub username or org |
| Repository name | the repo name |
| Workflow filename | `publish.yml` (must match exactly) |
| Environment | leave blank unless you use GitHub environments |

**Workflow:**

```yaml
on:
  push:
    tags: ["v*"]

jobs:
  publish:
    runs-on: ubuntu-latest
    permissions:
      id-token: write  # required for OIDC

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          registry-url: https://registry.npmjs.org
      - run: npm ci
      - run: npm publish --provenance
```

If `package.json`'s `files` field includes generated output (e.g. a built frontend), add `npm run build` before `npm publish`.

**Triggering a release:**
```bash
npm version patch   # or minor / major
git push --tags
```

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Using `NPM_TOKEN` | Switch to OIDC trusted publishing |
| Missing `permissions: id-token: write` | Required — OIDC token request fails without it |
| Missing `registry-url` in `setup-node` | Required — configures `.npmrc` for the right registry |
| Workflow filename doesn't match npmjs.com config | Must match exactly |
