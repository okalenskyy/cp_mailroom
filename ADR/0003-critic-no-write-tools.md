# ADR-0003: The Critic sub-agent has no write tools

- **Status:** accepted
- **Date:** 2026-04-29
- **Authors:** Trainer (sample ADR)
- **Deciders:** Trainer
- **Supersedes:** —
- **Tags:** agents, safety, drafts, audit

---

## Context

The Critic is a quality gate. It reviews `DraftCandidate` outputs from the
Drafter against a four-axis rubric (tone, grounded, correctness, no
overpromise) and reads the `audit.jsonl` after a `/triage --apply` run to
emit a markdown receipt of what actually happened.

If the Critic could call write tools — even legitimately, as a fallback
"correct the draft" path — the architecture would conflate quality
review with mutation. Two failure modes follow:

1. The Critic and the Drafter would compete for ownership of the
   `DraftCandidate`. A future maintainer would not know which agent
   "actually" produces the version that lands in Drafts.
2. A bug in the Critic could mutate the inbox. Today the Critic is
   read-only; a Critic bug at worst produces a wrong receipt.

Out of scope: the rubric's content (in
`.claude/skills/reply-drafter/SKILL.md`) and the audit-log format
(ADR-0006).

## Decision

The Critic's tool surface is **empty**. `src/mailroom/agents/critic.py`
imports nothing from `src/mailroom/mcp_server/gmail_tools.py`'s write
functions (`create_draft`, `apply_label`, `archive`, `snooze`). It can
read the audit log via the read-only `audit.tail` MCP tool, and it
consumes Pydantic objects passed in from the Lead Agent or the Drafter,
but it never originates a tool call that mutates state.

Concrete rules:

1. **No `mcp__mailroom__mail_(create_draft|apply_label|archive|snooze)`
   in the Critic's allowed-tools list.** Verified by grepping
   `src/mailroom/agents/critic.py` and `.claude/agents/` (the
   Critic does not appear in `.claude/agents/` because it is a runtime
   sub-agent, not a Claude Code subagent).
2. **No imports from `gmail_tools` write functions.** The
   `code-reviewer` subagent flags any new import.
3. **Output is always a structured `Verdict`.** Free-text rejections are
   forbidden because the Drafter cannot mechanically act on them.
4. **The Critic never calls the Drafter.** Rejection produces a
   `Verdict(accepted=False, findings=[...])`; the Lead Agent decides
   whether to re-invoke the Drafter or surface the rejection to the
   user.

## Consequences

### Positive

- **A bug in the Critic cannot mutate state.** Whatever the Critic does
  wrong, the user's inbox is unchanged.
- **Clean separation of duties.** Drafter produces; Critic reviews;
  Lead routes. New maintainers learn the architecture in three sentences.
- **Structured rejections compose.** `Verdict` is a Pydantic model
  (per ADR-0002); the Drafter can iterate on each finding independently.
- **The pattern transfers.** Critic-as-quality-gate is a teaching
  artifact participants can reuse in any future LLM project.

### Negative

- **Two-pass cost on rejected drafts.** A rejected draft means another
  Drafter call. We measured ~2.1× total tokens on rejected drafts; this
  is acceptable because evals show the rejection rate at <8%.
- **The Lead Agent owns retry policy.** Currently the Lead retries once
  on rejection and surfaces the second rejection to the user. Adding a
  third attempt would be a code change, not a Critic change — slightly
  more friction than a Critic-driven retry loop would have offered.
- **No "Critic auto-fixes typos" path.** A future maintainer might be
  tempted to add this; we intentionally do not, because it dissolves
  the boundary that makes the architecture safe.

### Neutral

- The Critic still reads the audit log via `audit.tail` (read-only); it
  does not fall back to direct file-system reads. This keeps tool
  governance consistent.

## Alternatives considered

### Alternative A — Self-critique inside the Drafter

- **Summary.** A single Drafter prompt that includes "now critique your
  output and revise" steps.
- **Why rejected.** Self-critique is empirically weak — the Drafter
  rationalises rather than rejects. In our 50-item evaluation, a single-
  agent self-critique loop caught 3/15 planted overpromises; the
  separate Critic caught 14/15.

### Alternative B — Critic with `mail.create_draft` for one-shot fixes

- **Summary.** When the Critic finds a small fix it could make
  (e.g., remove a stray "Sorry for the delay" without textual support),
  let it call `create_draft` directly.
- **Why rejected.** Two writers for `DraftCandidate` is exactly the
  conflation the architecture exists to prevent. The "small fix" path
  would grow until the Critic becomes a second Drafter. We keep the
  Critic read-only and let the Drafter take a second pass.

## Implementation notes

- `src/mailroom/agents/critic.py` defines two methods:
  ```python
  async def review_draft(candidate: DraftCandidate) -> Verdict
  async def review_audit(plan_id: str) -> str
  ```
  `Verdict` is in `schemas/draft_verdict.py` (D7 deliverable).
- The Critic uses Sonnet for `review_draft` (latency-sensitive) and
  Haiku for `review_audit` (volume-sensitive, cheaper).
- `tests/test_critic.py` includes:
  - `test_critic_rejects_overpromise` — input contains "I'll definitely
    have it by Friday" without thread evidence; expect `accepted=False`.
  - `test_critic_rejects_fabricated_recipient` — input contains a
    `to:` not present in the thread; expect `accepted=False`.
  - `test_critic_accepts_clean_draft` — input is a grounded reply;
    expect `accepted=True`.
  - `test_critic_has_no_write_tools` — imports `inspect`,
    asserts `critic.py` does not import `create_draft`, `apply_label`,
    `archive`, or `snooze`.

## Open questions

- [ ] Should the Critic emit a confidence score on each finding? —
      owner: Trainer, due: post-sprint (potential v2 ADR).

## References

- `src/mailroom/agents/critic.py`
- `src/mailroom/agents/drafter.py`
- `tests/test_critic.py`
- ADR-0002 (Pydantic hand-offs)
- ADR-0006 (audit log)
- Trainer's master plan, §3.2 ("The six sub-agents")
