---
name: no-re-export
description: Use when about to write `export { X } from "./other-module"`, or when thinking that re-exporting symbols from another module is the right way to preserve backward compatibility or avoid updating callers
---

# No Re-Exports

## Don't. Here's Why.

Re-exporting a symbol from another module creates a phantom dependency: the consumer appears to depend on module B, but actually depends on module A through B. When A's interface changes, B's re-export silently breaks. It also makes the codebase harder to navigate — the source of truth for a symbol is obscured.

**There is no valid reason to re-export.** Not for convenience. Not for backward compatibility. Not to avoid updating callers.

## The Most Common Temptation: Splitting a File

You move symbols from `api.ts` into a new `http-client.ts`. Many files import from `api.ts`. Rather than updating all those imports, you add re-exports to `api.ts`:

```typescript
// BAD — re-exporting to avoid updating callers
export { request, buildHeaders, retry } from "./http-client.js";
```

This feels safe, but it defeats the purpose of the split. The old coupling is still there, just hidden. `api.ts` still "contains" everything it did before, from the consumer's perspective.

**The right move: update all callers.** Find every file that imports the moved symbols and change those imports to point at the new module. This is not optional, even if there are many callers.

```bash
# Find all callers to update
grep -r "from.*api" src/
```

```typescript
// Update each caller directly
import { request, buildHeaders } from "./http-client.js";  // was api.js
```

## Barrel Files Are the Same Anti-Pattern

An `index.ts` that re-exports everything from sibling modules is just a re-export with extra steps. Don't create one.

## The Rule

If you find yourself writing `export { X } from "./somewhere"`, stop. Either:
1. The consumer should import from `"./somewhere"` directly — update the consumer, or
2. X belongs in this module — move it here instead of re-exporting it
