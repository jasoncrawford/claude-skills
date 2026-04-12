---
name: no-inline-import
description: Use when about to write an inline import (e.g. `const x = await import(...)` or `require(...)` inside a function body), or when thinking that a dynamic/inline import is the right solution to a problem
---

# No Inline Imports

## Don't. Here's Why.

Inline imports — `import()` or `require()` calls inside function bodies — are an anti-pattern. **Do not write them without explicit user permission.**

```ts
// Bad: inline import
async function doThing() {
  const { foo } = await import("./foo.js"); // ← don't do this
  foo();
}

// Good: top-level import
import { foo } from "./foo.js";

async function doThing() {
  foo();
}
```

## Why They're Bad

- **Hidden dependencies** — imports buried in function bodies are invisible to readers scanning the top of the file. The module's dependency graph becomes impossible to understand at a glance.
- **Deferred errors** — import failures surface at call time rather than at startup, making problems harder to catch and debug.
- **Harder to refactor** — static analysis tools (type checkers, bundlers, linters) work best with static imports. Inline imports break tree-shaking, circular dependency detection, and IDE "find all usages."
- **Fake optimization** — lazy loading is rarely worth it inside a backend service. The startup cost is paid once; call-time cost is paid on every invocation.

## The Common Temptations

**"I need to avoid a circular dependency."**
A circular dependency is a module design problem. Fix it: split the shared code into a third module that both can import. Do not mask it with a lazy inline import. See the `module-design` skill.

**"I want to lazy-load for performance."**
On a server or CLI, this is almost never a real win. On the frontend/bundler side, use the bundler's code-splitting API (e.g. `React.lazy`, a dynamic route), not a raw inline import sprinkled inside a function.

**"The type is only needed conditionally."**
Use a type-only top-level import (`import type { Foo } from "./foo.js"`). It is erased at compile time and has zero runtime cost.

## What To Do Instead

1. Move the import to the top of the file.
2. If that causes a circular dependency, identify the shared types or utilities and extract them into a new module.
3. If lazy loading is genuinely required (e.g. optional plugin system), make it explicit and document it — don't do it incidentally.
