---
name: offline-first-sync
description: Use when building an app that needs to work without a network connection, when adding multi-device sync to a local-first app, or when choosing a data architecture for a new app that may eventually need sync
---

# Offline-First with Event Sourcing and CRDT Sync

## Overview

Build the app as if the server doesn't exist. Local storage is the source of truth. The app works fully offline — no loading spinners, no "connecting..." screens, no degraded mode. Sync is a separate, additive layer that replicates data between devices. If you rip out the sync code, the app still works perfectly.

## When to Use

- Building any app where the user should never wait for a network request to see their data
- Adding multi-device sync to an existing local-first app
- Choosing between "fetch from server on load" vs. "read local, sync in background"
- Any app where users might be offline (mobile, travel, spotty wifi)

## The Architecture

Three independent layers, each of which works without the layers above it:

```
┌─────────────────────────────┐
│  3. Realtime (optional)     │  WebSocket push for instant cross-device updates
├─────────────────────────────┤
│  2. Sync (optional)         │  Push/pull events to/from server
├─────────────────────────────┤
│  1. Local event log         │  All mutations are events. State is projected from events.
└─────────────────────────────┘
```

Layer 1 is the app. Layers 2 and 3 are optional enhancements.

## Layer 1: Local Event Log

### Events, not state

Don't store the current state of your objects. Store the events that produced them. Current state is derived by replaying (projecting) the event log.

Three event types cover most apps:

| Event | Meaning |
|-------|---------|
| `item_created` | New item with initial field values |
| `field_changed` | Single field updated on an existing item |
| `item_deleted` | Item removed |

Each event carries:
- **id** — UUID, globally unique
- **itemId** — which item this affects
- **type** — one of the three above
- **field / value** — what changed (for `field_changed`)
- **timestamp** — when it happened (client clock)
- **clientId** — which device created it

### Projection

Current state is computed by replaying all events in order:

```
events: [
  { type: "item_created", itemId: "abc", value: { text: "Buy milk" } },
  { type: "field_changed", itemId: "abc", field: "completed", value: true },
]
→ projects to →
items: [{ id: "abc", text: "Buy milk", completed: true }]
```

Store the projected state in local storage as a cache (for fast reads). But the event log is the source of truth — if you lose the projected state, replay the log to rebuild it.

### Why events instead of state

- **Sync becomes trivial**: exchange events, not state diffs. No diffing algorithm needed.
- **Conflict resolution is per-field**: two devices can edit different fields of the same item without conflict.
- **Debugging is easy**: the full history of every change is in the log.
- **Undo is free**: (if you need it) just remove the last event and re-project.

### Per-field LWW timestamps

Every item tracks a `fieldUpdatedAt` timestamp for each mutable field. When projecting, a `field_changed` event is only applied if its timestamp is newer than the item's current timestamp for that field:

```javascript
// LWW check: only apply if this event is newer
if (event.timestamp < item[field + 'UpdatedAt']) continue;
item[field] = event.value;
item[field + 'UpdatedAt'] = event.timestamp;
```

This means two devices can independently edit different fields and both edits win. Only true conflicts (same field, same item) use last-write-wins — and for most apps, that's fine.

### Log compaction

The event log grows forever without compaction. After a successful sync (all events pushed, none pending), replace the entire log with synthetic `item_created` snapshot events — one per item, carrying the item's full current state. This collapses thousands of events down to N (number of active items).

Unsynced events are preserved through compaction — append them after the snapshots.

## Layer 2: Sync

Sync is push/pull over HTTP. No special protocol needed.

### Server schema

One table:

```sql
CREATE TABLE events (
  id         UUID PRIMARY KEY,
  item_id    UUID NOT NULL,
  type       TEXT NOT NULL,
  field      TEXT,
  value      JSONB,
  timestamp  BIGINT NOT NULL,
  client_id  TEXT NOT NULL,
  seq        BIGSERIAL NOT NULL  -- server-assigned ordering
);
```

The `seq` column is a server-assigned auto-incrementing sequence. Clients use it as a cursor for pulling.

### Push

Client sends all unpushed local events (where `seq === null`) to the server. Server inserts them (idempotent via `ON CONFLICT DO NOTHING` on the UUID primary key) and returns the assigned `seq` values. Client marks those events as pushed.

### Pull

Client sends its last-seen `seq` to the server. Server returns all events with `seq > cursor`. Client filters out its own events (by `clientId`), appends the rest to its local log, re-projects state, and advances the cursor.

Paginate pulls (e.g., 500 events per page) with a safety cap on total pages to prevent infinite loops.

### Sync cycle

```
push local events → pull remote events → compact log (if nothing pending)
```

Debounce the sync trigger (e.g., 2 seconds after last mutation) so rapid edits don't fire N requests.

### Retry with backoff

On sync failure, retry with exponential backoff (e.g., 5s base, 60s cap, 5 max retries). Reset retry count on success. Cancel pending retries when new mutations arrive (sync immediately instead of waiting).

### Ordering: Fractional indexing

User-defined ordering (drag-and-drop, insert-between) needs conflict-free position values. Fractional indexing uses lexicographically-sortable strings:

- Inserting between positions `"a"` and `"b"` produces `"an"` (midpoint)
- Inserting after `"z"` produces `"zn"` (append a character)
- No need to reindex existing items — positions are immutable once assigned

Always include a deterministic tiebreaker in your sort (e.g., `a.position.localeCompare(b.position) || a.id.localeCompare(b.id)`) so that items with identical positions have stable, consistent order across devices.

## Layer 3: Realtime (Optional)

Subscribe to database changes (e.g., Postgres `LISTEN/NOTIFY`, Supabase Realtime, or any WebSocket push). When a remote event arrives:

1. If the user is actively editing, **queue it** — don't interrupt their typing
2. Apply queued events when the user blurs / stops editing
3. Advance the cursor so the next pull doesn't re-fetch these events

Realtime is an optimization for responsiveness. The pull-based sync in Layer 2 is the source of correctness. If the WebSocket disconnects, the next pull catches up.

## API Design

Two endpoints are sufficient:

| Endpoint | Methods | Purpose |
|----------|---------|---------|
| `/api/events` | POST (push), GET (pull) | Event exchange |
| `/api/state` | GET | Server-side projection (for debugging/testing) |

The `/api/state` endpoint is optional — it replays all events server-side and returns the projected state. Useful for integration tests that need to verify server state without running a full client.

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Storing state instead of events | Store events. Derive state by projection. State in local storage is a cache. |
| Single timestamp per item (not per field) | Track `fieldUpdatedAt` for each mutable field. Otherwise all fields conflict when any field changes. |
| Diffing state to produce sync payloads | Exchange events directly. No diffing needed. |
| Applying remote events while user is typing | Queue them. Apply on blur. |
| No compaction | Compact after successful sync. Log size should be O(items), not O(all-edits-ever). |
| Integer positions for ordering | Use fractional indexing (strings). Integers require reindexing on insert. |
| Sort by position only, no tiebreaker | Add `|| a.id.localeCompare(b.id)`. Two devices inserting at the same position must sort deterministically. |
| Server clock for timestamps | Use client clock. Server just stores and relays. LWW uses client timestamps. |
| Sync required for app to function | App must work with sync layer removed entirely. If `enableSync()` never runs, nothing breaks. |
