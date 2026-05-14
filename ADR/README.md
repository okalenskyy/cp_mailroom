# Mailroom — Architectural Decision Records

> *Ten ADRs ship with the project. They capture the decisions Mailroom
> itself makes, in enough detail to drive Claude Code development. New
> decisions you make during the sprint go into additional ADRs alongside
> these.*

## How to use this folder

Each ADR is one page, immutable once `accepted`, and authoritative for
the decision it captures. Code that enforces an ADR carries a
`# see ADR-NNNN` comment so the connection survives refactors.

To write a new ADR for a decision you and your pair just made:

1. Invoke the `adr-coach` Claude Code subagent (`.claude/agents/adr-coach.md`).
2. It will pick the next available `NNNN`, ask you four questions,
   and produce a draft using `docs/ADR-template.md`.
3. Open it as a draft PR. Status starts at `proposed`. Both pair
   members must comment; trainer reviews within 48 hours.
4. Status flips to `accepted` on merge. The Decision section becomes
   immutable.

To reverse a decision: never edit the accepted ADR. Write a new one
with `Supersedes: ADR-NNNN`, and update the old ADR's status to
`superseded by ADR-MMMM`. Both files stay in the repo.

## The shipped ADRs

| #    | Title                                              | Tags                                  |
| ---- | -------------------------------------------------- | ------------------------------------- |
| 0001 | Use two scoped OAuth credentials                   | security, oauth, mcp                  |
| 0002 | Hand-offs between agents are Pydantic v2 models    | schemas, agents, contracts, mcp       |
| 0003 | The Critic sub-agent has no write tools            | agents, safety, drafts, audit         |
| 0004 | Snippet-first response shape on mail tools         | mcp, gmail, performance, context      |
| 0005 | Style profile contains aggregate features only     | privacy, drafter, style, security     |
| 0006 | Audit log is SHA-256 hash-chained JSONL on disk    | audit, safety, privacy, observability |
| 0007 | Plan-bound approval tokens for every mutation      | safety, hooks, mcp, approval-workflow |
| 0008 | Use Haiku as the eval judge                        | evals, cost, observability            |
| 0009 | Idempotency keys are required on every write call  | safety, mcp, hooks, retries           |
| 0010 | mail.send is intentionally not implemented in v1   | scope, safety, mcp, drafts            |

## Reading notes

Following three ADRs carry the project's signature ideas:

- **ADR-0007** — the *Approval Workflow* (the project's signature
  pattern; everything else feeds into it).
- **ADR-0005** — privacy by construction (the style profile is the
  best-told story of "data we don't have can't be exfiltrated").
- **ADR-0010** — knowing when not to ship (the engineering judgement
  to defer `mail.send`).

## Dependency graph (informal)

```
        0001 ─────────────┐
                          │
        0002 ────► 0003   │
            └────► 0004   │
                          ▼
        0005           0007 ◄──── 0009
                          │
                          ▼
        0006 ◄────────── 0010
                 ▲
                 │
        0003 ────┘
```

Read 0001 → 0002 → (0003, 0004) → 0005 → 0006 → 0007 → 0009 → 0010 → 0008
to walk the architecture in causal order.

## Where the ADRs are referenced

- `CLAUDE.md` (hard rules pull from these ADRs).
- `docs/ARCHITECTURE.md` (pointers, no duplication).
- Inline `# see ADR-NNNN` comments in code where the rule is enforced.

## When in doubt

If a decision you face is *not covered* by any ADR here, write a new
one. The threshold is the three-condition test from
`docs/HOW-TO-WRITE-ADRS.md`:

1. Hard to reverse later.
2. Future reader will ask "why?".
3. A reasonable engineer could have decided otherwise.

If all three are true, write the ADR. If any one is false, just ship.
