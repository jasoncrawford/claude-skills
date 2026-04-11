---
name: active-record-model
description: Use when creating model objects, adding database access, or designing a data layer — ensures model classes are self-contained, canonical, and properly emit change events
---

# Active Record Model Pattern

## One Class Per Concept

Every domain concept gets one canonical class. That class is the single representation used everywhere — persistence, routing, queuing, wire protocol. Multiple types for the same concept (e.g. `Task` interface + `TaskRow` + `TaskStore`) are a smell: consolidate.

For every DB table, there is exactly one model class. Non-DB models (in-memory only) follow the same pattern with a module-level Map as the backing store.

## Shared DB Client

Initialize the DB client once at startup, shared across all models:

```typescript
// src/foreman/db-client.ts
export let db: SupabaseClient<Database>;
export function initDb(supabase: SupabaseClient<Database>): void { db = supabase; }
```

Each model imports `db` from there. Never call `initModel(supabase)` separately per class.

Use generated types (`supabase gen types typescript --local > src/database.types.ts`) so query results are typed without manual casting.

## Class Structure

```typescript
export class Task {
  // ── Constructor (private) ──────────────────────────────────────────────────
  private constructor(row: TaskRow) { /* assign fields */ }

  // ── Computed/derived state (no DB call) ───────────────────────────────────
  get status(): TaskStatus { ... }

  // ── Static finders ────────────────────────────────────────────────────────
  static async get(id: string): Promise<Task | null>
  static async getByIssue(n: number): Promise<Task | null>
  static async list(opts?: ListOpts): Promise<Task[]>
  static async upsert(...): Promise<Task>

  // ── Instance mutations ────────────────────────────────────────────────────
  async complete(): Promise<void>   // write to DB, update this, emit event
  async assign(workerId: string): Promise<void>
  async delete(): Promise<void>

  // ── Test factory ──────────────────────────────────────────────────────────
  static fromTest(fields: Partial<TaskRow> & RequiredFields): Task
}
```

**Mutations must do three things:** write to DB, update `this` fields in memory (so callers see consistent state without re-fetching), and emit a change event.

## Change Events

Models emit events after mutations — they don't accept callbacks. Use a static EventEmitter on the class:

```typescript
export class Task {
  static readonly events = new EventEmitter();

  async complete(): Promise<void> {
    await db.from("tasks").update({ completed_at: now }).eq("task_id", this.taskId);
    this.completedAt = now;
    Task.events.emit("changed", this);
  }
}

// Elsewhere — any observer can subscribe:
Task.events.on("changed", () => adminWss.broadcastSnapshot());
```

This keeps the model ignorant of its observers. Any number of subscribers (dashboard, logger, test spy) can listen without the model knowing about them.

**Where to subscribe:** Observers should wire themselves up in their own constructor, not in the entry point (`index.ts`). If `TaskManager` needs to react to task changes, its constructor calls `Task.events.on(...)` — it owns that relationship. The entry point just instantiates the classes; it shouldn't know which models which observers care about.

```typescript
export class TaskManager extends EventEmitter {
  constructor() {
    super();
    Task.events.on("changed", () => this.emit("changed")); // TaskManager owns this wiring
  }
}
```

## In-Memory Models

Non-DB models follow the same pattern with a module-level Map:

```typescript
const registry = new Map<string, Worker>();

export class Worker {
  static register(workerId: string, ws: WebSocket): Worker { ... }
  static get(workerId: string): Worker | undefined { ... }
  static getIdle(): Worker[] { ... }

  assign(taskId: string): void { this.status = "busy"; this.currentTaskId = taskId; }
  remove(): void { registry.delete(this.workerId); }
}
```

No separate registry/store class needed — the Map is an implementation detail of the model.

## Testing

For tests that don't need the DB, mock at the model level:

```typescript
const task = Task.fromTest({ task_id: "t1", issue_number: 42, title: "Fix bug" });
vi.spyOn(Task, "getByIssue").mockResolvedValue(task);
vi.spyOn(task, "assign").mockImplementation(async (workerId) => {
  task.workerId = workerId; // mirror what the real method does in memory
});
```

`fromTest()` bypasses the DB and private constructor. Tests that are actually testing DB behavior should use a real DB instance, not mocks. See `test-discipline` skill.

## Anti-Patterns

**Intermediate store abstraction** — A `TaskStore` interface with a Supabase implementation and a separate in-memory implementation for tests. Just use the model class with `fromTest()` + `vi.spyOn` for tests that don't need the DB.

**Multiple representations of the same concept** — `GitHubEvent` (routing), `WebhookEventData` (insert), and `WebhookEvent` (DB row) all representing a GitHub webhook. Pick one canonical class; derive what you need from it.

**onChange callback on initModel** — Passing a callback into the model at initialization couples it to a specific observer. Use events instead so any number of observers can subscribe independently.
