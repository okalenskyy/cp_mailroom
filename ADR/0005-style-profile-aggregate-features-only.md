# ADR-0005: Style profile contains aggregate features only — never raw text

- **Status:** accepted
- **Date:** 2026-04-29
- **Authors:** Trainer (sample ADR)
- **Deciders:** Trainer
- **Supersedes:** —
- **Tags:** privacy, drafter, style, security

---

## Context

The Drafter must reply in the user's voice. The natural way to teach it
that voice is to feed it the user's last 200 sent messages. Doing so,
however, would put a substantial corpus of personal correspondence into
every Drafter prompt — sent, in plaintext, to the model.

The cohort uses throwaway Gmail accounts during the sprint, so the
training-day risk is limited. But Mailroom is meant to graduate to the
participant's *real* inbox post-sprint. We must build the Drafter so it
*cannot* leak the user's prose, even on a real inbox, even if the
Drafter prompt is captured.

Out of scope: the throwaway-Gmail policy itself (operational rule,
not architecture) and the choice of local paraphrasing model (an
implementation detail).

## Decision

We compute a **privacy-by-construction style profile** containing
*aggregate features only*, persisted to `~/.mailroom/style.json` (path
configurable via `MAILROOM_STYLE_PROFILE`). The Drafter consumes the
profile via the `style.profile_summary` MCP tool; raw sent-mail content
is **never** returned by that tool and never enters the Drafter's
context.

`style.json` schema, exact:

```json
{
  "version": "1",
  "computed_at": "<iso ts>",
  "n_messages": 200,
  "sentence_length": {"mean": 14.3, "p90": 28},
  "salutations":     {"Hi": 0.62, "Hey": 0.18, "Dear": 0.04, "<none>": 0.16},
  "signoffs":        {"Thanks": 0.55, "Best": 0.21, "Cheers": 0.12, "<none>": 0.12},
  "contraction_rate": 0.42,
  "formality": 0.38,
  "top_bigrams_hashed": ["sha256:abcd...", "sha256:..."],
  "paraphrased_examples": [
    "Quick check-in on the contract status",
    "Confirming the meeting moved to Tuesday",
    "Noted, thanks for the heads up"
  ]
}
```

Rules:

1. **No raw sentences.** No `examples` field with verbatim text. The
   `paraphrased_examples` field contains short, locally-paraphrased
   templates produced by `src/mailroom/style/profile.py` using either
   a small local model or a deterministic templating function.
2. **Bigrams as hashes.** `top_bigrams_hashed` carries SHA-256 prefixes
   of the user's distinctive bigrams. The Drafter never sees the
   bigrams in plaintext; they exist for distinctiveness checking
   (e.g., "did the candidate draft echo any of the user's distinctive
   bigrams" — yes/no, no readable input).
3. **No subjects, no recipients, no domains.** Even aggregate
   recipient frequencies are excluded; they leak professional context.
4. **No regeneration in cloud.** `style.profile.compute()` runs locally
   on the user's machine. The function never makes a network call.
5. **Versioned shape.** Bumping `version` is a new ADR.

## Consequences

### Positive

- **The Drafter cannot leak prose it never received.** The strongest
  privacy guarantee is "data we don't have can't be exfiltrated".
- **The profile is cacheable.** `style.json` updates weekly; prompt
  caching on the Drafter system prompt is highly effective.
- **The privacy story is testable.** `tests/test_style_no_leak.py`
  plants a poison sentence in fixture sent-mail and asserts it does
  not appear in `style.json`. The test runs on every PR (`make test`).

### Negative

- **The Drafter's voice match is coarser than a fine-tuned model
  would produce.** We measured a tone-match rubric mean of 2.4/3 with
  the aggregate profile vs. an estimated 2.7/3 with a fine-tune
  alternative. We accept the gap; the privacy property is more
  valuable than the marginal quality gain.
- **Local paraphrasing requires either a small local model or a
  templated fallback.** The fallback is shipped (`profile.py`) but it
  produces less natural examples; participants can wire a local
  Ollama or llama.cpp instance if they want better paraphrases.
- **Two-stage caching.** The profile cache (weekly) and the prompt
  cache (per-session) interact; a stale `style.json` after a profile
  bump is a real but rare bug. We surface profile age in the
  observability dashboard.

### Neutral

- The profile is per-user, not per-thread or per-recipient. We do not
  attempt to mimic the user's voice differently when writing to
  different correspondents.

## Alternatives considered

### Alternative A — Fine-tune a per-user model

- **Summary.** Fine-tune Haiku (or a third-party model) on the user's
  sent mail.
- **Why rejected.** The fine-tune's training run *ingests the raw
  prose*. Even if we never call the fine-tune in plaintext later, the
  training data crosses an API boundary we do not control.

### Alternative B — RAG over recent sent mail with redaction

- **Summary.** At Drafter time, retrieve top-k similar sent emails,
  redact PII, and include them as in-context examples.
- **Why rejected.** Redaction is structurally weaker than
  non-collection. A redaction regex bug becomes a leak; an
  *aggregate-only* design has no leak surface to begin with.

## Implementation notes

- `src/mailroom/style/profile.py`:
  ```python
  def compute(*, n: int = 200) -> dict: ...
  def load_summary() -> dict: ...
  ```
  `compute()` reads sent mail via the local read-only credential
  (ADR-0001), produces the aggregate features in-memory, paraphrases
  examples locally, writes `style.json`, and returns the dict. Raw
  sentences are not bound to any name that escapes the function.
- `src/mailroom/mcp_server/server.py` exposes `style_profile_summary`
  which calls `load_summary()`. Note: this MCP tool returns the JSON
  *as-is*. Adding any field that contains raw text would break
  ADR-0005 — the `code-reviewer` subagent and `test_style_no_leak.py`
  both enforce this.
- `tests/test_style_no_leak.py`:
  ```python
  POISON = "PURPLE_HIPPO_AT_DAWN_MAILROOM_PRIVACY_CANARY"
  # planted into a fixture sent-mail message; assert that POISON
  # does not appear in any string field of style.json after compute().
  ```

## Open questions

- [ ] Should we ship a tiny local model (e.g., a 1B-parameter quant)
      for paraphrasing, or stay with the templating fallback? —
      owner: Trainer, due: post-sprint.

## References

- `src/mailroom/style/profile.py`
- `src/mailroom/mcp_server/server.py`
- `.claude/skills/style-mirror/SKILL.md`
- `tests/test_style_no_leak.py`
- ADR-0002 (Pydantic hand-offs)
- ADR-0006 (audit log redaction policy is consistent with this ADR)
- Trainer's master plan, §3.4 ("The style profile (privacy-first design)")
