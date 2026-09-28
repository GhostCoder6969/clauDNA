---
title: Session Store — Session-Segmented Capture Design
date: 2026-09-28
status: draft
authors: [chrisrogers37]
repos: [claudna, claudron, claudlobby]
---

# Session Store — Session-Segmented Capture Design

## 1. Problem

clauDNA captures nothing locally about what a session did. `telemetry-emit.sh` writes an unbounded, opt-in JSONL of skill calls; `/claudna:capture` distills knowledge straight into the vault; failures, boundaries, and "what happened in this task" are lost when the session closes. Two workloads need the same substrate:

- **Interactive work** — one session per task, then close out. Wants a per-session record with readers, and an end-of-session summary.
- **Claudlobby workers** — long-running, never bounded by a task, kept alive by manager-issued `/clear` and by compaction. Wants the same record, cut into bounded pieces a harvester can take incrementally.

**Goal:** a local, append-only, session-id-keyed store that clauDNA writes and organizes, with readers, a per-segment `claude -p` summary, and an export door that Claudron *optionally* pulls from.

**Non-goals:** fleet metrics (tokens / cost / latency — Claude Code's OpenTelemetry export, configured by Claudlobby); pushing to any hosted service; vault writes (Claudron owns ingestion, dedup, ranking).

**Prior art, used as concept reference only:** artemis-skills' session capture (per-session dirs, `failures.jsonl`, a detached `claude -p` summarizer, a field-merge map). This design takes the *concepts* and rebuilds from first principles; nothing is ported.

## 2. Observed harness behavior (canary, 2026-09-28)

Verified with hook-payload logging against Claude Code 2.1.284, headless `--input-format stream-json` (prompt → `/clear` → prompt → `/compact` → prompt):

| Hook | session_id | source / reason / trigger |
|---|---|---|
| SessionStart | A | `startup` |
| SessionEnd | A | `clear` |
| SessionStart | **B** (new; new transcript file) | `clear` |
| PreCompact | B | `manual` |
| SessionStart | B (same; same transcript file) | `compact` |
| SessionEnd | B | `other` |

Consequences this design is built on:

1. `/clear` mints a new session id and transcript → **a session is self-bounding at clears.**
2. Compaction keeps the id and transcript; both sides are observable → **compaction is a clean segment boundary.**
3. SessionEnd(clear) and SessionStart(clear) arrive back-to-back from one `claude` process → **clear lineage is linkable** (§4.3).
4. A nested `claude -p` launched from inside a session adopted the *parent's* session id, even with `CLAUDE_CODE_SESSION_ID` unset (observed in a cloud container; plain-machine behavior unverified) → **every child we spawn gets an explicit `--session-id`** (§7).

Not yet verified (open, §11): that PreCompact's transcript byte offset lines up with the compaction record; whether the `claude` pid is stable across `/clear`.

## 3. Principles

- **P1 — Logs are truth; JSON files are projections.** Every `.json` in the store is rebuildable from the `.jsonl` logs beside it (`session rebuild <sid>`). A torn or deleted projection is a cache miss, never data loss.
- **P2 — One writer per file.** Hooks append events. The projector (same process, right after the append) rewrites projections via temp + `os.replace`. The summarizer writes only its own artifact, then appends an event announcing it.
- **P3 — Reference, don't copy.** The transcript is never copied into the store. Segments point at it by path + byte range + content hash.
- **P4 — Metadata by default.** Free text (prompts, stderr) is off unless opted in, and when on is scrubbed (`scripts/redact.py` rules) and capped.
- **P5 — Boundaries are observed, never inferred.** Only harness hook events open or close sessions and segments. Counters are derived from what exists on disk, never stored.
- **P6 — Fail open, bounded.** A store hook never blocks, never fails a session, and returns in well under a second. Heavy work detaches.
- **P7 — Everything is versioned.** Every event carries `v`; every projection and artifact carries `schema: "claudna.<name>/<major>"`.

## 4. Identity and boundaries

### 4.1 Units

| Unit | Identity | Opened by | Closed by |
|---|---|---|---|
| **Session** | Claude Code `session_id` | SessionStart `startup` \| `clear` \| `resume` | SessionEnd (any reason) |
| **Segment** | `(session_id, index)`, index 1-based | session open; SessionStart `compact` | next compact; session close |

A segment is a contiguous byte range of the session's transcript. A session with no compactions has exactly one segment.

### 4.2 Hook → store action

| Hook event | Store action |
|---|---|
| SessionStart `startup` / `resume` | `session.opened` (resume: reopen existing dir, open a new segment); `segment.opened` |
| SessionStart `clear` | `session.opened` with `parent_sid` from the clear link (§4.3); `segment.opened` |
| UserPromptSubmit | `prompt.submitted` (segment events) |
| PostToolUse (Skill) | `skill.invoked` |
| PostToolUse (error result) | `tool.failed` |
| PreCompact | `segment.sealed` — record end offset; start summarizer on that range. Idempotent: a second PreCompact (first was blocked) re-seals with the later offset. |
| SessionStart `compact` | `segment.opened` for index N+1, start = last seal offset. If no seal exists (missed PreCompact), seal first at current transcript size. |
| SessionEnd | `segment.sealed` (final); `session.closed`; start summarizer; on `reason=clear` write the clear link. |

**Why cut at SessionStart(compact), not PreCompact:** PreCompact can fire without a compaction following (clauDNA's own `precompact-reflect.sh` blocks the first attempt by design). Sealing is safe to repeat; opening a segment is not.

**The directory is the holder.** There is no stored counter. Under an exclusive `flock` on the session dir, `next = max(existing seg-NNN) + 1`, and the new segment exists the instant its `mkdir` succeeds (atomic). Only the highest-index segment can be open, so "which segment do I append to" is the same lookup: `current = max(existing seg-NNN)`. A crash between `mkdir` and the `segment.opened` event leaves an empty dir that is still the correct current segment; `rebuild` backfills the event. A stored counter could disagree with the directories after a crash; a derived one cannot. `session.json.segments.open` caches the answer for readers but is never consulted by writers.

### 4.3 Clear lineage

SessionEnd(`reason=clear`) writes `links/<claude-pid>.json` naming the ending session. The next SessionStart(`source=clear`) from the same pid consumes it (reads, then deletes) and records `parent_sid`. The parent's `session.json` learns its child through a `session.child_linked` event appended to the parent's log. A link older than 60 s is ignored (stale). If no link is found, `parent_sid` is `null` — lineage is best-effort, never guessed.

Worker identity across a chain of clears is `actor.bot_name` (from `BOT_NAME`, already used by `telemetry-emit.sh`); lineage gives order within it.

## 5. Layout

```
${CLAUDNA_STATE_DIR:-~/.claudna}/
  sessions/
    <sid>/
      lifecycle.jsonl        # session-scope events (§6.2)          — truth
      session.json           # projection of lifecycle.jsonl (§6.4)
      summary.json           # deterministic rollup of segment summaries (§6.7)
      consumers.json         # export cursors per consumer (§6.8)
      seg-001/
        events.jsonl         # in-segment activity (§6.3)          — truth
        segment.json         # projection (§6.5)
        summary.json         # claude -p artifact (§6.6)
      seg-002/ …
  links/
    <pid>.json               # clear handoff, ephemeral (§6.9)
```

`~/.claude/notes/` and Claude Code's own directories are out of bounds (CLAUDE.md). Directory mode `0700`, files `0600`.

**One activity stream, not a `failures.jsonl`.** Failures are `tool.failed` events in the segment's `events.jsonl`; `session failures` is a reader view. One stream keeps ordering intact (a failure is readable next to the prompt and skill call around it) and adds one file kind instead of N.

## 6. Data models

Schemas ship as JSON Schema (draft 2020-12) in `schemas/session-store/`, one file per model below, with valid and invalid fixtures in `tests/fixtures/session-store/`. The field tables here are the human-readable spec; the schema files are normative.

### 6.1 Shared types

| Type | Shape |
|---|---|
| `Timestamp` | RFC 3339 UTC, millisecond precision, `Z` suffix — `2026-09-28T17:04:05.123Z` |
| `SessionId` | string, as issued by Claude Code (UUID today; treat as opaque) |
| `SegIndex` | integer ≥ 1; directory name `seg-%03d` (widens past 999 without breaking sort for readers that parse the int) |
| `ByteRange` | `{ "start": int ≥ 0, "end": int ≥ start \| null }` — `end: null` means open |
| `TranscriptRef` | `{ "path": string, "range": ByteRange, "sha256": string \| null }` — hash over the range, set at seal |
| `Actor` | `{ "kind": "interactive" \| "headless" \| "bot", "bot_name": string \| null, "model": string \| null, "entrypoint": string \| null }` |
| `Origin` | `{ "cwd": string, "repo": string \| null, "branch": string \| null, "head": string \| null }` — repo = `owner/name` from the git remote |
| `Text` | scrubbed string, capped per field (caps listed where used); `null` when capture is off |

### 6.2 Event envelope (`lifecycle.jsonl`, `events.jsonl`)

Every line in every log:

```json
{"v": 1, "ts": "2026-09-28T17:04:05.123Z", "kind": "segment.sealed", "sid": "3fbb…", "seg": 2, "data": { … }}
```

| Field | Type | Notes |
|---|---|---|
| `v` | int | envelope version; readers skip lines with an unknown major |
| `ts` | Timestamp | stamped by the writer, not the caller |
| `kind` | string | `<noun>.<verb-past>`; the registry below is closed — unknown kinds are skipped, not errors |
| `sid` | SessionId | |
| `seg` | SegIndex \| null | null for session-scope events that aren't tied to a segment |
| `data` | object | per-kind payload |

**Lifecycle kinds** (`lifecycle.jsonl`):

| kind | data |
|---|---|
| `session.opened` | `{ source: "startup"\|"clear"\|"resume", parent_sid: SessionId\|null, actor: Actor, origin: Origin, transcript_path: string }` |
| `session.child_linked` | `{ child_sid: SessionId }` |
| `session.privacy_set` | `{ private: bool, by: "user"\|"policy" }` |
| `segment.opened` | `{ opened_by: "session_open"\|"compact", start: int }` |
| `segment.sealed` | `{ end: int, sealed_by: "precompact"\|"compact"\|"session_end", trigger: "manual"\|"auto"\|null }` |
| `summary.requested` | `{ job_id: string }` |
| `summary.completed` | `{ job_id: string, artifact: "seg-NNN/summary.json", input_sha256: string, duration_ms: int }` |
| `summary.failed` | `{ job_id: string, error: string (≤200), retryable: bool }` |
| `session.closed` | `{ reason: "clear"\|"resume"\|"logout"\|"prompt_input_exit"\|"other" }` |

**Activity kinds** (`seg-NNN/events.jsonl`):

| kind | data |
|---|---|
| `prompt.submitted` | `{ prompt_id: string\|null, chars: int, text: Text (≤500, off by default) }` |
| `skill.invoked` | `{ skill: string, args_chars: int }` |
| `tool.failed` | `{ tool: string, signature: string, exit_code: int\|null, command: Text (≤300), error: Text (≤800) }` — `signature` is a stable grouping key (tool + normalized first error line), computed at write time so readers group without re-parsing |
| `checkpoint.noted` | `{ note: Text (≤1000) }` — from `/claudna:session checkpoint` |

### 6.3 Why two logs

`lifecycle.jsonl` is low-volume and session-scope (dozens of lines); it is what `session.json` projects from and what the export cursor reads. `events.jsonl` is per-segment and higher-volume; sealing a segment freezes its file. Readers that only need structure never touch activity.

### 6.4 `session.json` — projection of `lifecycle.jsonl`

```json
{
  "schema": "claudna.session/1",
  "sid": "3fbb…",
  "parent_sid": "533c…",
  "children": [],
  "status": "open",
  "private": false,
  "actor":  { "kind": "interactive", "bot_name": null, "model": "…", "entrypoint": "cli" },
  "origin": { "cwd": "/…/clauDNA", "repo": "Claudfather/clauDNA", "branch": "main", "head": "abc123" },
  "transcript_path": "/…/3fbb….jsonl",
  "opened_at": "…", "opened_by": "clear",
  "closed_at": null, "close_reason": null,
  "segments": { "count": 2, "open": 2 },
  "summary": { "segments_done": 1, "segments_pending": 1, "segments_failed": 0 },
  "projected_from": { "lines": 7, "bytes": 1432 }
}
```

`status`: `open` → `closed`. `segments.open` is the open segment's index or `null`. `projected_from` lets a reader detect a stale projection (log grew since) and rebuild.

### 6.5 `segment.json` — projection

```json
{
  "schema": "claudna.segment/1",
  "sid": "3fbb…",
  "index": 2,
  "status": "sealed",
  "opened_at": "…", "opened_by": "compact",
  "sealed_at": "…", "sealed_by": "session_end",
  "transcript": { "path": "/…/3fbb….jsonl", "range": { "start": 48211, "end": 90377 }, "sha256": "…" },
  "counts": { "prompts": 4, "skills": 2, "failures": 1, "checkpoints": 0 },
  "summary": { "status": "done", "job_id": "…" }
}
```

`status`: `open` → `sealed`. A sealed segment whose `range.end` changes (re-seal after a blocked compaction) is still `sealed`; `sealed_at` moves. `summary.status`: `none` | `pending` | `done` | `failed` | `skipped` (private, or summaries disabled).

### 6.6 `seg-NNN/summary.json` — summarizer artifact

The one LLM-authored file. Written atomically by the summarizer; the matching `summary.completed` event makes it visible to readers and export.

```json
{
  "schema": "claudna.segment-summary/1",
  "sid": "3fbb…", "index": 2,
  "input":    { "range": { "start": 48211, "end": 90377 }, "sha256": "…", "prior_rollup_sha256": "…" },
  "producer": { "model": "claude-haiku-…", "prompt_version": "segment-summary/1", "duration_ms": 8123 },
  "title": "Wire PreCompact seal into session store",
  "summary": "…(≤600)",
  "done":        [{ "text": "…", "evidence": ["commit:abc123"] }],
  "in_progress": [{ "text": "…" }],
  "next":        [{ "text": "…" }],
  "decisions":   [{ "text": "…", "why": "…" }],
  "gotchas":     [{ "text": "…" }],
  "knowledge":   [{ "text": "…", "type": "knowledge|decision|runbook", "confidence": "high|medium|low" }],
  "artifacts":   { "prs": ["…"], "commits": ["…"], "files": ["…"] },
  "tags": ["…"]
}
```

- Every list item is an object (not a bare string) so fields can be added without a breaking change.
- `knowledge` items are **candidates** for Claudron, not vault notes — no maturity, no slug. Claudron decides.
- `input.sha256` makes the summarizer idempotent: a re-run over an unchanged range is a no-op.
- `prior_rollup_sha256` records which rollup the model was shown as context, so a summary is reproducible.

### 6.7 `sessions/<sid>/summary.json` — deterministic rollup

No LLM. Recomputed from the `done` segment summaries on every `summary.completed`. Merge rules are declared once, as data, and the rollup, the reader, and the export all use the same table:

| Field | Rule |
|---|---|
| `title` | latest segment's |
| `summary` | latest segment's |
| `done`, `decisions`, `gotchas`, `knowledge`, `artifacts.*`, `tags` | union across segments, dedup on normalized `text` (or value), first occurrence wins, keep `from_seg` |
| `in_progress`, `next` | latest segment's only (older ones are superseded by definition) |

```json
{ "schema": "claudna.session-summary/1", "sid": "…", "through_seg": 2, "fields": { … }, "segments": [1, 2] }
```

### 6.8 `consumers.json` — export cursors

```json
{
  "schema": "claudna.consumers/1",
  "sid": "…",
  "consumers": {
    "claudron": { "through_seg": 1, "acked_at": "…" }
  }
}
```

Written only by `session export --ack <consumer> --through <seg>`. The store never deletes a segment that any registered consumer hasn't acked, unless it exceeds the hard age cap (§9).

### 6.9 `links/<pid>.json` — clear handoff (ephemeral)

```json
{ "schema": "claudna.clear-link/1", "pid": 4242, "sid": "533c…", "ts": "…" }
```

Consumed and deleted by the next SessionStart(clear) from that pid; ignored after 60 s; swept at SessionStart.

## 7. Summarizer

A pure function of `(segment range, prior rollup) → segment summary`, run out of band.

- **When:** on `segment.sealed` (PreCompact and SessionEnd). Never at SessionStart.
- **How it runs:** the hook appends `summary.requested` and spawns a detached worker (own process group, so a group kill of the hook tree does not reap it; `nohup` + job control, no `setsid` — macOS portability). The hook returns immediately.
- **Child isolation:** the worker runs `claude -p` with an explicit fresh `--session-id`, `CLAUDNA_SESSION_CHILD=1` (every store hook exits 0 immediately when set), a hard timeout, strict JSON output validated against §6.6 before write.
- **Input:** the segment's transcript range, reduced to user/assistant prose (tool I/O dropped), with injected context blocks stripped and the text framed as untrusted data. The prior rollup is included as context so segment 5 knows what segments 1–4 did.
- **Idempotent:** skips if a `done` artifact already has the same `input.sha256`.
- **Gates:** skipped (`summary.status: skipped`) if the session is private, if `CLAUDNA_SESSION_SUMMARY=0`, or if the segment is trivially small (no prompts).
- **Default:** on for interactive sessions; Claudlobby sets `CLAUDNA_SESSION_SUMMARY` per bot (a fleet that compacts often pays per segment).

## 8. Readers and the export door

Readers are verbs on the existing `/claudna:session` skill (no new skill), backed by one stdlib script:

| Verb | Shows |
|---|---|
| `session list [--since] [--repo] [--bot]` | sessions, newest first, with status and segment counts |
| `session show <sid>` | session.json + rollup, lineage (parent/children) |
| `session timeline <sid>` | merged lifecycle + activity, ordered |
| `session failures [<sid>] [--group]` | `tool.failed` events, grouped by `signature` |
| `session rebuild <sid>` | regenerate every projection from logs |
| `session export …` | the Claudron door (below) |

**Export contract** (the only surface Claudron pins; drift-gated on their side per Claudron register rule R3):

```
session export --consumer claudron [--since-seg N] --json
session export --consumer claudron --ack --sid <sid> --through <seg>
```

Returns an envelope: `{ schema: "claudna.export/1", items: [{ sid, seg, session: <session.json subset>, summary: <segment summary> }], next: <cursor> }`. Claudron reads through this command and never parses the store's files — the storage layout stays clauDNA's to change; only the export envelope is contract.

## 9. Retention

- A segment is deletable once every registered consumer has acked it, or once it passes the hard cap (default 30 days), whichever comes first.
- Sessions with no registered consumers use the age cap alone.
- Sweeping runs at SessionStart (bounded to a few ms of `stat` calls), never at SessionEnd.
- Private sessions are swept on the same rules; they are never exported.

## 10. Implementation shape

- **Language:** stdlib Python ≥ 3.11 (matches `scripts/`), a small package invoked by thin `plugin-hooks/*.sh` wrappers. No third-party runtime deps.
- **Modules:** `store` (paths, locking, atomic write, append), `events` (envelope + kind registry), `project` (log → projections), `boundaries` (hook → action table in §4.2), `summarize` (worker), `readers`, `export`. Each module owns one concern; the hook table is data, not branching.
- **Validation:** JSON Schemas in `schemas/session-store/`; a stdlib-only structural validator in tests (no new dev dep), fixtures per model, and a projection round-trip test (log → projection → rebuild equals).
- **`telemetry-emit.sh`:** migrates onto `skill.invoked` events; the old path stays as a deprecated alias for one release.

## 11. Open questions

1. **State dir default** — `~/.claudna/` proposed. Alternatives: `$XDG_STATE_HOME/claudna`.
2. **Prompt text capture** — off by default (§6.2). On for interactive, off for bots?
3. **Compact offset canary** — confirm PreCompact's transcript size equals the offset where post-compact content begins.
4. **Pid stability across `/clear`** — the clear link (§4.3) assumes the same `claude` pid; confirm with a hook that records the `claude` pid, not the hook shell's.
5. **Nested session-id inheritance** — observed in a cloud container; test on a plain machine. Claudlobby should spawn workers with an explicit `--session-id` regardless.
6. **Claudron pull verb** — Claudron-side work; this spec defines only the door it calls.

## 12. Phasing

1. **Store core** — `store`, `events`, `project`, schemas + fixtures, `rebuild`. No hooks wired.
2. **Boundaries** — hook wrappers for SessionStart / PreCompact / SessionEnd, clear lineage, segment cut. Canaries for §11.3–11.4.
3. **Activity** — `prompt.submitted`, `skill.invoked`, `tool.failed`; migrate `telemetry-emit.sh`.
4. **Readers** — `list`, `show`, `timeline`, `failures` on `/claudna:session`.
5. **Summarizer** — worker, per-segment summary, rollup.
6. **Export + retention** — export envelope, acks, sweep; hand the contract to Claudron.
