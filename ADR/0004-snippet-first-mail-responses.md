# ADR-0004: Snippet-first response shape on mail tools

- **Status:** accepted
- **Date:** 2026-04-29
- **Authors:** Trainer (sample ADR)
- **Deciders:** Trainer
- **Supersedes:** —
- **Tags:** mcp, gmail, performance, context-engineering

---

## Context

The Sorter sub-agent classifies up to 100 unread messages per `/triage`
invocation. Returning full message bodies for every one of them is
catastrophic for both cost and context window: a single thread can be
30–60 KB of text, and 100 threads would dominate any reasonable
context budget.

At the same time, the Drafter and the Researcher genuinely need full
bodies for the threads they actually act on (typically 1–5 per
`/triage`). The MCP tool surface must support both demands without
forcing every consumer to decide for itself how much data to fetch.

Out of scope: which tool the Researcher uses (already decided —
`mail.get_thread`) and the structured fields exposed at the audit
boundary (ADR-0006).

## Decision

The MCP server splits mail reads into two response shapes:

1. **Snippet shape** — `mail.list_unread`, `mail.search`, and
   `contacts.lookup` return only IDs, headers, and a snippet capped at
   **200 characters**. No bodies, no quoted-reply chains, no
   attachments.
2. **Full shape** — `mail.get_thread` returns the full thread including
   bodies. It is the *only* tool that can produce body text, and is
   therefore the gate the Researcher must go through.

Concrete schemas (Pydantic v2, per ADR-0002), defined in
`src/mailroom/schemas/mail.py`:

```python
class MailSnippet(BaseModel):
    message_id: str
    thread_id: str
    from_addr: str
    to_addrs: list[str]
    subject: str
    snippet: str = Field(..., max_length=200)
    received_iso: str
    has_attachments: bool
    label_ids: list[str]

class MailThread(BaseModel):
    thread_id: str
    messages: list[MailMessage]    # MailMessage carries body
```

Implementation rules:

1. `mail.list_unread(max_results: int = 50)` returns
   `list[MailSnippet]`. The default cap is **50**; the absolute max is
   **100**.
2. `mail.get_thread(thread_id: str)` returns `MailThread`. The thread
   is truncated at the **last 8 messages** by default; older messages
   are fetched only when the Researcher explicitly asks via
   `mail.search`.
3. `mail.search(query: str, max_results: int = 25)` returns
   `list[MailSnippet]`. Same 200-character snippet cap.
4. **Snippet content is never reconstructed from a body.** Snippets
   come from Gmail's `snippet` field directly so they cost no tokens
   to compute.
5. **Body access is auditable.** Any call to `get_thread` lands in
   `audit.jsonl` with the `thread_id` and a redacted `from_addrs`
   list, but never the body itself.

## Consequences

### Positive

- **`/triage` over 100 unread messages costs ~12 KB of tokens for the
  Sorter**, not ~2 MB. We measured an 85% reduction in context size
  versus the body-on-everything alternative.
- **Body access is auditable in one place.** Reviewing
  `grep mail.get_thread audit.jsonl` shows every body read, ever.
- **Sub-agent boundaries map cleanly to tool boundaries.** Sorter sees
  snippets; Researcher sees threads; Drafter sees the Researcher's
  packet, not raw threads.

### Negative

- **The Sorter occasionally misclassifies because the 200-character
  snippet truncated the deciding sentence.** We measured this at <2%
  on the 50-item golden set. We accept the cost; the alternative is
  body access for the Sorter, which costs more than the misclassifications.
- **Two response shapes mean two fixture types** for evals. The
  fixture builder generates both.
- **The 200-char limit and the 50/100 max_results values are tuning
  parameters.** They can drift if a maintainer changes one without
  updating the other. We treat them as a versioned contract: any change
  is a new ADR.

### Neutral

- Subject lines are not redacted in snippets. They are part of the
  classification signal; the audit log still redacts addresses but
  retains subjects. (Subjects are plaintext on the wire anyway.)

## Alternatives considered

### Alternative A — Body-on-everything

- **Summary.** Return full bodies on `list_unread`; let the agent
  decide what to keep.
- **Why rejected.** Token cost dominated everything else in the
  D5 dry-run; a 100-thread `/triage` on real-looking mail used 1.9 MB
  of input tokens. At that scale, prompt caching cannot recover the
  cost because the input changes every run.

### Alternative B — Configurable snippet length per call

- **Summary.** Let the caller request 200, 500, or 1000 characters.
- **Why rejected.** A configuration knob would migrate from "200 by
  default" to "1000 because the Sorter sometimes wants more" within a
  week. The fixed contract forces explicit `get_thread` calls when
  more is genuinely needed, which lands in the audit log. Knobs without
  audit trails erode safety guarantees.

## Implementation notes

- `src/mailroom/mcp_server/gmail_tools.py`:
  - `list_unread` calls `users().messages().list(userId="me",
    q="is:unread", maxResults=max_results).execute()` then a batched
    `users().messages().get(format="metadata", metadataHeaders=
    ["From","To","Subject","Date"])` for each ID. Snippet comes from
    Gmail's response.
  - `get_thread` calls `users().threads().get(format="full")` and
    parses MIME parts. Truncation to 8 messages happens *after* fetch
    (Gmail bills the full thread regardless).
- `tests/test_schemas.py` enforces `snippet ≤ 200` chars and exact
  field membership of `MailSnippet`.
- `evals/fixtures/` ships pre-captured responses; the eval runner
  never hits the live API.

## Open questions

- [ ] Should attachments be surfaced in `MailSnippet` beyond
      `has_attachments: bool`? Right now there is no tool that exposes
      attachment metadata. — owner: Trainer, due: post-sprint.

## References

- `src/mailroom/schemas/mail.py` (D3 deliverable)
- `src/mailroom/mcp_server/gmail_tools.py`
- `src/mailroom/mcp_server/server.py`
- `evals/fixtures/`
- ADR-0002 (Pydantic hand-offs)
- ADR-0006 (audit log)
- Trainer's master plan, §3.3 ("The MCP server")
