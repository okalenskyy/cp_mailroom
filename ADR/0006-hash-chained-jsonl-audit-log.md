# ADR-0006: Audit log is SHA-256 hash-chained JSONL on disk

- **Status:** accepted
- **Date:** 2026-04-29
- **Authors:** Trainer (sample ADR)
- **Deciders:** Trainer
- **Supersedes:** —
- **Tags:** audit, safety, privacy, observability

---

## Context

Every mutating tool call must leave a tamper-evident trail. The Critic
reads this trail to emit `/triage --apply` receipts (ADR-0003); the
trainer reads it during the Day 10 pen-test to verify hooks denied
the right attacks; the participant reads it post-sprint to debug.

The audit must satisfy four constraints:

1. **Tamper-evident** — silent edits to past entries are detectable.
2. **PII-redacted** — email bodies and addresses never land on disk in
   plaintext (per the privacy posture in ADR-0005).
3. **Append-only** — a workflow that needs to "rewrite" history is
   architecturally suspect; the format makes that hard.
4. **Operationally trivial** — no database, no daemon, no schema
   migrations during the sprint.

Out of scope: long-term archival, off-machine storage, and the choice
of redaction patterns (covered in implementation).

## Decision

We persist an **append-only JSONL file** at
`${MAILROOM_AUDIT_LOG}` (default `./.mailroom/audit.jsonl`). Each line
is a single JSON object representing one MCP tool call (read or
write). Each line carries the **SHA-256** of the previous line in
`prev`, plus its own `hash`, forming a tamper-evident chain.

Exact line shape:

```json
{
  "ts": "2026-05-08T14:23:11.482927+00:00",
  "tool": "mcp__mailroom__mail_create_draft",
  "args": {
    "thread_id": "...",
    "to": ["<email>", "<email>"],
    "subject": "Re: Q3 review",
    "approval_token": "a1b2c3d4e5f6",
    "idempotency_key": "a1b2c3d4e5f6:0"
  },
  "result_summary": {"id": "draft_xyz"},
  "prev": "0000...0000",
  "hash": "9f86d081884c..."
}
```

Rules:

1. **`prev` is the SHA-256 of the previous full line, hex-encoded,
   lowercase.** First-line `prev` is sixty-four `0`s.
2. **`hash` is the SHA-256 of the JSON-serialised record (with
   `prev` included, sorted keys) before `hash` itself is added.**
3. **Email addresses, phone numbers redacted before write.** Regex
   patterns:
   ```python
   EMAIL_RX = r"[\w.+-]+@[\w-]+\.[\w.-]+"   # → "<email>"
   PHONE_RX = r"\b(?:\+?\d{1,2}[\s-]?)?\(?\d{3}\)?[\s-]?\d{3}[\s-]?\d{4}\b"
                                            # → "<phone>"
   ```
4. **`body` and `raw` keys are dropped entirely.** Not redacted —
   removed.
5. **Two writers, one format.** The `PostToolUse` hook
   (`.claude/hooks/post_tool_use.py`) writes the chain when a tool is
   invoked through Claude Code; `src/mailroom/audit/log.py` writes the
   identical format when invoked directly (CLI, eval harness). The
   formats must be byte-for-byte interchangeable.
6. **Append-only.** Truncating `audit.jsonl` is a project crime. The
   file's parent (`.mailroom/`) is gitignored, so the chain never
   ships.

## Consequences

### Positive

- **Tamper-evidence is verifiable in five lines of Python.** Read each
  line, recompute the hash chain, fail on any mismatch. We ship a
  `mailroom audit verify` CLI command (D13) that does this.
- **PII redaction at write-time is safer than at read-time.** A bug
  in a future reader cannot leak prose that was never written.
- **No external dependency on a database.** The file is a single
  artifact a participant can copy, inspect, or replay.
- **Plays well with prompt caching.** The audit log is *not* part of
  any agent's input context, so it does not invalidate prompt caches
  when it grows.

### Negative

- **No structured queries.** "Show me every draft to legal@" requires
  `jq` or a small Python helper. We accept this; the volume is small
  and the security property is more valuable than the convenience.
- **Hash chain is fragile to concurrent writes.** Two
  `PostToolUse` invocations racing on the same file would corrupt the
  chain. We use a single MCP server process; concurrent writes are
  not currently possible. If the architecture changes (e.g., a
  background scheduled task), this becomes a real bug — the
  observability work in D13 includes a fcntl flock guard.
- **Redaction regexes are not perfect.** They catch the standard
  formats; an obfuscated address like `user [at] example [dot] com`
  passes through. We accept this on the throwaway corpus; the
  post-sprint v2 work hardens the redactor.

### Neutral

- The audit file is local-only. Off-machine archival, if ever
  desired, is a separate ADR.

## Alternatives considered

### Alternative A — SQLite database with append-only triggers

- **Summary.** Use SQLite, write a `BEFORE UPDATE/DELETE` trigger
  that raises.
- **Why rejected.** Adds a runtime dependency on `sqlite3` (fine in
  Python's stdlib but an extra surface for participants). The
  tamper-evidence property comes from SQLite's own integrity, which
  is not architectural — a maintainer with disk access can edit the
  DB file. The hash chain on JSONL is independent of the storage
  engine and explicit in code.

### Alternative B — Standard structured logging (`logging` module + JSON)

- **Summary.** Use Python's `logging` with a JSON formatter.
- **Why rejected.** Two issues. First, `logging` rotates files by
  default; rotation breaks the chain. Second, `logging`'s default
  redaction is none; we would still need a custom filter, at which
  point we have re-implemented the proposed solution but with a
  configurable third party in the path.

## Implementation notes

- **`PostToolUse` hook** is the canonical writer. Reading
  `.claude/hooks/post_tool_use.py` shows the exact format.
- **`src/mailroom/audit/log.py`** mirrors the format. The two writers
  share a (currently duplicated) regex/redaction block; D13 work
  factors them into `src/mailroom/audit/redact.py`.
- **`audit.tail` MCP tool** is the only read interface used by the
  Critic; it reads the last N lines after redaction.
- **`tests/test_audit_chain.py`** (new on D9 alongside the hooks)
  asserts:
  - first-line `prev` is sixty-four zeros,
  - `prev` of line N+1 equals SHA-256 of line N,
  - planted PII does not appear after a `write()` call.

## Open questions

- [ ] Should we sign each line with an HMAC over a per-machine key,
      so a maintainer with disk access cannot rewrite the chain
      undetected? — owner: Trainer, due: post-sprint (v2 candidate).

## References

- `.claude/hooks/post_tool_use.py`
- `src/mailroom/audit/log.py`
- `src/mailroom/mcp_server/server.py` (registers `audit_tail`)
- `tests/test_hooks.py`, `tests/test_audit_chain.py` (D9 deliverable)
- ADR-0003 (Critic reads via `audit.tail`)
- ADR-0005 (privacy posture)
- ADR-0007 (plan_id is the audit anchor)
- Trainer's master plan, §3.5 ("The Approval Workflow")
