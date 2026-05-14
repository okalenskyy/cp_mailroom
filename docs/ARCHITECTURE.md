# Architecture

A Lead Agent built on the Claude Agent SDK orchestrates six specialist
sub-agents that share a custom Python MCP server. Two scoped OAuth
credentials separate read from write. The signature pattern is the
**Approval Workflow** — every mutation is gated by a hook that requires a
fresh, expirable, plan-bound approval token.

## Sub-agents

| Agent      | Tools                                  | Output              |
| ---------- | --------------------------------------- | ------------------- |
| Sorter     | mail.list_unread, contacts.lookup       | TriagePlan          |
| Researcher | mail.get_thread, mail.search, kb.search | ResearchPacket      |
| Drafter    | style.profile_summary                   | DraftCandidate      |
| Scheduler  | cal.busy, cal.propose_slot              | MeetingProposal?    |
| Critic     | (audit reader; no write tools)          | Verdict             |
| Briefer    | (no tools)                              | markdown            |

## Approval Workflow (sequence)

```
User → /triage  ─→  Lead → Sorter → TriagePlan saved
                         → Briefer renders the checklist
User → /triage --apply  ─→ each write tool carries approval_token=<plan_id>
                         → PreToolUse hook validates {plan_exists, fresh, ids∈plan, recipient_allowed, snooze_ok, idempotent}
                         → write executes
                         → PostToolUse hook appends redacted, hash-chained record
                         → Critic reads audit, emits a receipt
```

## See also

- `docs/architecture.mmd` — the Mermaid diagram source.
