# ADR-0008: Use Haiku as the eval judge, with checked-in prompt and temp=0

- **Status:** accepted
- **Date:** 2026-04-29
- **Authors:** Trainer (sample ADR)
- **Deciders:** Trainer
- **Supersedes:** —
- **Tags:** evals, cost, observability

---

## Context

The eval harness scores the Drafter's output against a four-axis rubric
(`evals/rubric.md`) on every PR via `make evals`. The agent under test
uses Sonnet (the project's default for production paths). The eval
*judge*, however, is a separate model we choose explicitly.

A naive choice — "use Sonnet to judge Sonnet" — costs more than it
buys. Sonnet's incremental judgment quality over a smaller model on a
well-anchored rubric is small; the cost difference is large. With ~50
items × 3 runs per PR × ~12 active PRs per week, the eval workload is
the cohort's biggest budget line.

Out of scope: the rubric content (in `evals/rubric.md`) and the
golden-set composition (ADR-0010 area, but not strictly an ADR).

## Decision

We use **`claude-haiku-4-5-20251001`** as the eval judge, configured at
`temperature=0`, with the judge prompt checked in at
`evals/judge_prompt.md`. Sonnet remains the default agent model
(`MAILROOM_MODEL=claude-sonnet-4-6`); Haiku is set via
`MAILROOM_JUDGE_MODEL=claude-haiku-4-5-20251001`.

Concrete rules:

1. **Model identifiers are pinned.** Strings live in `.env.example`
   and `src/mailroom/settings.py`; tests fail if either drifts.
2. **`temperature=0`** on every judge call. No top-p tuning, no
   sampling. Repeatability beats stylistic quality for evals.
3. **The judge prompt is a file**, not a string in code. Diffs to
   `evals/judge_prompt.md` are subject to PR review; an approved
   prompt change requires a re-baseline of `evals/.baseline.json`.
4. **The judge is blind to the gold answer.** It scores the model's
   output against the rubric anchors only. The runner must not pass
   `gold_facts` into the judge prompt.
5. **The runner caches judgments.** Given identical (rubric, prompt,
   model output) tuples, the judge runs once; subsequent calls hit
   `evals/.judge_cache.jsonl` (gitignored). This makes regression
   diffs cheap.

## Consequences

### Positive

- **Eval cost per PR drops by ~6×.** Measured: 50 items × 3 runs at
  Sonnet judging ~ $0.36 per PR; the same with Haiku judging
  ~ $0.06. Across the 15-day sprint this is the difference between
  $25 and $5 per pair on evals alone.
- **`temperature=0` makes regressions interpretable.** A score
  change comes from a model output change, not a judge sample
  change.
- **The checked-in prompt is part of the contract.** Reviewers can
  ask "what changed in the rubric?" and answer it from the diff.

### Negative

- **Haiku judges are slightly noisier on the "tone match" axis.**
  We measured 0.08 SD vs Sonnet's 0.04 SD on the same 50-item set.
  We accept the noise; it is below the regression-gate threshold
  (5 points = 0.15 on a 0–3 scale).
- **The judge cache must be invalidated when the rubric changes.**
  The runner detects rubric-file changes via SHA-256 and clears the
  cache; this works but introduces a subtle dependency.
- **Two model identifiers to update at every Anthropic release.**
  We ship a `make models-up` target that suggests the latest IDs
  (and warns if they have changed since the last `evals/.baseline`).

### Neutral

- The judge sees the *redacted* version of the model output if the
  output passes through the audit logger first. For evals this does
  not matter; the model output is judged before audit.

## Alternatives considered

### Alternative A — Sonnet as judge

- **Summary.** Use the same model the system under test uses.
- **Why rejected.** 6× the cost for a measured 0.04 SD reduction in
  noise — not a worthwhile trade for a project running 12 PRs per
  week. The bigger model also tends to "self-flatter" on outputs
  similar to its own style; using a different model family avoids
  the issue (this is well-documented in the LLM-as-judge literature).

### Alternative B — Deterministic rubric (regex/keyword scoring)

- **Summary.** Replace the LLM judge with deterministic checks
  (e.g., "draft must contain a thread-cited fact").
- **Why rejected.** Tone match and overpromise detection cannot be
  done with regex without massive false-positive rates. We do ship
  *some* deterministic checks (cited thread IDs must exist;
  recipients must come from the input thread) — these run alongside
  the LLM judge, not instead of it.

## Implementation notes

- **`evals/judge.py`:**
  ```python
  from anthropic import AsyncAnthropic
  from mailroom.settings import settings

  client = AsyncAnthropic(api_key=settings.anthropic_api_key)

  async def judge_draft(thread: dict, draft: dict) -> RubricResult:
      prompt = (Path(__file__).parent / "judge_prompt.md").read_text()
      msg = await client.messages.create(
          model=settings.judge_model,
          temperature=0,
          max_tokens=600,
          messages=[{
              "role": "user",
              "content": _format_input(prompt, thread, draft),
          }],
      )
      return _parse(msg.content[0].text)
  ```
- **`evals/judge_prompt.md`** (D11 deliverable) contains the rubric
  anchors and the JSON output schema the judge must produce.
- **`evals/runner.py`** caches via:
  ```python
  cache_key = sha256(prompt_text + json.dumps(thread, sort_keys=True)
                                + json.dumps(draft, sort_keys=True)).hexdigest()
  ```
- **`evals/.baseline.json`** is checked in. PRs that move the
  baseline must include a "Why" line in the description.

## Open questions

- [ ] Should we add a second judge (different model family) and
      report inter-rater agreement? — owner: Trainer, due: post-sprint.

## References

- `evals/judge.py`
- `evals/judge_prompt.md` (D11 deliverable)
- `evals/runner.py`
- `evals/rubric.md`
- `evals/.baseline.json`
- `src/mailroom/settings.py`
- `.env.example`
- ADR-0006 (audit log; not a dependency, related)
- Trainer's master plan, §6 (Day 12 — Eval harness and CI gate)
