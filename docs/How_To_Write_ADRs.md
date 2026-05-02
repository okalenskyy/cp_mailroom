# How to write Architectural Decision Records (ADRs)

---

## 1. What an ADR is

An Architectural Decision Record is a short, dated, *immutable* document that
captures one architectural decision and the reasoning behind it. It is written
at the moment the decision is made, by the people who made it, for the people
who will inherit the codebase six months from now.

Three properties matter:

- **Short.** One page. If you cannot fit it on one page, the decision is not
  ready.
- **Dated.** The date the decision is *accepted*, not the date you started
  writing.
- **Immutable.** You never edit the Decision section of an accepted ADR. To
  change direction, write a new ADR that supersedes the old one.

ADRs are a writing discipline as much as an architecture artifact. The act of
writing forces the implicit reasoning out of the senior's head and onto a page
the junior can argue with.

## 2. What an ADR is not

| Not this                                 | Because…                                                    |
| ---------------------------------------- | ----------------------------------------------------------- |
| A design document                        | Designs describe systems; ADRs describe *choices*.          |
| A runbook or operations guide            | Operations live in `docs/runbooks/`, not in `docs/adr/`.    |
| A wishlist or roadmap                    | Roadmaps describe intent; ADRs describe a decision *now*.   |
| A "we discussed it" note                 | If no decision was made, do not write an ADR.               |
| A novel                                  | The reader has 60 seconds.                                  |

## 3. When to write one

Write an ADR when the decision satisfies *all three* of:

1. **It will be hard to reverse later.** Either because code depends on it,
   data has been migrated, or a contract exists with the outside world.
2. **A future reader will ask "why?"** If the answer to "why is it this way"
   would be obvious from reading the code, no ADR is needed.
3. **A reasonable engineer could have decided otherwise.** If there was no
   real alternative, you didn't make a decision; you took a default.

In practice, on Mailroom, this triggers for things like:

- the choice to mint two scoped OAuth credentials instead of one
- the introduction of the Critic sub-agent as a quality gate
- pinning Pydantic v2 across every cross-agent payload
- the eval rubric anchors and the regression-gate threshold
- the audit log format and hash-chaining scheme

Counter-examples — these *do not* warrant ADRs:

- choosing a variable name
- adding a unit test
- bumping a dependency by a patch version
- a refactor that does not change architecture

If you're unsure, ask in the pair sync. As a rough heuristic: each pair on
Mailroom should produce **3 to 6 ADRs** across the 15 days. Fewer than three
means you are letting decisions slide; more than six means you are
over-formalising small choices.

## 4. The lifecycle of an ADR

```
draft  ──►  proposed  ──►  accepted  ──►  superseded
                              │
                              └──►  deprecated
```

- **draft** — local only, not yet shared. Most ADRs spend an hour here.
- **proposed** — opened as a draft PR. Both pair members must comment;
  trainer reviews within 48 hours.
- **accepted** — merged to `main`. Decision section is now immutable.
- **superseded by ADR-MMMM** — replaced by a newer ADR. The new ADR's
  `Supersedes:` field names the old one. The old file stays in the repo.
- **deprecated** — the decision is no longer relevant (e.g., the entire
  feature was removed). No replacement exists.

You never delete an ADR. The history *is* the artefact.

## 5. The discipline of each section

The template is at `04_Mailroom_ADR_Template.md` (or `docs/ADR-template.md` in
the scaffold). The headers are not optional — they exist so a reader can scan
twelve ADRs in five minutes.

### Title

A complete decision in five to nine words, starting with a verb in the
imperative *or* naming the chosen approach. The title is the most-read line
in the document; spend two minutes on it.

| Style                          | Example                                          |
| ------------------------------ | ------------------------------------------------ |
| Imperative verb                | "Use two scoped OAuth credentials"               |
| Names the chosen approach      | "Plan-bound approval tokens"                     |
| Stack pin                      | "Pin Pydantic v2 for cross-agent payloads"       |
| **Avoid**                      | "Authentication", "Schemas", "Notes on tooling"  |
| **Avoid (questions)**          | "How should we authenticate?"                    |

### Context

Three things, in 1–3 short paragraphs:

1. **What changed** that makes this decision necessary now.
2. **The forces** — requirements, constraints, non-negotiables.
3. **The boundary** — what is *out of scope* for this ADR.

Cite specifics: file paths, issue numbers, prior ADR numbers, exam-objective
references. The reader six months from now is the audience.

A good test: if you removed the Decision section, the Context alone should
let an experienced engineer guess two or three reasonable options.

### Decision

One paragraph. Active voice. Present tense. First-person plural ("we").

The first sentence must be a complete decision the reader can act on.

If the decision has knobs (TTLs, thresholds, model identifiers, file paths),
pin the values here. "We use Sonnet" is not a decision; "We use
`claude-sonnet-4-6` for agent loops and `claude-haiku-4-5-20251001` for the
eval judge" is.

What this section is *not*: the rationale. The rationale lives implicitly in
Context and explicitly in Alternatives Considered.

### Consequences

Three sub-buckets — *positive*, *negative*, *neutral* — and the negative
bucket must not be empty. Every architectural choice has costs; an ADR with
no negatives is a sales pitch, not a decision record.

Each bullet is a *concrete second-order effect*, not a restatement of the
decision.

| Restating the decision (bad)            | Real consequence (good)                                                |
| --------------------------------------- | ---------------------------------------------------------------------- |
| "We have two OAuth credentials."        | "First-run setup adds 30 s for the second consent screen."             |
| "We use Pydantic v2."                   | "All sub-agents now share a `model_dump_json` codepath; no shadow JSON." |
| "We have hooks."                        | "Every write tool call now incurs a sub-process spawn (~12 ms p50)."   |

### Alternatives considered

At least two. For each, name the alternative, give one or two evaluation
criteria, and explain *why it was rejected*. "Worse" is not a reason; say
specifically what is worse, and ideally cite numbers.

If you cannot list a real alternative you seriously evaluated, the decision
was not a decision — it was a default. Either find one (talk to your pair,
ask in office hours) or scrap the ADR.

### Implementation notes (optional)

Use only when the *how* needs nailing down for the decision to be concrete:
file paths to touch, migration steps, feature flags, rollout order, who pages
whom if it breaks. Skip this section if it would be empty.

### Open questions (optional)

Things you deliberately did not decide. Each item should have an owner and a
target date or follow-up ADR number. More than three items here means the
ADR is not ready to move from "proposed" to "accepted".

### References

PR numbers, issue numbers, prior ADRs, external posts, exam-objective IDs.
If this ADR is referenced from code (good!), include the file paths.

## 6. A worked example

Below is a complete ADR for one of the real Mailroom decisions. Use it as a
shape reference.

---

### ADR-0007: Plan-bound approval tokens for every mutation

- **Status:** accepted
- **Date:** 2026-05-08
- **Authors:** John Doe, Johanna Doe
- **Deciders:** Johanna Doe
- **Supersedes:** —
- **Tags:** safety, hooks, mcp

#### Context

Mailroom mutates the user's inbox: it labels, archives, snoozes, and creates
drafts. The cost of a wrong action is high (lost mail, sent draft to wrong
recipient, contract email archived past a deadline). The agent loop is
non-deterministic and the email body is attacker-controllable, so the
*system*, not the agent, has to enforce safety.

By Day 8 we have a working `/triage` that produces a structured plan, a
Critic that reviews drafts, and write tools in the MCP server. We need a
mechanism that ties any individual write call to the plan the user
explicitly approved, in a way the `PreToolUse` hook can verify deterministically.

Out of scope for this ADR: the audit log format (ADR-0008), the
identifier-redaction policy (ADR-0009), and v2's `mail.send` flow.

#### Decision

We require every mutating MCP tool call to carry an `approval_token` and an
`idempotency_key`. The `approval_token` is the 12-character hex id of a plan
file under `${MAILROOM_PLANS_DIR}/<id>.json` produced by `/triage` (or any
other read-only sub-agent that emits a plan). The `PreToolUse` hook
(`.claude/hooks/pre_tool_use.py`) validates the token on every call:

1. the plan file exists,
2. it was created within the last `MAILROOM_PLAN_TTL_MINUTES` (default 30),
3. the call's `message_id` / `thread_id` is one of the plan's target IDs,
4. for `mail.create_draft`, the recipient passes the sensitive-allowlist
   check unless `unlock_sensitive=True`,
5. for `mail.snooze`, `days <= MAILROOM_MAX_SNOOZE_DAYS` (default 30),
6. the `idempotency_key` is unique within the plan.

Plans are produced read-only and saved before the user sees them. The user
moves from plan to action by re-invoking `/triage --apply <plan_id>`, which
threads the same plan_id into every write call.

#### Consequences

##### Positive

- Mutations cannot occur without an explicit, recent, plan-bound user step.
- The hook's checks are pure functions of the plan file and the tool args;
  they are easy to unit test (see `tests/test_hooks.py`).
- Idempotency keys make retries safe; a duplicated approval cannot
  double-apply a label or archive a thread twice.
- The plan file is the audit trail's anchor: every `audit.jsonl` line
  references the plan id, so a human can reconstruct what was approved.

##### Negative

- Adds ~12 ms p50 per write call (sub-process spawn for the hook). The eval
  harness now runs slightly hotter — accepted as the price of safety.
- Plans expire after 30 minutes; users who walk away from a triage and
  return later get a friendly "plan expired" message instead of completion.
  This is *intended* but will surface as a UX nit for ~10 % of sessions.
- The hook now contains real logic; bugs in the hook can lock out
  legitimate writes. We mitigate with the 12-attack red-team battery on
  Day 10.

##### Neutral

- The plan format becomes a public contract between the Sorter and the hook
  (see `src/mailroom/schemas/triage_plan.py`). Future schema changes
  require a migration path or a compatibility window.

#### Alternatives considered

##### Alternative A — Confirmation-only via a slash command

- **Summary.** `/triage --apply` would simply re-invoke the write tools
  with no token; "I am running --apply" is treated as the consent.
- **Why rejected.** The Lead Agent could call `--apply` itself in a future
  refactor, defeating the gate. Defence-in-depth requires the proof of
  consent to live in the *call*, not in the invocation context.

##### Alternative B — Per-call human confirmation prompt

- **Summary.** The hook prompts the user (yes/no) on every individual
  write call.
- **Why rejected.** A 50-action triage produces 50 prompts. We measured 11
  s of friction per prompt; at 50 actions that is ~9 minutes for what
  should be a 30-second operation. Users would learn to mash "yes" and the
  prompt becomes ceremonial.

##### Alternative C — JWT-signed approval tokens with a per-pair private key

- **Summary.** Use HMAC-signed tokens carrying claims (target IDs,
  expiry, recipient allowlist) so the hook does not need to read disk.
- **Why rejected.** Adds a key-management problem we do not need on a
  single-machine sandbox project. Plan files on disk give us the same
  guarantees with no key rotation work, plus they double as the audit
  anchor (a JWT does not).

#### Implementation notes

- New env vars: `MAILROOM_PLAN_TTL_MINUTES`, `MAILROOM_MAX_SNOOZE_DAYS`,
  `MAILROOM_SENSITIVE_RECIPIENTS` (see `.env.example`).
- The `Sorter` calls `TriagePlan.persist(settings.plans_dir)` after
  classification.
- All four write tools in `src/mailroom/mcp_server/gmail_tools.py` accept
  `approval_token` and `idempotency_key` keyword args.
- The hook lives at `.claude/hooks/pre_tool_use.py` and is wired in
  `.claude/mcp.json`.

#### Open questions

- [ ] Should plans persist across reboots, or be tied to session? — owner:
      Pair 03, due: Day 13 (see ADR-0010 candidate).

#### References

- `src/mailroom/schemas/triage_plan.py`
- `.claude/hooks/pre_tool_use.py`
- `tests/test_hooks.py`
- ADR-0001 (two scoped OAuth credentials)
- Trainer's master plan, §3.5 ("The Approval Workflow")
- PR #142, PR #149

---

That entire ADR fits on a single page when rendered. Notice what it does:

- the title is a decision in seven words,
- the Context names exactly *which* decision is being made and what is out of
  scope,
- the Decision pins the values (`30` minutes, `30` days, the 12-char hex id
  format),
- Consequences include a number (~12 ms) and an honest UX cost,
- Alternatives are real options the pair seriously considered, not strawmen,
- References point to code, tests, prior ADRs, and the PR.

## 7. Common pitfalls

| Pitfall                                                         | Why it bites                                                | Fix                                                          |
| --------------------------------------------------------------- | ----------------------------------------------------------- | ------------------------------------------------------------ |
| Title is a noun, not a decision                                 | Reader cannot act from the title alone                      | Rewrite as imperative or named approach                      |
| Context describes the system, not the *forces*                  | Reader reconstructs facts but not why the choice was needed | Name what changed; name the constraints                      |
| Decision section restates options                               | The reader cannot tell what was decided                     | First sentence must be the decision, full stop               |
| Negative consequences section is empty                          | ADR reads like a sales pitch; trust evaporates              | Find the real cost; if there isn't one, the choice was trivial and the ADR shouldn't exist |
| Alternatives are obvious strawmen                               | Reader doesn't trust the rationale                          | Name alternatives a thoughtful colleague would actually raise |
| ADR is two pages                                                | Won't be read; won't be cross-referenced                    | Cut. Move detail into linked code, runbooks, or follow-up ADRs |
| ADR talks about implementation steps, not the decision          | Becomes a runbook, ages badly                               | Move steps to *Implementation notes*; keep the decision crisp |
| Editing an accepted ADR's Decision section                      | Breaks the immutability contract                            | Write a new ADR; supersede the old one                       |
| Multiple decisions in one ADR                                   | Unfindable later; cannot supersede half of an ADR           | Split into one ADR per decision                              |
| ADR opens months after the code shipped                         | Memory has decayed; rationale is fiction                    | Write the ADR *while* the code is in review                  |


Detailed information on ARD: https://github.com/okalenskyy/architecture-decision-record#what-is-an-architecture-decision-record
---

