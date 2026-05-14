# ADR-0002: Hand-offs between agents are Pydantic v2 models, not free text

- **Status:** accepted
- **Date:** 2026-04-29
- **Authors:** Trainer (sample ADR)
- **Deciders:** Trainer
- **Supersedes:** —
- **Tags:** schemas, agents, contracts, mcp

---

## Context

Mailroom orchestrates a Lead Agent and six specialist sub-agents (Sorter,
Researcher, Drafter, Scheduler, Critic, Briefer). Every workflow involves
data crossing at least one agent boundary: `list_unread → Sorter → TriagePlan
→ Briefer`, or `get_thread → Researcher → ResearchPacket → Drafter →
DraftCandidate → Critic`.

The default LLM-orchestration pattern is to pass natural-language strings
between sub-agents. We have rejected that pattern. With six agents and a
strict safety story (the `PreToolUse` hook references field-level facts
like `target_id` and `idempotency_key`), free-text hand-offs would force
every consumer to re-parse the producer's prose, accumulate parsing bugs,
and silently drift over time.

Out of scope: serialization at the *MCP boundary* (where JSON is
unavoidable), persistence formats, and the `audit.jsonl` line shape (see
ADR-0006).

## Decision

We require every cross-agent hand-off to be a **Pydantic v2 `BaseModel`
subclass** defined under `src/mailroom/schemas/`. Agent boundary functions
accept and return models, not `dict`s. JSON is used only at the MCP
boundary; inside the Python process, models flow as Python objects.

Three concrete schemas ship in the scaffold:

- `schemas/triage_plan.py` — `TriagePlan`, `ClassifiedMessage`, `Action`.
- `schemas/research_packet.py` — `ResearchPacket`, `PriorThread`, `KBNote`,
  `SenderProfile`.
- `schemas/draft_candidate.py` — `DraftCandidate`.

Rules that govern every schema in this folder:

1. **One file per top-level model.** A model can compose smaller models
   from the same file but must not pull from another schema file at the
   top level.
2. **Field-level validators inline.** Use `Field(..., ge=0, le=1)` and
   `model_validator(mode="after")` rather than separate validator
   functions.
3. **Stable contracts.** Renaming or retyping a field is a breaking
   change; every such change requires a new ADR or a `Supersedes:`
   relationship.
4. **No `dict[str, Any]` fields.** If a payload genuinely needs an
   open-ended bag, model it as `notes: list[Note]` with a `Note` model.
5. **`model_config = ConfigDict(extra="forbid")`** on every top-level
   model so unknown keys fail loudly.
6. **JSON serialization only at the MCP edge.** Inside Python, agents
   pass instances; only the MCP server's tool wrappers call
   `model_dump_json()`.

## Consequences

### Positive

- **Producers cannot accidentally drop fields** — `extra="forbid"` plus
  required-field defaults make missing data a typed runtime error rather
  than a silent `None`.
- **Hooks can rely on field semantics.** `PreToolUse` reads
  `args["approval_token"]` knowing it is a non-empty string of length 12,
  because that is enforced upstream by the Pydantic constructor.
- **Schemas double as documentation.** Reading `schemas/triage_plan.py`
  is the fastest way to understand the data flowing between Sorter and
  Briefer.
- **Free unit tests.** `tests/test_schemas.py` exercises the constraints
  in isolation; failures are precise (`ValidationError` with field path).

### Negative

- **30–50 lines of boilerplate per schema.** Junior pairs occasionally
  experience this as friction; we lean on the `code-reviewer` subagent
  to gently refuse plain-dict PRs.
- **Schema changes ripple.** Adding a field to `ClassifiedMessage` may
  require touching the Sorter, the Briefer, and one test. This cost is
  acceptable; the alternative (free-text drift) is worse but slower to
  surface.
- **Pydantic v2 is a peer dependency.** Pinned in `pyproject.toml` at
  `>=2.7`. Any future migration to v3 requires a Day-13-style triage
  window.

### Neutral

- Schemas live under `src/mailroom/schemas/` and are imported absolutely
  (`from mailroom.schemas.triage_plan import TriagePlan`). Relative
  imports from inside `agents/` are forbidden.

## Alternatives considered

### Alternative A — Free-text hand-offs with regex extraction

- **Summary.** Sub-agents emit prose; consumers regex out the fields
  they need.
- **Why rejected.** We measured a 14% extraction-failure rate on a 50-
  example dry-run when the producer prompt was unchanged across runs.
  That rate is the *floor*, not the ceiling — every prompt revision
  re-introduces drift.

### Alternative B — TypedDict / dataclass instead of Pydantic

- **Summary.** Use `typing.TypedDict` (or stdlib `dataclass`) for
  hand-offs.
- **Why rejected.** Neither validates at construction time. The hooks'
  safety checks rely on validation having already happened upstream;
  pushing it to runtime branches makes the hook the validator of last
  resort, which violates the single-responsibility split between
  agents (correctness) and hooks (safety).

## Implementation notes

- Every agent function signature uses model types:
  ```python
  async def sort(messages: list[dict]) -> TriagePlan: ...
  async def gather(thread_id: str) -> ResearchPacket: ...
  async def compose(*, thread: dict, packet: ResearchPacket,
                    proposal: dict | None = None) -> DraftCandidate: ...
  ```
- The MCP tool wrappers in `src/mailroom/mcp_server/server.py` accept
  primitive args and return JSON-serialisable structures; they call
  `.model_dump()` on any model that crosses the boundary.
- `tests/test_schemas.py` enforces every constraint under `extra="forbid"`
  and the `confidence ∈ [0, 1]` bound.

## Open questions

- [ ] Should we adopt `pydantic-settings` for `settings.py`? Currently
      a hand-rolled `BaseModel`. — owner: Trainer, due: post-sprint.

## References

- `src/mailroom/schemas/triage_plan.py`
- `src/mailroom/schemas/research_packet.py`
- `src/mailroom/schemas/draft_candidate.py`
- `tests/test_schemas.py`
- ADR-0006 (audit log JSONL line shape)
- Trainer's master plan, §3.2 ("The six sub-agents")
