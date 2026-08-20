# Agent Note: Session deletion (workspace.deleteSession)

Status: implemented

English | [中文](2026-08-18-session-deletion.zh.md)

## Problem

Session rows offered only **archive** (a display-layer hide that keeps the log and accounting slot) and a purely visual "Delete session" placeholder had been converted into archive without a handler. The product decision at archive time was "archive, not delete"; the user's real need is permanent removal of throwaway Q&A sessions: the persisted log must go (disk reclaim), the workspace accounting slots must go, the archive-set membership must go, and every client surface (workspace groups, Ungrouped, search, flat list) must drop the row.

## Decision

**Add `SessionPersistence.delete(id)` to the seam, a `deleteSession` on the workspace registry that cleans accounting and the archive set, and a new `workspace.deleteSession` unary that orchestrates log deletion + accounting cleanup and emits a `host/session-removed` frame for cold sessions.**

- Persistence seam: `SessionPersistence.delete(id)` is abstract. The coordinator implements it as a serialized per-id operation: wait for retirement, invalidate prepared sources, call a new optional backend hook `deleteStored(id)`, and remove the in-memory state. A backend without `deleteStored` rejects with a clear error; both first-party backends implement it — JSONL removes the session's `<root>/<project>/<session-id>/` directory, SQLite deletes the `sessions` row (events cascade). The seam's README Known Limitations change from "No deletion or retention API" to "retention policy remains deferred; deletion exists".
- Workspace registry: `deleteSession(sessionId)` rides the registry operation chain. It validates the session is known (live or persisted) — unknown ids reject with `WorkspaceUnknownSessionError` — refuses a **live** session with a new `WorkspaceSessionLiveError` (the host discards `AgentHandle`s and cannot dispose a live agent today, so deleting under a live write-behind would corrupt the log), then removes the id from every workspace's `sessionIds` account and from `archivedSessionIds`, durably. The registry emits the cordis event `session/deleted` (declared on the sessions domain's Events, alongside `session/disposed`) after the durable write.
- RPC: `workspace.deleteSession({sessionId}) → { deleted: true }`. Unknown ids map to the existing `session-not-found` error code; live sessions map to a new `session-live` code. The handler calls `sessionPersistence.delete(id)` then `workspaceRegistry.deleteSession(id)`; the host stream's `session/deleted` listener pushes `host/session-removed`, reusing the client's existing full removal handling (summary drop, conversation removed flag, selection fallback to the New Session view). The message-feedback service cascades on the same event: its per-Session sidecar row is deleted on the same operation tail, so no orphan feedback row remains for the in-product deletion path.
- Client runtime: `IWorkspaces.deleteSession` + `WorkspaceRuntime.deleteSession` + `WorkspaceManager.deleteSession` calling the unary; the row disappears through the existing `host/session-removed` path, so no client-side list surgery is needed.
- UI: the session row menu gains a **Delete session** item (danger styling, confirmation dialog — the archive item stays non-destructive with no dialog). Locales zh/en.

## Alternatives considered

**Deleting log files out-of-band from the host (the pre-feature "maintenance" answer).** Rejected: it bypasses the seam, cannot work for the SQLite backend (no per-session artifact), and leaves workspace accounting and the archive set stale.

**Refusing live sessions at the RPC but allowing the client to dispose first.** Rejected for this iteration: `AgentRegistry` has no dispose-by-id and the host discards every `AgentHandle` (documented in the per-session-agent-presets note), so a live-disposal path is a separate lifecycle feature. `session-live` is the honest boundary today; the error text tells the user the session is currently open.

**Reusing `session/disposed` for cold removal.** Rejected: it fires only for live sessions; the new `session/deleted` event is the cold-session counterpart and maps to the same `host/session-removed` frame.

## Consequences

- Deleting the currently-open session is refused (`session-live`); the client surfaces the host message. Deleting a session whose agent is attached but idle is also refused until a dispose capability ships.
- Deletion is terminal: no unarchive, no trash. The log and accounting are removed in one durable sequence; a crash between the two leaves the log gone with stale accounting slots that the startup header-index filter already tolerates (reported as "session header is missing"), and a subsequent registry mutation prunes them.
- The `workspace.list` response shape is unchanged (the deleted session simply leaves `sessionIds`); the archive set shrinks via the existing full-snapshot posture.
- The workspace-management e2e pins the archive round trip (pre-existing); the delete flow is covered by domain tests: unknown rejection, live rejection, accounting/archive-set cleanup, restart recovery. Persistence tests pin coordinator delete (retirement wait, prepared-source invalidation, missing-hook rejection) and both backend `deleteStored` implementations. The feedback cascade test pins row removal on `session/deleted`.
