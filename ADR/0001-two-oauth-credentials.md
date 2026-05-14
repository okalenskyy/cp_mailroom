# ADR-0001: Use two scoped OAuth credentials

- **Status:** accepted
- **Date:** 2026-04-29
- **Authors:** Trainer (sample ADR)
- **Deciders:** Trainer
- **Supersedes:** —
- **Tags:** security, oauth, mcp

---

## Context

Mailroom needs to read and mutate Gmail. The simplest approach is one OAuth
client requesting `gmail.modify`, used by both read and write tools — this
is also the path the official Google quickstarts demonstrate.

Mailroom runs locally on a participant's laptop and stores tokens in
`.secrets/`. The threat model includes accidental token leakage in commits
or screenshots, and the agent loop reading attacker-controllable email
content. We want a meaningful blast-radius reduction without making Day 1
materially harder.

Out of scope: token rotation, hardware-backed key storage, and the v2
`mail.send` flow.

## Decision

We mint two OAuth tokens. The read-only token requests `gmail.readonly` and
`calendar.readonly`. The write token requests `gmail.modify`. The MCP server
holds both `Credentials` objects and routes read tools through the read
client and write tools through the write client. Refresh logic is
centralised in `src/mailroom/mcp_server/auth.py` so the two paths cannot
drift.

## Consequences

### Positive

- A token-stealing exploit on the read path cannot mutate state.
- The two-credential split is a teaching moment for OAuth scope hygiene —
  participants experience why "least privilege" is not just slogans.
- The split makes future hardening (e.g., short-lived write tokens
  re-minted per `--apply`) a small change rather than a refactor.

### Negative

- Two consent screens at first run; participants who skip the second
  silently lose write functionality and only notice on Day 7.
- Refresh logic exists in two places. We centralise in `auth.py` to avoid
  drift, but it remains a maintenance surface.
- Some Gmail SDK examples assume a single credential; participants
  occasionally copy-paste code that breaks the split.

### Neutral

- Both tokens land in `.secrets/` and are `.gitignored`. The pre-commit
  hook (`make lint`) catches accidental commits of `token_*.json`.

## Alternatives considered

### Alternative A — Single `gmail.modify` token, route by code convention

- **Summary.** One credential, with a code-level rule that "read functions
  must not call write methods".
- **Why rejected.** The attacker doesn't read our convention; they read
  our token. A code-only split provides no defence against agent
  misbehaviour or token exfiltration.

### Alternative B — One token now, split later if we feel like it

- **Summary.** Defer the split to v2.
- **Why rejected.** "Defer security" is a recurring engineering anti-pattern;
  the cost to add the split on Day 1 is ~30 lines and one extra consent
  screen. The cost to retrofit later is much higher because every write
  tool's signature changes.

## Implementation notes

- `RO_SCOPES` and `W_SCOPES` are defined in
  `src/mailroom/mcp_server/auth.py`.
- `scripts/oauth_setup.py` runs two consecutive `InstalledAppFlow` consents
  and writes `token_ro.json` + `token_w.json` under `.secrets/`.
- `mcp.json` injects both token paths via env vars; the MCP server reads
  them at start-up.

## Open questions

- [ ] Do we want short-lived write tokens re-minted per `/triage --apply`?
      — owner: Trainer, due: post-sprint (v2 candidate ADR).

## References

- `src/mailroom/mcp_server/auth.py`
- `scripts/oauth_setup.py`
- Google docs — *Choose Gmail API scopes*
- Trainer's master plan, §3.3 ("The MCP server")
