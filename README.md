# ProcessLease Protocol (draft-01)

Open protocol for **detecting, tracking, and cleaning up stale/forgotten processes** spawned by AI coding harnesses (Claude Code, Codex, OpenCode, Cursor, Hermes, custom MCP servers) on developer machines.

**Spec:** [`spec.md`](./spec.md) — version `processlease-0.1.0-draft-01`

## Why this exists

On 2026-08-10, six orphaned GitNexus MCP Node.js processes ran 4–5.5 hours at ~80–98% CPU each (~540% aggregate / 5.4 cores) after a Hermes→Codex→wrapper→MCP spawn chain lost its parent. Signature: `PPID=1`, uncaught-exception busy loop, zero sockets, SIGTERM ignored. The wrapper — not the upstream CLI — was the orphan seam.

The same class of leak shows up as forgotten MCP servers (days old), `/tmp` dev servers, and detached LSP helpers. Harnesses spawn many short-lived children; when the session dies without a lease or supervisor, the machine accumulates orphans.

## Name candidates

| Candidate | Rationale | GitHub `full_name` exact match (2026-08-10) |
|---|---|---|
| **ProcessLease** (preferred) | Emphasizes time-bounded ownership + heartbeat renewal | none |
| **HarnessReap** | Emphasizes AI-harness scope + cleanup | none |
| **SpawnReap** | Emphasizes spawn-chain lifecycle | none |

Avoided collisions: `orphanguard` (existing), `spawnwatcher` (existing), `orphankit` (near-hit on Orphan Kitten Project).

## What the protocol covers

1. **Identity & lineage** — canonical process records (pid/ppid/pgid, start time, exe/argv hashes, spawner, session, workspace)
2. **Liveness** — stdin/pipe or heartbeat lease with watchdog + grace
3. **Orphan detection** — PPID=1 + age/CPU; no-sockets; exception-loop (high CPU + no I/O + SIGTERM ignored)
4. **Lifecycle FSM** — NEW → ATTACHED → ORPHANED/STALE/SUSPECT → TERMINATING → TERMINATED / FORGOTTEN
5. **Cleanup** — children→parent, SIGTERM→SIGKILL, kill-by-record with re-validation, never kill ATTACHED
6. **Registry** — SQLite preferred / JSONL ok under XDG state; privacy redaction; audit log
7. **Harness hooks** — supervisor wrapper, launchd KeepAlive guard, Claude/Codex/OpenCode/MCP/CLI, linter vs daemon
8. **Wire formats** — versioned events, MCP tools, linter exit codes
9. **Safety** — dry-run default, opt-in auto-kill, protected paths
10. **Adoption** — `PROCESSLEASE=1`, inventory-once migration

## Modes

| Mode | Behavior | Default |
|---|---|---|
| **Linter** | Read-only report; CI-friendly exit codes | recommended first step |
| **Daemon** | Enforce cleanup after consent / opt-in | off until configured |

## Quick start (conceptual — no implementation in this directory)

```bash
# Opt in
export PROCESSLEASE=1
export PROCESSLEASE_MODE=lint   # or: enforce

# Report-only inventory of existing orphans
processlease inventory --dry-run

# Later: daemon with explicit auto-kill opt-in
processlease daemon --opt-in-auto-kill
```

## Non-goals (this directory)

- No repository creation
- No reference implementation
- Spec-only deliverable grounded in a real incident

## Status

`draft-01`. Feedback welcome against the motivating incident and the ten protocol sections in `spec.md`.
