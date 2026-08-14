# Agent Note: Unarchiving, an Archived view, and durable deletion for the workspace browser

Status: implemented

English | [中文](2026-08-14-unarchive-archived-sessions.zh.md)

## Problem

Archiving a session (`workspace.archiveSession`) added it to the registry-global archive set and hid it from every grouping surface — the grouped tree, the flat list, and search — with no way to bring it back: there was no unarchive RPC and no UI that listed archived sessions. A session archived by mistake, or archived before the user was done with it, was invisible and unreachable, which made "archive" a one-way action instead of a way to quiet the list. The only escape hatch was permanent deletion, and the append-only persistence layer had no `delete` primitive at all.

## Decision

`workspace.unarchiveSession({ sessionId })` removes one session from the registry-global archive set and answers the full updated set, mirroring `archiveSession` exactly. The workspace registry gains `unarchiveSession(id)`, which filters the id out of the durable `archivedSessionIds` state and resolves without writing for an id not in the set (the idempotent skip makes a lost unarchive retry safe). The existing `host/archived-sessions-changed` frame and `workspace.list` baseline already carry the set, so no new event or reconnect path is needed.

The workspace browser now renders an `Archived` section at the bottom of both the grouped tree and the flat list (`deriveArchived` filters the session list to top-level archived ids, newest first). Archived rows reuse the session row menu with `archive` replaced by `unarchive` plus a destructive `delete`; unarchive restores the row to its account slot, and delete opens a confirmation dialog before committing. Unarchiving restores the session's grouping-surface slot because archiving never touched workspace accounting.

Durable deletion is a persistence-layer primitive, not a workspace-browser hack. `SessionPersistence.delete(id)` (a non-abstract default that rejects, so test doubles inherit it) delegates to `PersistenceCoordinator.delete(id)`, which first waits for an in-flight retirement drain, serializes on the per-id chain, rejects a still-live (owned) session, invalidates any retained cold view, calls the new `PersistenceBackend.deleteStored(id)` hook, and drops the in-memory state. JSONL removes the session directory; SQLite deletes the `sessions` row (`events` cascade). `session.delete({ sessionId })` rejects only a genuinely running agent with `session-busy`, then disposes any idle attached session through the new `AgentRegistry.dispose(id)` — the registry retains every `create`/`resume` handle's shared teardown, so this also covers sessions attached outside `ensureSession` (subagents and forks) — and then calls `workspaceRegistry.removeSession(id)` (archive set plus every account slot) before the persistence delete. Disposal flushes and releases the coordinator's live ownership before the log is removed, so a partial failure leaves the session Ungrouped rather than a dangling account. The session-query read model converges through its existing list reconciliation.

## Alternatives considered

**Unarchive-only, no Archived view** — rejected. Restoring the RPC alone would make unarchiving possible only through a future surface; without a visible Archived section the user still could not find what they archived, which is the actual complaint. The view is the fix.

**An Archived filter/toggle instead of a fixed trailing section** — rejected. A filter complicates the existing grouped/flat/search derivation and hides archived sessions behind an extra control; a fixed section keeps them always visible where they land.

**Delete only the archived set without a persistence primitive** — rejected. Scrubbing accounting alone would leave the durable log orphaned on disk and resurrect it on the next list reconcile; delete must remove storage, so it rides the persistence layer.

**Force-delete a running session's log** — rejected. Disposing an in-flight agent's durable log mid-write is a different, riskier lifecycle problem, so `session-busy` still forces an explicit stop first; only idle attached sessions are disposed before deletion.

## Consequences

Users can see archived sessions again, restore them to their workspace (or Ungrouped), and permanently delete them from the Archived section in one confirmed action. The wire and reconnect baselines for unarchive are unchanged because they already carried the archive set. Deletion widens the persistence Service Definition and both first-party backends with one storage hook, and adds `AgentRegistry.dispose(id)` as the registry-level teardown for any live agent; the coordinator's live-session rejection remains the safety boundary that keeps the append-only contract from being violated by a mid-write removal.
