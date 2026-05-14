# ADR-0007: Plan-bound approval tokens for every mutation

- **Status:** accepted
- **Date:** 2026-04-29
- **Authors:** Trainer (sample ADR)
- **Deciders:** Trainer
- **Supersedes:** —
- **Tags:** safety, hooks, mcp, approval-workflow

---

## Context

Mailroom mutates the user's inbox: it labels, archives, snoozes, and
creates drafts. The cost of a wrong action is high (lost mail,
sent draft to wrong recipient, contract email archived past a
deadline). The agent loop is non-deterministic and the email body is
attacker-controllable, so the *system*, not the agent, must enforce
safety.

By Day 8 we have a working `/triage` that produces a structured plan,
a Critic that reviews drafts, and write tools in the MCP server. We
need a mechanism that ties any individual write call to the plan the
user explicitly approved, in a way the `PreToolUse` hook can verify
deterministically.

Out of scope: the audit log format (ADR-0006), the
identifier-redaction policy (ADR-0006 §3), and v2's `mail.send` flow
(ADR-0010).

## Decision

We require every mutating MCP tool call to carry an `approval_token`
and an `idempotency_key`. The `approval_token` is the **12-character
hex id** of a plan file under `${MAILROOM_PLANS_DIR}/<id>.json`
(default `./.mailroom/plans/`) produced by `/triage` (or any other
read-only sub-agent that emits a plan).

The `PreToolUse` hook (`.claude/hooks/pre_tool_use.py`) validates the
token on every mutating call and enforces six invariants:

1. **`approval_token` is non-empty.** Missing → deny.
2. **The plan file exists** at `${MAILROOM_PLANS_DIR}/<id>.json`.
   Missing → deny.
3. **The plan is fresh.** `created_at` within the last
   `MAILROOM_PLAN_TTL_MINUTES` (default **30**). Older → deny.
4. **The call's `message_id`/`thread_id` is present in the plan's
   target IDs.** Mismatched → deny.
5. **For `mail.create_draft`, recipients pass the
   sensitive-allowlist check** unless `unlock_sensitive=True`.
   Allowlist (default): `legal@,pr@,finance@,security@`. Configurable
   via `MAILROOM_SENSITIVE_RECIPIENTS`.
6. **For `mail.snooze`, `days <= MAILROOM_MAX_SNOOZE_DAYS`** (default
   **30**). Larger → deny.
7. **Plus** `idempotency_key` must be unique within the plan; format
   `<plan_id>:<action_index>` (per ADR-0009).

Plans are produced read-only and saved before the user sees them. The
user moves from plan to action by re-invoking `/triage --apply
<plan_id>`, which threads the same plan_id into every write call.

Plan file shape (Pydantic, in `src/mailroom/schemas/triage_plan.py`):

```python
class TriagePlan(BaseModel):
    id: str = Field(default_factory=lambda: uuid.uuid4().hex[:12])
    created_at: datetime = Field(default_factory=
                                 lambda: datetime.now(timezone.utc))
    messages: list[ClassifiedMessage]
    actions: list[Action]
```

## Consequences

### Positive

- **Mutations cannot occur without an explicit, recent, plan-bound
  user step.** The hook denies any call missing its token, even if
  the agent loop tries to forge one.
- **The hook's checks are pure functions** of the plan file and the
  tool args; they are easy to unit test
  (`tests/test_hooks.py` — twelve red-team attacks ship on D10).
- **Idempotency keys make retries safe** (per ADR-0009). A duplicated
  approval cannot double-apply a label or archive a thread twice.
- **The plan file is the audit trail's anchor.** Every `audit.jsonl`
  line references the plan id (per ADR-0006), so a human can
  reconstruct what was approved.

### Negative

- **Adds ~12 ms p50 per write call** (sub-process spawn for the
  hook). The eval harness now runs slightly hotter — accepted as the
  price of safety.
- **Plans expire after 30 minutes**; users who walk away from a
  triage and return later get a friendly "plan expired" message
  instead of completion. This is *intended* but will surface as a UX
  nit for ~10% of sessions.
- **The hook now contains real logic; bugs in the hook can lock out
  legitimate writes.** We mitigate with the 12-attack red-team
  battery on Day 10.

### Neutral

- The plan format becomes a public contract between the Sorter and
  the hook. Future schema changes require a migration path or a
  compatibility window — see `MAILROOM_PLAN_SCHEMA_VERSION`
  (defaulted to `"1"`).

## Alternatives considered

### Alternative A — Confirmation-only via a slash command

- **Summary.** `/triage --apply` would simply re-invoke the write
  tools with no token; "I am running --apply" is treated as the
  consent.
- **Why rejected.** The Lead Agent could call `--apply` itself in a
  future refactor, defeating the gate. Defence-in-depth requires the
  proof of consent to live in the *call*, not in the invocation
  context.

### Alternative B — Per-call human confirmation prompt

- **Summary.** The hook prompts the user (yes/no) on every individual
  write call.
- **Why rejected.** A 50-action triage produces 50 prompts. We
  measured 11 s of friction per prompt; at 50 actions that is ~9
  minutes for what should be a 30-second operation. Users would
  learn to mash "yes" and the prompt becomes ceremonial.

### Alternative C — JWT-signed approval tokens with a per-pair private key

- **Summary.** Use HMAC-signed tokens carrying claims (target IDs,
  expiry, recipient allowlist) so the hook does not need to read
  disk.
- **Why rejected.** Adds a key-management problem we do not need on
  a single-machine sandbox project. Plan files on disk give us the
  same guarantees with no key rotation work, plus they double as the
  audit anchor (a JWT does not).

## Implementation notes

- **Env vars (in `.env.example`):**
  - `MAILROOM_PLAN_TTL_MINUTES` (default `30`)
  - `MAILROOM_MAX_SNOOZE_DAYS` (default `30`)
  - `MAILROOM_SENSITIVE_RECIPIENTS` (default `legal@,pr@,finance@,security@`)
  - `MAILROOM_PLANS_DIR` (default `./.mailroom/plans`)
- **Sorter call** at the end of classification:
  ```python
  plan = TriagePlan(messages=..., actions=...)
  plan.persist(settings.plans_dir)
  ```
- **All four write tools** in
  `src/mailroom/mcp_server/gmail_tools.py` accept
  `approval_token: str` and `idempotency_key: str` keyword args.
  Tool wrappers in `server.py` declare both as required.
- **The hook** lives at `.claude/hooks/pre_tool_use.py` and is wired
  in `.claude/mcp.json`'s `PreToolUse.matcher` regex
  `mcp__mailroom__mail_(create_draft|apply_label|archive|snooze)`.
- **Tests** (`tests/test_hooks.py`) cover six positive checks
  (legitimate calls pass) and six negative checks (each invariant
  blocks its corresponding attack).

## Open questions

- [ ] Should plans persist across reboots, or be tied to session?
      Currently plans live on disk indefinitely (until TTL). —
      owner: Trainer, due: Day 13 (potential ADR follow-up).

## References

- `src/mailroom/schemas/triage_plan.py`
- `.claude/hooks/pre_tool_use.py`
- `.claude/commands/triage.md`
- `tests/test_hooks.py`
- ADR-0001 (two scoped OAuth credentials)
- ADR-0006 (audit log; plan_id is the anchor)
- ADR-0009 (idempotency keys)
- ADR-0010 (`mail.send` would still need a token in v2)
- Trainer's master plan, §3.5 ("The Approval Workflow")
- `05_Mailroom_How_To_Write_ADRs.md` §6 (this ADR is the worked example)
