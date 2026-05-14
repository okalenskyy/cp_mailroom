# ADR-0010: mail.send is intentionally not implemented in v1

- **Status:** accepted
- **Date:** 2026-04-29
- **Authors:** Trainer (sample ADR)
- **Deciders:** Trainer
- **Supersedes:** —
- **Tags:** scope, safety, mcp, drafts

---

## Context

The Drafter produces high-quality reply candidates that land in the
Gmail Drafts folder via `mail.create_draft`. The natural next move is
to add `mail.send`, letting the user say "send the draft I just
reviewed". From an engineering standpoint this is one MCP tool wrapper
plus a slash command. From an architectural standpoint it is the
single highest-impact safety boundary in the project.

A wrong draft sitting in Drafts is a UX nit. A wrong message sent to a
real recipient is irreversible — sometimes career-defining,
occasionally legally consequential. Mailroom is a 15-day cohort
project with a junior+senior pair structure; v1's goal is to nail the
read path, the structured plan, the Critic, the hooks, and the audit
chain. Adding a sending path before those are battle-tested would
trade safety for ceremony.

Out of scope: the v2 design of `/send` (a separate ADR when the
feature is added).

## Decision

**v1 ships without a `mail.send` tool.** The MCP server in
`src/mailroom/mcp_server/server.py` exposes `mail.create_draft` and
nothing that sends. The OAuth write scope requested by
`scripts/oauth_setup.py` is `gmail.modify`, **not** `gmail.send` —
even if a participant adds a `mail_send` function, the credential
will refuse it.

Three concrete enforcement points:

1. **No `mail_send` tool registered** in `server.py`. CI lint job
   greps `@server.tool` definitions and fails if it sees one named
   `mail_send` or `mail.send`.
2. **OAuth scope deliberately narrow.** `W_SCOPES` in
   `src/mailroom/mcp_server/auth.py` is exactly
   `["https://www.googleapis.com/auth/gmail.modify"]` — no send
   scope. Adding the scope would require re-consenting and
   re-running `scripts/oauth_setup.py`.
3. **Documented v2 unlock path.** When `/send` is implemented post-
   sprint, it must require the user to type the literal string
   `CONFIRM` plus a one-line justification, both appended to
   `audit.jsonl` (per ADR-0006). The corresponding new ADR
   supersedes this one.

## Consequences

### Positive

- **The blast radius of any v1 bug is bounded.** No bug in the
  Drafter, the Lead, the agent loop, the prompt cache, the Hooks,
  or the MCP server can send mail. The user sends mail.
- **Day 10 pen-test is meaningfully easier.** The 12-attack red-team
  battery does not need a "what if the agent sends a real email"
  scenario; the architecture makes that scenario impossible.
- **Teaches the right architectural reflex.** Cohort participants
  learn that sometimes the best decision is *not to ship a feature*.
  This is a recurring pattern in production AI systems.
- **Demo-friendly.** "It drafts; you click send" is reassuring on
  stage in ways that "it sends, but it's safe, trust the hook" is not.

### Negative

- **The daily-driver story has a manual final step.** Users open
  Gmail, review the draft, click send. We measured this at ~6 s
  per draft on the cohort dry-run. Acceptable; the friction
  *is* the safety property.
- **The architecture cannot be advertised as "fully autonomous".**
  This is an honesty cost; we lean into it ("Mailroom is a
  human-in-the-loop assistant").
- **`/send` is a known v2 demand.** Some pairs will be tempted to
  ship a quick implementation in week 2. The `code-reviewer`
  subagent and the trainer hold the line.

### Neutral

- The naming is intentional: the absent tool would be
  `mail.send`, not `mail.send_draft` or `drafts.send`. Future
  search of the repo for "send" turns up exactly one place
  (this ADR), which is the right answer.

## Alternatives considered

### Alternative A — Ship `mail.send` with a CONFIRM prompt

- **Summary.** Implement send, gated on the user typing `CONFIRM`
  in the slash command argument.
- **Why rejected.** A typed CONFIRM is one extra keystroke compared
  to "click send in Gmail". The marginal automation value is small;
  the risk surface added (a sending-capable token, a sending-capable
  hook gap, a sending-capable agent path) is large. We re-evaluate
  for v2 once the v1 safety story has a track record.

### Alternative B — Ship `mail.send` behind a feature flag, off by default

- **Summary.** Implement the tool, register it conditionally on
  `MAILROOM_ENABLE_SEND=1`.
- **Why rejected.** Feature flags age badly. A flag-default-off path
  rots; participants enabling it for "just one demo" set a precedent.
  The architectural signal of "the tool does not exist" is stronger
  than "the tool exists but is off".

## Implementation notes

- **`src/mailroom/mcp_server/auth.py`:**
  ```python
  W_SCOPES = ["https://www.googleapis.com/auth/gmail.modify"]
  # NOTE: gmail.send is intentionally absent. See ADR-0010.
  ```
- **`src/mailroom/mcp_server/server.py`** has no `@server.tool()`
  named `mail_send`. The lint job in `.github/workflows/lint.yml`
  adds:
  ```yaml
  - name: Forbid mail.send tool
    run: |
      ! grep -n "mail_send\|mail\.send" -- $(git ls-files 'src/**/*.py')
  ```
- **`README.md`** Quickstart explicitly notes that the user manually
  sends from Gmail's UI.
- **The participant pre-read brief** (`02_Mailroom_Participant_PreRead.md`)
  rule #3: "Drafts, never sends. `mail.send` is *deliberately not
  implemented* in v1."

## Open questions

- [ ] What is the v2 unlock checklist? Draft proposal: (a) all 12
      red-team attacks pass on a fresh build; (b) eval CI green for
      30 days; (c) a new ADR supersedes this one with the
      `/send CONFIRM <reason>` flow specified. — owner: Trainer,
      due: post-sprint.

## References

- `src/mailroom/mcp_server/auth.py`
- `src/mailroom/mcp_server/server.py`
- `scripts/oauth_setup.py`
- `.github/workflows/lint.yml`
- `README.md`
- ADR-0001 (two scoped OAuth credentials; the W scope is exactly
  `gmail.modify`)
- ADR-0007 (plan-bound approval tokens; v2 `/send` extends, not
  bypasses)
- ADR-0009 (idempotency keys; v2 `/send` will require them)
- Trainer's master plan, §12 ("Stretch goals"), `/send` with
  confirmation
