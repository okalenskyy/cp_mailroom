# ADR-0009: Idempotency keys are required on every mutating tool call

- **Status:** accepted
- **Date:** 2026-04-29
- **Authors:** Trainer (sample ADR)
- **Deciders:** Trainer
- **Supersedes:** —
- **Tags:** safety, mcp, hooks, retries

---

## Context

Mailroom mutates the user's inbox via four MCP tools: `mail.create_draft`,
`mail.apply_label`, `mail.archive`, `mail.snooze`. Network calls fail.
Hooks reject and ask for retries. The agent loop occasionally
double-emits a tool call after a streaming hiccup. Without
deduplication, a duplicate call could:

- create two drafts on the same thread,
- apply a label twice (no harm, but wasteful),
- archive a message that was already archived (Gmail returns success;
  log noise but no real bug),
- snooze for 30 days, then snooze again for 30 days, *extending* the
  snooze beyond the policy bound.

Two of the four cases are real bugs. We need an architectural answer
that does not depend on the agent loop being well-behaved.

Out of scope: how plans are produced (ADR-0007), audit log format
(ADR-0006), and the choice not to ship `mail.send` in v1
(ADR-0010).

## Decision

Every mutating MCP tool call requires an `idempotency_key` parameter.
The format is fixed: `f"{plan_id}:{action_index}"` where `plan_id` is
the 12-char hex from the `TriagePlan` (per ADR-0007) and
`action_index` is the zero-based position of the action in
`plan.actions`.

The `PreToolUse` hook (`.claude/hooks/pre_tool_use.py`) enforces
**three** idempotency rules:

1. **Required.** Missing `idempotency_key` → deny.
2. **Format.** Must match regex `^[0-9a-f]{12}:\d+$`. Bad shape →
   deny.
3. **Replay-prevention.** The `idempotency_key` must not appear in
   `audit.jsonl` already. The hook reads the audit log on each call
   and refuses keys it has seen. Replay attempts → deny.

In addition, the implementations of the four write tools each treat
duplicate calls as no-ops:

- `apply_label` checks the message's existing labels and returns the
  same `result_summary` without an API call if the label is already
  present.
- `archive` checks `INBOX` membership and returns the same
  `result_summary` if the thread is already archived.
- `snooze` checks `SNOOZED` membership and rejects if already
  snoozed (to avoid the policy-bypass case).
- `create_draft` checks for an existing draft on the same thread
  with the same `idempotency_key` in the draft's `internalDate`-
  adjacent metadata; if found, returns the existing draft id.

## Consequences

### Positive

- **Safe retries.** A timeout-driven retry by the agent loop, the
  user, or a future scheduled task cannot double-apply.
- **Replay defence.** A captured `audit.jsonl` line cannot be
  replayed against the system to re-trigger an action; the hook
  refuses.
- **Auditable.** The key is the natural join column between the
  plan, the action list, the audit log, and Gmail's response.
- **Format constraint catches drift.** A regex-shape rule means a
  pair that switches `plan_id` to a UUID by accident gets a clear
  failure at the gate, not a silent bug downstream.

### Negative

- **The hook reads the audit log on every write call.** This is an
  O(N) scan; with N ~ 10⁴ lines (a heavy day) it adds ~3 ms p50.
  Acceptable; we add a tail-only fast path for N > 10⁵ in D13.
- **`snooze` rejecting already-snoozed messages is a small UX
  surprise.** Users who manually snooze and then run `/triage
  --apply` will see a denial. The denial message points them at
  `mail.search is:snoozed`.
- **The `create_draft` dedup check** uses Gmail metadata in a way
  that is technically not an idempotency primitive Gmail provides;
  we synthesise it. If Gmail's metadata model changes, we revisit.

### Neutral

- The audit log retains denied calls (with `decision: "deny"` and
  `reason`). Replay-prevention is therefore visible in the chain
  itself.

## Alternatives considered

### Alternative A — Server-side dedup only (no hook check)

- **Summary.** Trust the four write-tool implementations to dedup;
  skip the hook check.
- **Why rejected.** A bug in any one of the four tools becomes a
  safety bug. Defense-in-depth says the hook should refuse anything
  it has not seen approved, and the tool should be conservative on
  top of that.

### Alternative B — UUID `idempotency_key` (opaque)

- **Summary.** Generate random UUIDs at call time.
- **Why rejected.** A UUID has no relationship to the plan it
  belongs to. Joining the audit log to plans, debugging "why did
  this action run?", and writing the regex-shape gate all become
  harder. The plan-bound format is more constrained but
  dramatically easier to reason about.

## Implementation notes

- **All four write-tool wrappers in
  `src/mailroom/mcp_server/server.py`** declare
  `idempotency_key: str` as a required parameter.
- **`/triage --apply` slash command** (`.claude/commands/triage.md`)
  passes `idempotency_key=f"{plan_id}:{i}"` for the i-th action.
- **`PreToolUse` hook** adds a step between current invariants 4 and
  5 (per ADR-0007 §1–§7 numbering):
  ```python
  if not args.get("idempotency_key"):
      deny("idempotency_key required")
  if not re.fullmatch(r"[0-9a-f]{12}:\d+", args["idempotency_key"]):
      deny("idempotency_key format invalid")
  if _key_in_audit_log(args["idempotency_key"]):
      deny(f"idempotency_key already used")
  ```
- **`tests/test_hooks.py`** adds three cases:
  - `no_idempotency_key` (already in scaffold)
  - `bad_format_idempotency_key` (D9 deliverable)
  - `replayed_idempotency_key` (D9 deliverable; planted prior audit
    line)

## Open questions

- [ ] Should we add a "force replay" admin escape hatch (e.g., for
      tests)? Currently no. — owner: Trainer, due: post-sprint.

## References

- `.claude/hooks/pre_tool_use.py`
- `.claude/commands/triage.md`
- `src/mailroom/mcp_server/server.py`
- `src/mailroom/mcp_server/gmail_tools.py`
- `tests/test_hooks.py`
- ADR-0006 (audit log; the replay check reads it)
- ADR-0007 (plan-bound approval tokens; this ADR extends invariant set)
- Trainer's master plan, §3.5 ("The Approval Workflow"), invariant 3
