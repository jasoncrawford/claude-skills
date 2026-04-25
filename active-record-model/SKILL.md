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
// src/db-client.ts
export let db: DbClient<Database>;
export function initDb(client: DbClient<Database>): void { db = client; }
```

Each model imports `db` from there. Never call `initModel(client)` separately per class.

Use generated types (e.g. `supabase gen types typescript --local > src/database.types.ts` for Supabase, or your ORM's codegen equivalent) so query results are typed without manual casting.

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

**Test isolation:** Add a `_reset()` static that clears the Map, and call it in `beforeEach`. Because the Map is module-level (not instance-level), tests share state across runs without an explicit reset.

```typescript
export class Worker {
  static _reset(): void { registry.clear(); }
}

// in tests:
beforeEach(() => Worker._reset());
```

**Max listeners warning:** If tests spin up many instances that each subscribe to `Model.events`, Node.js will warn about a potential memory leak. Suppress it at module level:

```typescript
Worker.events.setMaxListeners(0); // at bottom of worker-registry.ts
```

## Base Class (Optional Pattern)

When several models share the same CRUD boilerplate, extract it into an abstract base class rather than repeating it in each model:

```typescript
export abstract class ActiveRecord {
  protected static readonly tableName: string;
  protected static readonly primaryKey: string;

  // Override in subclasses to add joins, filters, or ordering
  protected static select() {
    return (db.from as any)(this.tableName).select("*");
  }

  static async get(id: string | number): Promise<any> {
    const { data, error } = await (this as any).select()
      .eq((this as any).primaryKey, id).maybeSingle();
    if (error) throw error;
    return data ? new (this as any)(data) : null;
  }

  static async insert(data: Record<string, unknown>): Promise<any> {
    const { data: row, error } = await (db.from as any)((this as any).tableName)
      .insert(data).select().single();
    if (error) throw error;
    return new (this as any)(row);
  }

  protected async update(changes: Record<string, unknown>): Promise<this> {
    const cls = this.constructor as typeof ActiveRecord;
    const { data, error } = await (db.from as any)(cls.tableName)
      .update(changes).eq(cls.primaryKey, this.getPrimaryKeyValue())
      .select().single();
    if (error) throw error;
    return new (cls as any)(data) as this;
  }
}

export class Order extends ActiveRecord {
  protected static readonly tableName = "orders";
  protected static readonly primaryKey = "order_id";

  // Narrow the inherited return type
  static get(id: string): Promise<Order | null> {
    return super.get(id) as Promise<Order | null>;
  }
}
```

Subclasses can override `select()` to add joins without touching any other CRUD method:

```typescript
export class Order extends ActiveRecord {
  protected static select() {
    // All finders (get, list, getBy) automatically include the join
    return (db.from as any)(this.tableName).select("*, customers(name)");
  }
}
```

The `as any` casts are intentional — `db.from(tableName)` takes a runtime string so the DB client can't infer per-table types. This is an acceptable trade-off for shared boilerplate; all public API surface still has explicit return types.

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
