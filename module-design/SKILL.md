---
name: module-design
description: Use when creating new modules, splitting existing ones, deciding what to export, or organizing related functions — ensures high cohesion, low coupling, and minimal public interfaces
---

# Module Design

## Core Principles

**High cohesion** — A module is responsible for one clear conceptual thing. Everything in the module relates to that thing. If you can't describe what the module does in one sentence without "and," it's probably two modules.

**Low coupling** — Modules depend on as few other modules as possible, and those dependencies flow in one direction. A change in module A should rarely force a change in module B.

**Minimal exports** — Only export what other modules actually need. Everything else is internal. The exported surface is the module's contract; the less you promise, the more freedom you have to change internals.

## Exports

**Default to private.** A symbol should be internal unless something outside the module needs it. "Might be useful later" is not a reason to export.

**Never re-export from other modules.** Not for convenience, not for backward compatibility. If consumer C needs something from module A, C imports from A directly. Re-exports create phantom dependencies: C appears to depend on B, but actually depends on A through B. When A changes, B's interface silently breaks.

```typescript
// BAD — re-exporting from another module
export { TaskQueue } from "./task-queue.js";
export { WorkerRegistry } from "./worker-registry.js";

// GOOD — consumers import directly from the source
import { TaskQueue } from "./task-queue.js";
import { WorkerRegistry } from "./worker-registry.js";
```

**When splitting a file, update all callers.** The right response to "callers import from the old file" is to update every caller to import from the new file — not to add re-exports from the old file so callers don't have to change. Re-exports for this reason are especially harmful: they make the split look complete while leaving the old coupling intact. Update all callers, even if there are many. This is not optional.

**No barrel files.** An `index.ts` that re-exports everything from sibling modules is a re-export with extra steps. It obscures where things live and defeats tree-shaking. If a directory needs an entry point, it should contain the module's own code, not a list of re-exports.

## Objects Over Loose Functions

When you have a collection of related functions — especially ones that share state, configuration, or service dependencies — organize them into an object (class or plain object) and export the object, not the individual functions.

```typescript
// BAD — loose functions that all take similar parameters
export function addTask(queue: Map<string, Task>, task: Task) { ... }
export function removeTask(queue: Map<string, Task>, id: string) { ... }
export function getNextTask(queue: Map<string, Task>) { ... }

// GOOD — state and methods live together in one object
export class TaskQueue {
  private queue = new Map<string, Task>();
  add(task: Task) { ... }
  remove(id: string) { ... }
  next() { ... }
}
```

This eliminates the repeated parameter, makes the shared state explicit, and gives consumers a single import instead of many.

**When a module has one primary class, make that class the only value export.** Constants, lookup tables, factory functions, and cache helpers that belong conceptually to the class should be static members of the class — not top-level exports alongside it. This keeps the class self-contained, reduces the exported surface, and makes the relationship explicit.

```typescript
// BAD — companion constants and helpers exported alongside the class
export const EFFORT_LEVELS = [...] as const;
export type EffortValue = typeof EFFORT_LEVELS[number];
export function findModel(models: ModelInfo[], id: string) { ... }
export function getCachedModels() { ... }
export class Settings { ... }  // one class among many exports

// GOOD — class is the only value export; companions are static members
export class Settings {
  static readonly EFFORT_LEVELS = [...] as const;
  static findModel(models: ModelInfo[], id: string) { ... }
  static getCachedModels() { ... }
  // ...instance members
}
export type EffortValue = typeof Settings.EFFORT_LEVELS[number]; // type export is fine
```

**Module-level mutable state is the same anti-pattern.** A module with `let` variables at the top and exported functions that close over them is just a singleton written badly — it has all the downsides (hidden state, untestable, can't have two instances) with none of the clarity of a class.

```typescript
// BAD — module-level state + exported functions = implicit singleton
let _active = false;
let _text = "";
export function start(getText: () => string) { _active = true; ... }
export function stop() { _active = false; ... }
export function update() { _text = ...; }

// GOOD — explicit class, state is visible and owned
export class StatusBar {
  private active = false;
  private text = "";
  start(getText: () => string) { this.active = true; ... }
  stop() { this.active = false; ... }
  update() { this.text = ...; }
}
```

## Module Boundaries

**Split by concept, not by layer.** A module for "all the database functions" or "all the utilities" has low cohesion. A module for "task persistence" or "event formatting" has high cohesion.

**One module, one owner of its state.** If two modules both read and write the same data structure, one of them should own it and the other should go through the owner's interface.

**Dependencies should flow toward stability.** Core domain modules should not depend on transport, rendering, or framework-specific code. Depend on things that change less often than you do.

## When to Split a Module

- It has two groups of exports that are used by non-overlapping sets of consumers
- It has two internal concerns that don't interact (they share no private state or helpers)
- Its description requires "and" — "task state **and** WebSocket plumbing"

## When NOT to Split

- The pieces would immediately need to import from each other (circular dependency = they're actually one concept)
- The split creates modules with 1-2 functions that only make sense in context of the original
- You're splitting for file size alone — a large cohesive module is better than several tiny coupled ones
