---
name: mvc-architecture
description: Use when adding functionality or significant new code — ensures logic lands in the right layer (model, view, or controller) with clean separation of concerns
---

# MVC Architecture

## The Layers

**Model** — State + business logic. Models own data, enforce invariants, and expose methods that encapsulate domain operations. They know nothing about HTTP, WebSockets, CLI, or rendering. This is where most of the interesting code lives.

**Controller** — Coordination. Controllers receive input (HTTP requests, WebSocket messages, CLI commands, events), call the appropriate model methods, and hand results to views. Controllers should be thin — if you're writing an `if/else` chain or a loop that computes something, it probably belongs in the model.

**View** — Display + rendering. Views turn model state into output: HTML, terminal text, JSON responses, status bars. Any string-building, formatting, or layout logic belongs here, not in the controller or model.

## Where Does This Code Belong?

| If you're writing... | It belongs in... | Not in... |
|---|---|---|
| A conditional that decides what to do with data | Model | Controller |
| A loop that transforms or filters a collection | Model | Controller |
| State tracking (maps, queues, counters) | Model | Controller |
| Validation or invariant enforcement | Model | Controller |
| Database queries, inserts, persistence logic | Model | Controller |
| A method that coordinates: parse input → call model → send response | Controller | Model |
| Webhook/event routing and dispatch | Controller | Model |
| Formatting a timestamp, building a status string | View | Controller |
| Rendering a list, composing terminal output | View | Controller |
| Assembling an HTML page or JSON response body | View | Controller |

## The Key Test

**Does this logic belong to the domain, or to the plumbing that connects it to the outside world?** Domain logic — state changes, validation, queries, persistence — belongs in the model. Plumbing — receiving HTTP requests, dispatching WebSocket messages, rendering output — belongs in the controller or view.

Database access lives in the model layer. The DB is part of the domain: it's how models persist and query state. The boundary that matters is between domain logic (model) and transport/display (controller/view), not between "pure code" and "code that does I/O."

## Common Anti-Patterns

**Multiple representations of the same concept** — Separate types for the same domain object (e.g. a `Task` interface, a `TaskRow` DB type, and a `TaskStore` abstraction all representing tasks). Pick one canonical model class and use it everywhere — persistence, routing, queuing, wire protocol. See `active-record-model` skill for how to structure it.

**Fat controller** — Business logic accumulating in the controller because "it's just a few lines." Those lines grow. Move them to a model method.

```typescript
// BAD — controller doing model work
wss.on("message", (msg) => {
  if (task.status === "pending" && worker.idle && !task.labels.includes("blocked")) {
    task.status = "assigned";
    task.workerId = worker.id;
    task.assignedAt = new Date();
    // ... more logic
  }
});

// GOOD — controller delegates to model
wss.on("message", (msg) => {
  const result = taskModel.assign(task.id, worker.id);
  if (result.ok) sendMsg(worker.id, { type: "task_assigned", ...result.task });
});
```

**Display logic in the controller** — Building strings, formatting dates, or composing output in the controller rather than a view/display module.

```typescript
// BAD — controller building display strings
const summary = `[${new Date().toISOString()}] Task #${task.id} assigned to ${worker.name}`;
console.log(summary);

// GOOD — view handles formatting
display.print(view.formatAssignment(task, worker));
```

**Model aware of transport** — A model that imports WebSocket, HTTP, or rendering modules has leaked concerns. Models emit events or return values; controllers and views handle I/O.

## Applies To Both Client and Server

The layers map differently depending on context, but the principle is the same: models own domain logic, controllers handle I/O, views handle rendering.

### Server

- **Model** — domain objects, DB queries, persistence, business rules
- **Controller** — HTTP/WebSocket handlers: parse request → call model → format response
- **View** — response formatting, log formatting, dashboard data shapes

### CLI / Terminal App

- **Model** — domain state and operations. A `Workspace` class that manages a git checkout, a `Settings` class that owns user preferences, a connection-state object. No display calls — these classes should emit events or return values, never call `print()` directly.
- **Controller** — handles input and coordinates. In a terminal app there are typically two kinds: a **REPL controller** (reads user input, dispatches commands) and a **protocol controller** (handles messages from an external source — a WebSocket, a daemon, a server). Controllers receive a `display` object and call it directly. That's appropriate: I/O is their job.
- **View** — the display object(s): renders text to stdout, manages status bars, formats diffs. A `Display` class with methods like `print()`, `printDiff()`, `printError()`. Receives config (e.g. verbose flag) in its constructor. The status bar is a reactive view-model that subscribes to model events and re-renders on change.

**The key anti-pattern for client code:** models calling display functions directly.

```typescript
// BAD — model knows about display
class Workspace {
  async reset() {
    await this._doReset();
    display.print("Workspace reset.");  // ← model shouldn't know about display
  }
}

// GOOD — model emits an event; controller or view handles display
class Workspace extends EventEmitter {
  async reset() {
    await this._doReset();
    this.emit("reset");                 // ← model just says what happened
  }
}

// In the controller that owns the workspace:
workspace.on("reset", () => display.print("Workspace reset."));
```

### Web Frontend (React / similar)

- **Model** — state atoms, stores, or context. Business rules and data transformations.
- **Controller** — event handlers, form submission logic, routing, data-fetching hooks.
- **View** — components, templates, CSS. Reads from model/store, calls controller handlers on user action.

## Dependency Management Across Layers

**Global infrastructure** (config, database, external API clients) — these are process-wide singletons. It's fine for any layer to access them directly via a module import or `getConfig()`. They don't need to be injected.

**Domain objects** (models, views, controllers) — these should either receive their dependencies via constructor injection, or create what they need with `new` at the point of use. Don't reach for a global when a constructor parameter is cleaner.

```typescript
// GOOD — Display is a view object; it receives config at construction
class Display {
  constructor(private config: BrunelConfig) {}
  print(line: string) {
    if (this.config.verbose) { ... }
  }
}

// GOOD — controller receives the display it needs
class WorkerSession {
  constructor(private display: Display, ...) {}
  onTaskAssigned(task: Task) {
    this.display.print(`Task #${task.number} assigned.`);
  }
}

// BAD — model reaching for a global display
class Workspace {
  async create() {
    // ...
    display.print("Created.");  // ← don't import display as a global in a model
  }
}
```

The pattern: **inject domain dependencies; access infrastructure directly.**

## Why This Matters

**Testability** — Most core logic lives in models, which can be tested with plain function calls. No need to spin up servers or mock I/O for the important stuff.

**DRY** — Multiple controllers (REST endpoint, WebSocket handler, CLI command) can share the same model methods instead of duplicating logic.

**Readability** — When a controller is thin, you can read it as a table of contents: "on this input, do this action, return this response." The details live in well-named model methods.

## Composition Over Inheritance Between Controllers

When one controller needs the capabilities of another — for example, a command dispatcher that uses a command registry — prefer **composition** (has-a) over **inheritance** (is-a). Inheritance implies a subtype relationship; composition is more honest about what's actually happening.

```typescript
// BAD — inheritance blurs the role boundary
class CommandController extends CommandRegistry {
  dispatch(input: string) { /* uses inherited register/lookup */ }
}
// Now a CommandController *is* a CommandRegistry — callers can't tell them apart.

// GOOD — composition makes the relationship explicit
class CommandRegistry {
  register(name: string, opts: CommandOpts): void { ... }
  lookup(name: string): CommandEntry | undefined { ... }
}
class CommandController {
  constructor(private readonly registry: CommandRegistry) {}
  dispatch(input: string) { /* calls this.registry.lookup() */ }
}
// Callers that only register commands accept CommandRegistry.
// Callers that dispatch accept CommandController.
```

**Narrow interfaces for dependency injection.** When a controller receives a dependency like a display object, type it as a narrow interface with only the methods that controller actually calls. This keeps coupling minimal and lets tests pass lightweight stubs without casts.

```typescript
// GOOD — only the methods this controller uses
interface WorkerDisplay {
  print(line: string | null): void;
  printForemanMessage(msg: Wire.ForemanMessage): void;
}
class WorkerSession {
  constructor(private display: WorkerDisplay) {}
}
// Tests can pass { print: vi.fn(), printForemanMessage: vi.fn() } without any cast.
```
