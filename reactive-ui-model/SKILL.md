---
name: reactive-ui-model
description: Use when building or refactoring any real-time updating UI — terminal status bars, web dashboards with WebSocket/React, or any display that must reflect changing state. Also use when you find yourself writing manual refresh or update calls scattered across a class.
---

# Reactive UI Model

## Overview

A model holds all display state and emits events when it changes. The display subscribes once and refreshes automatically. Nothing else calls the renderer directly.

This beats scattered manual refresh calls: they drift out of sync as code grows.

## When to Use

- Terminal status bars (CLI workers, progress displays)
- Web dashboards driven by WebSocket or polling
- Any UI field that mirrors mutable backend state
- Refactoring code with many explicit `refresh()` / `updateStatus()` / `render()` calls scattered at mutation sites

**Not needed for** static output, one-shot rendering, or UIs that only update on explicit user action.

## Core Pattern

```
State mutation → model.update() → emit("change") → display re-renders
```

**Before (scattered manual calls):**
```typescript
this.connectionStatus = "connected";
this.disconnectCode = undefined;
this.refreshStatus();          // easy to forget
// ... 10 other places also call refreshStatus()
```

**After (reactive model):**
```typescript
this.statusModel.update({ connectionStatus: "connected", disconnectCode: undefined });
// display subscribed once in start() — refreshes automatically
```

## Implementation

### The model (TypeScript / Node EventEmitter)

```typescript
import { EventEmitter } from "node:events";

type StatusPatch = {
  connectionStatus?: "connected" | "disconnected" | "reconnecting";
  taskName?: string | undefined;
  branch?: string;
  // add fields as needed
};

class StatusModel extends EventEmitter {
  private _connectionStatus = "disconnected";
  private _taskName: string | undefined;
  private _branch = "";
  private _countdownTimer: ReturnType<typeof setInterval> | null = null;

  // Batching: multiple fields in one call → one "change" event
  update(patch: StatusPatch): void {
    if ("connectionStatus" in patch) this._connectionStatus = patch.connectionStatus!;
    if ("taskName" in patch) this._taskName = patch.taskName;
    if ("branch" in patch) this._branch = patch.branch!;
    this.emit("change");
  }

  // Timers belong in the model, not the consumer.
  // The display shows retryInSeconds; the model drives the countdown.
  startCountdown(ms: number): void {
    this._clearCountdown();
    this._reconnectAt = Date.now() + ms;
    this._countdownTimer = setInterval(() => this.emit("change"), 1000);
    this.emit("change");
  }

  stopCountdown(): void { this._clearCountdown(); }

  private _clearCountdown(): void {
    if (this._countdownTimer) { clearInterval(this._countdownTimer); this._countdownTimer = null; }
  }

  // Expose read-only getters as needed
  get connectionStatus() { return this._connectionStatus; }
  get taskName() { return this._taskName; }
  get branch() { return this._branch; }
}
```

### The display subscription (set up once, in `start()`)

```typescript
start(): void {
  // Subscribe ONCE. All future state changes auto-refresh the display.
  this.statusModel.on("change", () => this.display.updateStatusBar?.());
  this.display.startStatusBar?.(() => this.statusModel.getStatusText());
  // ... rest of startup
}
```

### React / web equivalent

```typescript
// Instead of EventEmitter, use a state atom or context
const [status, setStatus] = useState(model.snapshot());
useEffect(() => {
  model.on("change", () => setStatus(model.snapshot()));
  return () => model.off("change", handler);
}, []);
```

## Key Design Decisions

| Decision | Why |
|----------|-----|
| **Timer lives in the model** | Countdown state is model state. Consumers only read `retryInSeconds`. |
| **Batch multiple fields in one `update()` call** | One mutation → one "change" event → one re-render, not N. |
| **Subscribe once in `start()`** | Adding the subscription to call sites recreates the scattered-calls problem. |
| **`update()` uses `"key" in patch`** | Distinguishes "set to undefined" (clear) from "not mentioned" (leave unchanged). |

## The Renderer Is Also a Class

The reactive model pattern applies to the renderer too. The renderer has its own state (interval timers, active flags, text buffers, registered callbacks) — don't spread that across module-level variables. Make it a class.

```typescript
// BAD — module-level state + functions = implicit singleton
let _active = false;
let _text = "";
let _interval: ReturnType<typeof setInterval> | null = null;
export function startStatus(getText: () => string) { ... }
export function stopStatus() { ... }

// GOOD — renderer owns its state explicitly
export class StatusBarRenderer {
  private active = false;
  private text = "";
  private interval: ReturnType<typeof setInterval> | null = null;
  start(getText: () => string) { ... }
  stop() { ... }
}
```

The model holds domain state and emits events. The renderer subscribes and draws. Both should be classes.

## Common Mistakes

**Timer in the consumer, not the model:**
```typescript
// ❌ CountdownTimer managed by WorkerSession
this.countdownTimer = setInterval(() => this.refreshStatus(), 1000);
// ✓ CountdownTimer managed by StatusModel — call startCountdown(ms) instead
```

**Calling the display directly at mutation sites:**
```typescript
// ❌
this.currentBranch = branch;
this.refreshStatus();   // manual call — easy to miss elsewhere
// ✓ Just update the model; subscription handles refresh
this.statusModel.update({ branch });
```

**Separate clear + update calls causing double renders:**
```typescript
// ❌ Two "change" events → two repaints
model.clearTimeout();                         // fires "change" (retryAt: null, old status)
model.update({ connectionStatus: "connected" }); // fires "change" again
// ✓ Batch into one update so you get exactly one repaint
model.update({ connectionStatus: "connected", reconnectAt: undefined });
```

**Re-subscribing on each state change:**
```typescript
// ❌ subscribing inside the event handler that fires on each mutation
model.on("change", () => {
  model.on("change", () => display.refresh()); // duplicate listeners pile up
});
// ✓ subscribe once in start()
```
