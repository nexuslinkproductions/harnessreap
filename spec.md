# HarnessReap Protocol

**Status:** Draft-01  
**Spec version:** `draft-01`  
**Date:** 2026-08-10  
**License:** CC0-1.0 (spec text); implementations may use any license  
**Scope:** Developer machines running AI coding harnesses (Claude Code, Codex, OpenCode, Cursor, Hermes, custom MCP servers, LSP helpers, ephemeral dev servers)

> Working title for this draft is **HarnessReap**. See §0 for alternate name candidates. Implementations MUST advertise the protocol id `harnessreap` and the version string `draft-01` until a later draft renames or freezes the id.

---

## 0. Name candidates

| Candidate | Rationale | GitHub `full_name` exact hits (2026-08-10) |
|---|---|---|
| **harnessreap** | Short; names the actor class (AI harness) and the action (reap). Avoids “orphan” trademark/repo collisions. | none |
| **processlease** | Emphasizes the liveness/lease contract rather than killing. Good for adoption messaging (“take a lease, renew heartbeat”). | none |
| **spawnreap** | Focuses on spawn→supervise→reap lifecycle; tool-agnostic. | none |

Rejected / collided during naming scan: `orphanguard` (existing public repo), `spawnwatcher` (existing public repos). `mcpjanitor` is free but too MCP-specific for a general harness protocol.

This document uses **HarnessReap** as the normative protocol id for draft-01.

---

## 1. Motivation (grounding incident)

### 1.1 Incident summary (FACT)

On 2026-08-10, a MacBook Pro (Apple M2 Pro, macOS 26.4.1) ran **six orphaned GitNexus MCP Node.js processes** for approximately **4–5.5 hours** at **~80–98% CPU each** (~540% aggregate ≈ 5.4 cores). Observed signature:

- `PPID=1` (parent died; process reparented to launchd/init)
- Uncaught-exception **busy loop** (confirmed via stack sample)
- **Zero network sockets**
- **`SIGTERM` ineffective** (exception loop swallowed cooperative shutdown)
- Spawn lineage: `Hermes → Codex → gitnexus-mcp.mjs wrapper → gitnexus CLI mcp`
- Upstream GitNexus CLI already had a stdin-EOF fix (PR #2049), but the **wrapper** was the orphan seam: no stdin EOF handler, no signal forwarding, no SIGTERM→SIGKILL escalation, no process-group cleanup
- Remediation required identity-revalidated `SIGKILL` of those six PIDs only
- Wrapper fix committed in YURI-OS-MUSUBI (`e2f818766`): `superviseChild` with stdin EOF shutdown, SIGTERM/SIGINT/SIGHUP forward, grace then SIGKILL, idempotent latch

### 1.2 Adjacent forgotten-process class (FACT)

Same host also exhibited long-lived forgotten (usually idle) processes typical of AI-harness sprawl:

- 5× `whatsapp-mcp` Python processes with `PPID=1`, age **18–21 days**
- Astro `dev` server ~18h from `/private/tmp`
- Stale `tsserver` / `pyright` / YAML LSP helpers under harness trees
- Pre-fix GitNexus wrapper+CLI pairs still alive under live Codex

### 1.3 Problem statement

AI harnesses spawn many short-lived children (MCP servers, LSPs, CLIs, scrapers, temp servers). When the harness session dies, children can:

1. become **orphans** (`PPID=1`),
2. be **forgotten** (no registry, no owner session),
3. enter **pathological loops** that ignore cooperative signals and burn cores.

There is no shared open contract for identity, liveness, detection, or safe cleanup across harnesses. This protocol defines that contract.

---

## 2. Goals and non-goals

### 2.1 Goals

1. Canonical **identity + lineage** for every supervised spawn.
2. A **liveness lease** (heartbeat and/or stdin/pipe channel) with watchdog timeouts.
3. Deterministic **orphan / stale / suspect** detection heuristics.
4. A **lifecycle FSM** with explicit transition rules.
5. **Safe cleanup**: identity re-validation, children-before-parent, SIGTERM grace → SIGKILL escalation, dry-run default.
6. A local **registry + audit log** with privacy redaction.
7. Clear **harness integration points** (wrapper library, hooks, MCP companion, CLI, CI linter, daemon).
8. Versioned **event schema** and wire formats.
9. A **security & safety policy** that prefers false negatives over killing healthy work.
10. A practical **adoption path** from “no protocol” → inventory → linter → optional daemon enforce.

### 2.2 Non-goals

- Not a general anti-malware scanner.
- Not a RAM “cleaner” or placebo optimizer.
- Not a cross-user/system-wide killer of Apple/system daemons.
- Not a replacement for OS process accounting (`launchd`, cgroups, systemd).
- Not required to terminate processes it cannot identity-revalidate.

---

## 3. Terminology

| Term | Definition |
|---|---|
| **Harness** | Parent AI/tooling runtime that spawns children (Claude Code, Codex, OpenCode, Cursor, Hermes, etc.). |
| **Supervisor / wrapper** | Process that owns spawn+teardown (e.g. `gitnexus-mcp.mjs` + `superviseChild`). |
| **Record** | Canonical registry entry for one supervised process or process group. |
| **Lease** | Soft claim of liveness: must be renewed via heartbeat or pipe activity. |
| **Orphan** | Live process whose original parent is gone (`PPID=1` on Unix, equivalent on other OS). |
| **Forgotten** | Process with no live harness session and no valid registry lease (may still be orphaned). |
| **Identity re-validation** | Re-check pid + start time + exe fingerprint (+ optional argv hash / cwd) immediately before any signal. |
| **Linter mode** | Report-only; never signals. |
| **Daemon mode** | Optional enforce path; still dry-run unless auto-kill is explicitly enabled. |

---

## 4. Identity & lineage

### 4.1 Canonical process record

Every supervised spawn MUST create a record with at least:

| Field | Type | Notes |
|---|---|---|
| `record_id` | string (ULID/UUID) | Stable across pid reuse |
| `protocol` | `"harnessreap"` | Constant |
| `spec_version` | `"draft-01"` | Spec version |
| `pid` | integer | Current pid |
| `ppid` | integer | Parent at spawn / last observe |
| `pgid` | integer \| null | Process group id when available |
| `sid` | integer \| null | Session id when available |
| `start_time` | string (RFC3339) or OS boot-relative ticks | MUST be used in identity checks |
| `start_time_ticks` | integer \| null | Preferred OS-native start stamp for re-validation |
| `executable_path` | string | Resolved path when possible |
| `executable_hash` | string \| null | SHA-256 of executable bytes; optional but recommended |
| `argv_hash` | string | SHA-256 of normalized argv; **do not store raw secret-bearing argv** |
| `argv_redacted` | string[] \| null | Optional safe display argv after redaction |
| `cwd` | string \| null | Working directory |
| `uid` | integer \| null | Owner uid |
| `spawner` | object | See §4.2 |
| `session_id` | string \| null | Harness session / conversation / job id |
| `workspace` | string \| null | Project root / workspace path |
| `role` | string | e.g. `mcp-server`, `lsp`, `dev-server`, `cli-helper`, `wrapper` |
| `state` | enum | FSM state (§7) |
| `created_at` | RFC3339 | Record creation |
| `updated_at` | RFC3339 | Last mutation |
| `last_heartbeat_at` | RFC3339 \| null | Lease renewal |
| `lease_ttl_ms` | integer | Default 30000 |
| `watchdog_misses` | integer | Consecutive missed heartbeats |
| `sockets` | object \| null | Optional last observed socket summary |
| `cpu_pct_ewma` | number \| null | Optional smoothed CPU% |
| `signals_history` | array | Recent signal attempts / outcomes |
| `exit` | object \| null | `{ code, signal, at }` when terminated |
| `tags` | string[] | Free-form, non-secret |

### 4.2 Spawner identity

```json
{
  "harness": "hermes",
  "harness_version": "…",
  "supervisor": "gitnexus-mcp.mjs",
  "supervisor_version": "…",
  "parent_record_id": null,
  "chain": ["hermes", "codex", "gitnexus-mcp.mjs", "gitnexus-cli"]
}
```

### 4.3 Identity fingerprint

Implementations MUST compute:

```
fingerprint = H(
  pid || start_time_ticks || executable_path || executable_hash || argv_hash || cwd?
)
```

Before every kill/signal, re-read live process metadata and compare to the stored fingerprint. On mismatch: **abort signal**, mark record `SUSPECT` or `FORGOTTEN`, emit `identity-mismatch`.

Pid reuse without start-time check is a protocol violation.

---

## 5. Liveness contract

### 5.1 Lease model

A supervised child holds a **lease**:

- Default `lease_ttl_ms = 30000`
- Renewed by heartbeat event and/or observed pipe/stdin activity
- Watchdog may declare lease expired after `watchdog_miss_threshold` consecutive misses (default **3**)

### 5.2 Liveness channels (any one MAY suffice; wrappers SHOULD implement ≥1)

1. **Stdin/pipe EOF channel (required for stdio MCP wrappers)**  
   Supervisor listens for stdin `end`/`close` from the harness host and begins shutdown of the child (normative pattern from the GitNexus wrapper fix).
2. **Heartbeat events**  
   Child or supervisor emits `heartbeat` JSON events to the registry / local socket at ≤ `lease_ttl_ms / 2`.
3. **Supervisor ping**  
   Supervisor probes child (e.g. MCP ping / noop) and records success as heartbeat.

### 5.3 Grace periods

| Name | Default | Meaning |
|---|---:|---|
| `attach_grace_ms` | 5000 | NEW → ATTACHED window before orphan heuristics apply |
| `lease_ttl_ms` | 30000 | Heartbeat lease length |
| `term_grace_ms` | 500–2000 | After SIGTERM before SIGKILL (wrapper incident used 500ms; daemons MAY use 2000ms) |
| `orphan_age_threshold_s` | 300 | Min orphan age before STALE classification for low-CPU orphans |
| `busy_orphan_age_threshold_s` | 60 | Faster path for high-CPU orphans |
| `forgotten_age_threshold_s` | 86400 | Age after which an idle orphan may be labeled FORGOTTEN |

### 5.4 Supervisor MUST

1. Forward `SIGTERM` / `SIGINT` / `SIGHUP` (or platform equivalents) to the child.
2. On stdin EOF / harness disconnect: request child stop.
3. Escalate to `SIGKILL` after `term_grace_ms` if the child ignores cooperative signals.
4. Use an **idempotent latch** so repeated EOF/signal handlers do not race multiple escalations chaotically (single shutdown state machine).
5. Prefer supervising a **process group** when children spawn grandchildren.

---

## 6. Orphan & pathology detection

Detectors run in linter or daemon mode. Classification is additive; higher severity wins.

### 6.1 Heuristics

| ID | Condition | Suggested state |
|---|---|---|
| `H1` | `PPID==1` AND age ≥ `orphan_age_threshold_s` | `ORPHANED` → maybe `STALE` |
| `H2` | `PPID==1` AND `cpu_pct` ≥ `cpu_hot_threshold` (default 50) for ≥ `busy_orphan_age_threshold_s` | `SUSPECT` (busy orphan) |
| `H3` | `PPID==1` AND no sockets AND role ∈ {mcp-server, network-helper} AND age ≥ threshold | `STALE` |
| `H4` Exception-loop signature: high CPU + no I/O progress + cooperative signal ignored | `SUSPECT` → eligible for escalate after policy gates |
| `H5` Lease expired + parent harness session dead | `STALE` |
| `H6` Registry record exists but live fingerprint mismatch | `SUSPECT` / `FORGOTTEN` (stale record) |
| `H7` No registry record + matches known harness spawn patterns + `PPID==1` + age ≥ forgotten threshold | `FORGOTTEN` |

### 6.2 Exception-loop signature (H4)

All of:

1. `cpu_pct_ewma ≥ 70` for ≥ 60s
2. Negligible voluntary I/O progress (no socket bytes / no meaningful disk read-write progress — implementation-defined sampling)
3. At least one cooperative signal attempt recorded as ignored / still alive after `term_grace_ms`
4. Preferably stack/sample evidence when available (optional; not required for classification)

### 6.3 Exit / signal history

Records SHOULD retain a ring buffer (last N=16) of:

```json
{"at":"…","action":"signal","signal":"SIGTERM","result":"ignored|delivered|esrch|identity-mismatch"}
```

This history feeds H4 and audit.

### 6.4 Never-auto-classify as killable

- State `ATTACHED` with fresh heartbeat
- Processes under protected paths (§10)
- Processes failing identity re-validation
- Interactive user terminals / login shells
- System PIDs (pid 0/1, kernel tasks)

---

## 7. Lifecycle FSM

### 7.1 States

```
NEW → ATTACHED → ORPHANED → STALE → SUSPECT → TERMINATING → TERMINATED
                 ↘ FORGOTTEN
ATTACHED → STALE (lease expiry with dead session)
Any non-terminal → FORGOTTEN (untracked discovery / abandoned record)
TERMINATED is terminal
```

| State | Meaning |
|---|---|
| `NEW` | Record created; child starting |
| `ATTACHED` | Parent alive; lease healthy |
| `ORPHANED` | Parent gone (`PPID=1` or equivalent); lease may still be soft-alive briefly |
| `STALE` | Orphaned or lease-expired; idle or low utility; cleanup candidate under policy |
| `SUSPECT` | Pathological or identity-uncertain; needs extra gates before kill |
| `TERMINATING` | Cleanup in progress (SIGTERM clock running / escalation armed) |
| `TERMINATED` | Process gone; exit captured |
| `FORGOTTEN` | Discovered without owner / abandoned; inventory class; not necessarily hot |

### 7.2 Transition rules (normative)

| From | To | Trigger |
|---|---|---|
| — | `NEW` | `spawn` event / supervisor register |
| `NEW` | `ATTACHED` | First successful heartbeat OR supervisor confirms child running within `attach_grace_ms` |
| `NEW` | `TERMINATED` | Spawn failed / immediate exit |
| `ATTACHED` | `ORPHANED` | Parent death / `PPID==1` observed |
| `ATTACHED` | `STALE` | Lease expired AND harness session marked dead |
| `ATTACHED` | `TERMINATING` | Explicit supervised shutdown |
| `ORPHANED` | `STALE` | Age/CPU/socket heuristics (H1/H3/H5) |
| `ORPHANED` | `SUSPECT` | H2/H4 busy or signal-immune |
| `STALE` | `SUSPECT` | Becomes hot / signal-immune |
| `STALE`/`SUSPECT`/`FORGOTTEN` | `TERMINATING` | Cleanup authorized (user consent or auto-kill opt-in + policy) |
| `TERMINATING` | `TERMINATING` | Escalation `SIGTERM→SIGKILL` (same state, `escalated` event) |
| `TERMINATING` | `TERMINATED` | Process exit observed |
| `ORPHANED`/`STALE` | `FORGOTTEN` | Exceeds forgotten age OR inventory import without session |
| `*` | `SUSPECT` | Identity mismatch on re-validation |
| `TERMINATED` | — | No further transitions |

Illegal: `ATTACHED → TERMINATED` without `TERMINATING` unless the child exits on its own (then implementations MAY jump to `TERMINATED` with reason `self-exit`).

---

## 8. Cleanup semantics

### 8.1 Ordering

1. Enumerate children of the target record (and process-group members when tracked).
2. Signal **children → parent** (deepest first).
3. For each target: identity re-validate → `SIGTERM` → wait `term_grace_ms` → re-validate → `SIGKILL` if still alive and policy allows.
4. Mark `TERMINATED` only after `kill -0` / equivalent proves absence OR exit wait succeeds.

### 8.2 Kill-by-record, never kill-by-name

Implementations MUST NOT `pkill -f gitnexus` style broad name kills as a protocol cleanup action. Name patterns may be used only for **discovery hints**, never as the signal target selector.

### 8.3 Idempotent latches

Shutdown and escalation MUST be latch-guarded:

- first call arms SIGTERM + grace timer
- subsequent calls are no-ops until grace fires or exit occurs
- SIGKILL fires at most once per termination attempt generation

### 8.4 Consent hooks

| Mode | Behavior |
|---|---|
| `lint` | No signals |
| `interactive` | Prompt / UI / TUI confirm per record or batch |
| `daemon-dry-run` | Plan cleanup events only |
| `daemon-enforce` | Requires explicit opt-in config (`auto_kill=true`) AND record severity gates |

### 8.5 Hard rules

1. NEVER kill `ATTACHED` healthy leases.
2. NEVER SIGKILL without a prior cooperative attempt **unless** H4 already documented cooperative failure in-session OR user explicitly requests immediate SIGKILL for a validated record.
3. NEVER signal on identity mismatch.
4. NEVER traverse outside the user’s permission domain.

---

## 9. Registry

### 9.1 Location (XDG-aligned)

| OS | State root |
|---|---|
| macOS / Linux | `${XDG_STATE_HOME:-$HOME/.local/state}/harnessreap/` |
| Windows | `%LOCALAPPDATA%\harnessreap\` |

Override: env `HARNESSREAP_STATE_DIR`.

Suggested layout:

```
harnessreap/
  registry.sqlite        # primary
  registry.jsonl         # optional mirror / export
  events.jsonl           # append-only audit
  state.json             # daemon/linter snapshot (optional)
  config.toml            # user config
```

### 9.2 Storage format

- **Primary:** SQLite (`registry.sqlite`) with tables `records`, `events`, `sessions`.
- **Interchange / CI artifacts:** JSONL event stream + JSON statefile (§12).
- Implementations MAY run JSONL-only for MVP linters, but MUST be able to export the canonical record schema.

### 9.3 Retention

| Class | Default retention |
|---|---|
| Active records | Until `TERMINATED` + 7 days |
| Terminated records | 7–30 days |
| Audit events | 30–90 days |
| Inventory snapshots | Last 20 or 30 days |

### 9.4 Privacy

MUST:

- Hash argv; do not persist secrets
- Redact env (`TOKEN`, `KEY`, `SECRET`, `PASSWORD`, `AUTHORIZATION`, etc.)
- Avoid storing full raw command lines when they may contain credentials; store `argv_redacted` only
- Restrict file modes to user-only (`0600` / `0700`)

SHOULD:

- Support workspace path truncation
- Support `privacy.exclude_workspaces` globs

---

## 10. Security & safety policy

1. **Dry-run default.** All tools default to report-only until `auto_kill` is explicitly enabled.
2. **Identity re-validation before every signal.**
3. **No SIGKILL of non-validated records.**
4. **Protected-path exclusions** (never auto-target), examples:
   - `/System`, `/usr/libexec`, `/sbin`
   - `kernel_task`, `launchd`, WindowServer
   - user-configured deny paths
5. **Allowlist roles for enforce mode** (default): `mcp-server`, `lsp`, `dev-server`, `cli-helper`, `wrapper`.
6. **Rate limits:** max N kills / hour unless interactive override.
7. **Audit everything:** planned, skipped, signaled, escalated, aborted.
8. **Least privilege:** daemon runs as the user, not root. Root mode is out of scope for draft-01.

---

## 11. Harness integration points

### 11.1 Wrapper / supervisor pattern (normative minimum)

Any harness-spawned long-lived child SHOULD be launched through a supervisor that:

1. Registers a record (`spawn` event)
2. Implements stdin EOF → stop
3. Forwards termination signals
4. Escalates SIGTERM → SIGKILL with grace
5. Uses idempotent shutdown latch
6. Emits `terminated` / `cleaned` / `escalated` events

Reference pattern: `superviseChild` in YURI-OS-MUSUBI `_SYSTEM/Scripts/gitnexus-mcp.mjs` (commit `e2f818766`).

### 11.2 Opt-in switches

| Mechanism | Example |
|---|---|
| Env | `HARNESSREAP=1`, `HARNESSREAP_MODE=lint|daemon`, `HARNESSREAP_AUTO_KILL=0` |
| Config key | `harnessreap.enabled = true` in harness config |
| Wrapper library | `npx harnessreap-supervise -- …` / `import { supervise } from 'harnessreap'` |
| MCP companion | MCP server exposing tools in §12.3 |

### 11.3 Hook surfaces

| Harness | Integration idea |
|---|---|
| Claude Code | SessionStart/SessionEnd hooks register/expire session leases; PreToolUse spawn wrappers |
| Codex / OpenCode | Plugin hooks around MCP/tool process spawn |
| Cursor | Extension/companion scanning agent terminals + MCP child PIDs |
| Hermes | Gateway/plugin: wrap MCP launches; session-scoped registry |
| launchd | Avoid `KeepAlive=true` for ephemeral MCP helpers; if used, pair with HarnessReap lease checks so revive loops are intentional |
| CI | `harnessreap lint --json` report-only gate |

### 11.4 Modes

| Mode | Mutates processes? | Typical use |
|---|---|---|
| `lint` | No | CI + local doctor |
| `inventory` | No | One-shot migration scan |
| `daemon-dry-run` | No | Continuous detect + plan |
| `daemon-enforce` | Yes (opt-in) | Auto cleanup for allowlisted SUSPECT/STALE |

---

## 12. Protocol formats

### 12.1 Versioning

- Spec id: `harnessreap`
- Spec version: `draft-01`
- Event envelope always includes both.

Backward-compatible additive changes keep `draft-01` until a breaking draft (`draft-02`).

### 12.2 Event envelope

```json
{
  "protocol": "harnessreap",
  "spec_version": "draft-01",
  "event_id": "01JABCDEFGHIJKLMNOPQRSTUVW",
  "type": "spawn|heartbeat|orphan-detected|cleaned|escalated|identity-mismatch|state-changed",
  "ts": "2026-08-10T14:22:01.234Z",
  "record_id": "01J…",
  "session_id": "hermes-…",
  "data": {}
}
```

### 12.3 MCP tool wire format (draft)

Tools (companion MCP server):

| Tool | Args (JSON) | Result |
|---|---|---|
| `harnessreap.register` | `{record partial}` | `{record_id}` |
| `harnessreap.heartbeat` | `{record_id}` | `{ok:true, lease_expires_at}` |
| `harnessreap.list` | `{state?, workspace?}` | `{records:[…]}` |
| `harnessreap.lint` | `{paths?}` | `{findings:[…], exit_code}` |
| `harnessreap.plan_cleanup` | `{record_ids}` | `{plan:[…]}` dry-run |
| `harnessreap.cleanup` | `{record_ids, confirm:true}` | `{results:[…]}` enforce |

### 12.4 CLI linter exit codes

| Code | Meaning |
|---:|---|
| 0 | Clean (no stale/orphan/suspect findings above warn threshold) |
| 1 | Usage / config error |
| 2 | Findings present (STALE/ORPHANED/FORGOTTEN) — report-only failure for CI |
| 3 | SUSPECT / busy-orphan findings present |
| 4 | Registry corruption / identity inconsistency |
| 5 | Enforce mode attempted cleanup with partial failure |

---

## 13. Event schema examples

### 13.1 `spawn`

```json
{
  "protocol": "harnessreap",
  "spec_version": "draft-01",
  "event_id": "01J9SPAWN00000000000000001",
  "type": "spawn",
  "ts": "2026-08-10T10:05:12.000Z",
  "record_id": "01J9REC000000000000000001",
  "session_id": "hermes-20260810-codex-92028",
  "data": {
    "state": "NEW",
    "pid": 92101,
    "ppid": 92094,
    "pgid": 92094,
    "start_time": "2026-08-10T10:05:12.000Z",
    "start_time_ticks": 9876543210,
    "executable_path": "/Users/marcelspatz/.hermes/node/bin/node",
    "executable_hash": "sha256:…",
    "argv_hash": "sha256:…",
    "argv_redacted": [
      "/Users/marcelspatz/.hermes/node/bin/node",
      "/Users/…/gitnexus/dist/cli/index.js",
      "mcp"
    ],
    "cwd": "/Users/marcelspatz/YURI-OS-MUSUBI",
    "workspace": "/Users/marcelspatz/YURI-OS-MUSUBI",
    "role": "mcp-server",
    "lease_ttl_ms": 30000,
    "spawner": {
      "harness": "hermes",
      "supervisor": "gitnexus-mcp.mjs",
      "chain": ["hermes", "codex", "gitnexus-mcp.mjs", "gitnexus-cli"]
    }
  }
}
```

### 13.2 `heartbeat`

```json
{
  "protocol": "harnessreap",
  "spec_version": "draft-01",
  "event_id": "01J9HB00000000000000000001",
  "type": "heartbeat",
  "ts": "2026-08-10T10:05:40.000Z",
  "record_id": "01J9REC000000000000000001",
  "session_id": "hermes-20260810-codex-92028",
  "data": {
    "state": "ATTACHED",
    "pid": 92101,
    "lease_expires_at": "2026-08-10T10:06:10.000Z",
    "channel": "supervisor-ping",
    "cpu_pct_ewma": 1.2
  }
}
```

### 13.3 `orphan-detected`

```json
{
  "protocol": "harnessreap",
  "spec_version": "draft-01",
  "event_id": "01J9ORPH000000000000000001",
  "type": "orphan-detected",
  "ts": "2026-08-10T12:10:00.000Z",
  "record_id": "01J9REC000000000000000077",
  "session_id": null,
  "data": {
    "state": "SUSPECT",
    "from_state": "ATTACHED",
    "pid": 33724,
    "ppid": 1,
    "heuristics": ["H1", "H2", "H3", "H4"],
    "age_s": 16200,
    "cpu_pct_ewma": 93.4,
    "sockets": {"count": 0},
    "notes": "Parent harness/wrapper died; exception-loop signature"
  }
}
```

### 13.4 `cleaned`

```json
{
  "protocol": "harnessreap",
  "spec_version": "draft-01",
  "event_id": "01J9CLEAN00000000000000001",
  "type": "cleaned",
  "ts": "2026-08-10T16:20:11.000Z",
  "record_id": "01J9REC000000000000000077",
  "session_id": null,
  "data": {
    "state": "TERMINATED",
    "pid": 33724,
    "mode": "interactive",
    "signals": ["SIGTERM", "SIGKILL"],
    "identity_revalidated": true,
    "fingerprint_matched": true,
    "exit": {"code": null, "signal": "SIGKILL", "at": "2026-08-10T16:20:11.000Z"}
  }
}
```

### 13.5 `escalated`

```json
{
  "protocol": "harnessreap",
  "spec_version": "draft-01",
  "event_id": "01J9ESC0000000000000000001",
  "type": "escalated",
  "ts": "2026-08-10T16:20:10.500Z",
  "record_id": "01J9REC000000000000000077",
  "session_id": null,
  "data": {
    "state": "TERMINATING",
    "pid": 33724,
    "from_signal": "SIGTERM",
    "to_signal": "SIGKILL",
    "grace_ms": 500,
    "reason": "cooperative-signal-ignored",
    "identity_revalidated": true
  }
}
```

---

## 14. Statefile example

Path: `${HARNESSREAP_STATE_DIR}/state.json`

```json
{
  "protocol": "harnessreap",
  "spec_version": "draft-01",
  "generated_at": "2026-08-10T17:00:00.000Z",
  "host": {
    "os": "macos",
    "os_version": "26.4.1",
    "arch": "arm64",
    "hostname_hash": "sha256:…"
  },
  "mode": "lint",
  "config": {
    "auto_kill": false,
    "lease_ttl_ms": 30000,
    "term_grace_ms": 500,
    "orphan_age_threshold_s": 300,
    "busy_orphan_age_threshold_s": 60
  },
  "sessions": [
    {
      "session_id": "hermes-20260810-codex-92028",
      "harness": "hermes",
      "alive": true,
      "workspace": "/Users/marcelspatz/browserlink"
    }
  ],
  "records": [
    {
      "record_id": "01J9REC000000000000000001",
      "state": "ATTACHED",
      "pid": 92101,
      "ppid": 92094,
      "role": "mcp-server",
      "workspace": "/Users/marcelspatz/YURI-OS-MUSUBI",
      "last_heartbeat_at": "2026-08-10T16:59:50.000Z"
    },
    {
      "record_id": "01J9REC000000000000000088",
      "state": "FORGOTTEN",
      "pid": 18442,
      "ppid": 1,
      "role": "mcp-server",
      "workspace": "/Users/marcelspatz/whatsapp-mcp/whatsapp-mcp-server",
      "age_s": 1728000,
      "cpu_pct_ewma": 0.1,
      "heuristics": ["H1", "H7"]
    },
    {
      "record_id": "01J9REC000000000000000077",
      "state": "SUSPECT",
      "pid": 33724,
      "ppid": 1,
      "role": "mcp-server",
      "workspace": "/Users/marcelspatz/YURI-OS-MUSUBI",
      "age_s": 16200,
      "cpu_pct_ewma": 93.4,
      "heuristics": ["H1", "H2", "H3", "H4"]
    }
  ],
  "findings_summary": {
    "attached": 1,
    "orphaned": 0,
    "stale": 0,
    "suspect": 1,
    "forgotten": 1,
    "terminating": 0
  }
}
```

---

## 15. Adoption path

### 15.1 Harness opt-in (MVP → full)

1. **MVP linter (read-only):** install CLI; run `harnessreap inventory` once; then `harnessreap lint` in CI/local doctor. No wrappers required.
2. **Supervisor library:** wrap new spawns; emit `spawn`/`heartbeat`/`terminated`.
3. **Session hooks:** mark sessions alive/dead so lease expiry can distinguish crash from intentional detach.
4. **Daemon dry-run:** continuous detection; notifications only.
5. **Daemon enforce:** explicit `auto_kill=true` + allowlisted roles + rate limits.

### 15.2 Migration from “no protocol”

One-shot inventory algorithm:

1. Scan user processes.
2. Select candidates: `PPID==1` OR known harness parent patterns OR known MCP/LSP argv hashes.
3. Create `FORGOTTEN`/`ORPHANED`/`SUSPECT` records without killing.
4. Present report; optional interactive cleanup with identity checks.
5. Going forward, new spawns register as `NEW`.

### 15.3 Compatibility with existing fixes

HarnessReap does not replace per-project wrapper hardening. It standardizes the contract those wrappers should satisfy and provides a cross-harness detector when wrappers are missing or outdated (as with pre-fix GitNexus pairs).

---

## 16. Conformance checklist (draft-01)

An implementation may claim **HarnessReap draft-01 lint conformance** if it:

- [ ] Emits/consumes the event envelope with `protocol` + `spec_version`
- [ ] Can inventory and classify H1–H7 without signaling
- [ ] Uses identity fingerprints including start time
- [ ] Redacts secrets in persisted argv/env
- [ ] Implements exit codes in §12.4 for linter mode

An implementation may claim **supervise conformance** if it additionally:

- [ ] Implements stdin EOF shutdown OR equivalent disconnect channel
- [ ] Forwards termination signals
- [ ] Escalates to SIGKILL after grace with idempotent latch
- [ ] Registers spawn + terminated events

An implementation may claim **enforce conformance** only if:

- [ ] Dry-run is default
- [ ] Auto-kill is explicit opt-in
- [ ] Identity re-validation occurs before every signal
- [ ] Children-before-parent ordering is honored when relationships are known
- [ ] Audit events are persisted for cleaned/escalated/aborted

---

## 17. Open questions

1. Cross-platform start-time tick normalization (macOS `start_time_ticks` vs Linux `/proc` starttime vs Windows `CreateTime`).
2. Whether cgroup/job objects should be first-class for process-group cleanup on Linux/Windows.
3. Standard severity scoring for CI (warn vs fail thresholds).
4. Optional signed records for multi-user machines (out of scope for draft-01).
5. Formal JSON Schema publication path (`schema/draft-01/*.json`) in a future repo.

---

## 18. Appendix A — Incident → requirement mapping

| Incident fact | Protocol requirement |
|---|---|
| PPID=1 orphans after harness death | Orphan detection H1; supervisor must not rely on parent alone |
| Busy-loop ignored SIGTERM | Escalation + H4 + SIGKILL after grace |
| Wrapper missing stdin EOF | Liveness channel §5.2.1 normative for stdio MCP |
| Kill-by-identity needed | Fingerprint re-validation §4.3 / §8 |
| Harmless 18-day whatsapp-mcp orphans | FORGOTTEN class; inventory without panic kill |
| Pre-fix wrappers still alive | Inventory + version/tag awareness; lint can flag unsupervised patterns |
| Multiple harnesses (Hermes/Codex/Claude/Cursor) | Spawner chain + shared registry semantics |

## 19. Appendix B — Minimal config (`config.toml`)

```toml
spec_version = "draft-01"
mode = "lint"            # lint | inventory | daemon-dry-run | daemon-enforce
auto_kill = false
lease_ttl_ms = 30000
term_grace_ms = 500
orphan_age_threshold_s = 300
busy_orphan_age_threshold_s = 60
cpu_hot_threshold = 50
forgotten_age_threshold_s = 86400
allow_roles = ["mcp-server", "lsp", "dev-server", "cli-helper", "wrapper"]
protected_path_globs = ["/System/**", "/usr/libexec/**"]
privacy.redact_env_regex = "(?i)(token|key|secret|password|authorization|cookie)"
```

---

**End of HarnessReap Protocol draft-01.**
