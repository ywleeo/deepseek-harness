# Agent Note: Session close (live-agent disposal) and idle retirement

Status: implemented

English | [中文](2026-08-18-session-close.zh.md)

## Problem

`workspace.deleteSession` (the [session-deletion feature](2026-08-18-session-deletion.md)) refused every **live** session with `session-live`, and the error text told users to "close the session first" — but no close capability existed. The host discarded every `AgentHandle` and `AgentRegistry` had no dispose-by-id, so a session that was ever opened or prompted stayed attached (its agent resident in memory) until the host process exited: deleting "unopened" sessions failed, and the live set grew unbounded with every opened session.

## Decision

**Add dispose-by-id to `AgentRegistry`, a `session.close` unary, close-then-delete in `workspace.deleteSession`, a `host/session-closed` frame for the close path, and a configurable idle-retirement scan that disposes quiescent agents.**

- `AgentRegistry.create`/`resume` now retain each returned handle's disposer in a per-id map (dropped on `agent/disposed`), and a new `dispose(id, guard?)` runs that disposer — the loop's full teardown (stop, quiescence so the write-behind drains, unregister the agent, detach its session, unwind the scope). The optional `guard` is evaluated synchronously immediately before the disposer arms its abort, so a caller (idle retirement) that re-checks quiescence cannot race a fresh prompt into cancellation.
- `session.close({sessionId})` — disposes the live agent; idempotent `{closed: true}` for an already-cold persisted session; `session-not-found` for a definite miss; `agent-busy` for session-backed subagents (the ownership fence applies like every other verb).
- `workspace.deleteSession` is close-then-delete: the gateway disposes the live agent (subagent fence included) before calling the registry, so an attached-but-idle session deletes like a cold one. `session-live` survives only for a session re-resumed in the close→delete race window, and the registry still refuses live sessions as a non-web-caller safety net.
- **Host stream signaling**: the host stream maps `session/disposed` to `host/session-removed` (load-bearing for subagent detach). A close marks the id in a gateway-local `closingSessions` set for the duration of the dispose, so clients instead receive the new `host/session-closed` frame: the row STAYS (a closed session is cold, not deleted), running flips false, and live-only state (jobs, pending buffers/interactions) is dropped — mirroring the removal path minus the removal. Deletion still emits `host/session-removed` (from both the dispose and `session/deleted`).
- **Idle retirement** (`idleSessionCloseMs` on the gateway config, default 30 minutes, `0` disables): a 60-second scan disposes agents that have been quiescent (idle status, empty inbox) past the threshold. The clock starts on the agent's first quiescent observation; any activity (status flip or inbox work) resets it. Retirement passes the quiescence guard to `dispose` and never targets subagent-owned identities. A retired session is simply cold — the next prompt transparently resumes it.
- Client runtime: `ISessions.close` + `SessionRuntime.close` + `SessionManager.close` calling the unary; the `host/session-closed` case in `SessionManager.handleHostEnvelope`. UI: the session row menu gains a non-destructive **Close session** item (dialog-free, like archive); locales zh/en.

## Alternatives considered

**Deleting live sessions by having the registry dispose directly.** Rejected: agent teardown is the agent layer's job (`packages/core/agent`), and the registry (`packages/workspace`) should not own it; the gateway orchestrates close-then-delete where the full context (subagent fence, persistence) lives.

**Reusing `host/session-status { running: false }` for close.** Rejected: the client could not distinguish "agent went idle" (jobs stay) from "agent disposed" (jobs/pending state die), so close needs its own frame with removal-path cleanup minus the removal.

**Event-driven idle detection (retire on an idle signal).** Rejected in favor of a bounded periodic scan: a scan with a synchronous quiescence guard is simpler to reason about and test than wiring every wake/settle edge; the scan cadence (60 s) is far smaller than the default threshold.

## Consequences

- The misleading `session-live`/"(open in a client)" experience is gone: deleting any ordinary session works (close first), and idle sessions auto-close.
- Live-set memory now tracks recent activity instead of every session ever opened. A retired session costs nothing while cold and resumes transparently on the next prompt.
- `session.close` and idle retirement cancel a running turn if one is in flight (close semantics is dispose; the quiescence guard only protects against a turn that STARTED after the caller's own check). Clients keep the row and can keep typing — the next prompt cold-resumes.
- Subagent sessions remain fenced end to end: close, delete, and retirement all refuse them (`agent-busy` / skip), because their teardown belongs to the parent agent.
- Covered by tests: `AgentRegistry.dispose` (guard acceptance/refusal, external-teardown disposer cleanup), `session.close` (live dispose, cold idempotence, unknown, subagent fence, `host/session-closed` frame), delete-close (live delete, subagent fence), and the pure `pickIdleRetirementTargets` decision (quiescence, cutoff, inbox, subagent skip, clock start).
