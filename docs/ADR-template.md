# ADR-NNNN: <a short, decision-shaped title>

<!--
NAMING:
  File:  docs/adr/NNNN-kebab-case-title.md     (NNNN = next four-digit number)
  Title: starts with a verb in the imperative, OR names the chosen approach.
         GOOD:  "Use two scoped OAuth credentials"
         BAD :  "OAuth notes" or "Authentication"

LENGTH: one page. If you can't say it in one page, the decision isn't ready.
VOICE:  active, present tense, first-person plural ("we"), specific.

See `docs/HOW-TO-WRITE-ADRS.md` for principles, pitfalls, and a worked example.
-->

- **Status:** proposed | accepted | deprecated | superseded by ADR-MMMM
- **Date:** YYYY-MM-DD                   <!-- the day status flipped to "accepted" -->
- **Authors:** <pair name(s) — at least two when accepted>
- **Deciders:** <who must agree before status flips to "accepted">
- **Supersedes:** <ADR-XXXX, if this replaces an earlier decision>
- **Tags:** <comma-separated, e.g. mcp, security, oauth, evals>

---

## Context

<!--
What changed in the world that forces a decision now?
Name the *forces*: requirements, constraints, non-negotiables, deadlines.
Cite specifics — file paths, issue numbers, prior ADRs.
1–3 short paragraphs.
-->

<!-- Replace this paragraph with the real context. -->

## Decision

<!--
One paragraph. Active voice. Present tense. First-person plural ("we").
The first sentence MUST be a complete decision the reader can act on.
Pin any concrete values (TTLs, model identifiers, file paths).
-->

<!-- Replace this paragraph with the real decision. -->

## Consequences

<!--
Three sub-buckets. The "Negative" bucket must not be empty — every
architectural choice has costs. Each bullet is a concrete second-order
effect, NOT a restatement of the decision.
-->

### Positive
- <effect>

### Negative
- <effect>

### Neutral
- <effect>

## Alternatives considered

<!--
At least two real alternatives. For each: name it, give one or two evaluation
criteria, and explain *why it was rejected* — specifically. "Worse" is not a
reason; say what is worse, with numbers if you have them.
-->

### Alternative A — <name>

- **Summary.** <one sentence>
- **Why rejected.** <specific, evidence-bound reason>

### Alternative B — <name>

- **Summary.** <one sentence>
- **Why rejected.** <specific, evidence-bound reason>

## Implementation notes

<!-- OPTIONAL. Skip the section entirely if it would be empty. -->

- <step or note>

## Open questions

<!-- OPTIONAL. Each item gets an owner and a target date or follow-up ADR. -->

- [ ] <question> — owner: <name>, due: YYYY-MM-DD

## References

- <PR, issue, prior ADR, code path, exam-objective ID>

---

<!--
REVIEW RULES:
  1. ADR opens as "proposed" in a draft PR.
  2. Both pair members sign off in the PR.
  3. Trainer reviews within 48 hours.
  4. Status flips to "accepted" on merge to main.
  5. Never edit an accepted ADR's Decision section. To change direction,
     write a new ADR with `Supersedes: ADR-NNNN` and flip the old one to
     "superseded by ADR-MMMM".
-->
