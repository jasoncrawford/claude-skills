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

This separation works the same way regardless of context:

- **Server**: Model = domain objects + DB operations. Controller = HTTP/WebSocket handlers. View = response formatting, log formatting, admin dashboard data.
- **Client/frontend**: Model = state management + business rules. Controller = event handlers, form submission, routing. View = components, templates, rendering.

## Why This Matters

**Testability** — Most core logic lives in models, which can be tested with plain function calls. No need to spin up servers or mock I/O for the important stuff.

**DRY** — Multiple controllers (REST endpoint, WebSocket handler, CLI command) can share the same model methods instead of duplicating logic.

**Readability** — When a controller is thin, you can read it as a table of contents: "on this input, do this action, return this response." The details live in well-named model methods.
