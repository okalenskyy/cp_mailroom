# Mailroom — Your AI Inbox Triage Assistant

A privacy-conscious, human-in-the-loop AI inbox triage assistant built on the
Claude Agent SDK and a custom Python MCP server. Built as the community project
for the Claude Certified Architect Essential certification.

---

## What you can do with it

- **`/morning-inbox`** — a 1-page brief: who needs you today, what you owe people,
  what you're waiting on, ghost meetings buried in threads.
- **`/triage`** — proposes a structured action plan (classify, archive, label,
  snooze). Nothing is applied until you run `/triage --apply <id>`.
- **`/draft <thread_id>`** — produces a reply in your own writing voice. The
  draft lands in Gmail's drafts folder. **Never auto-sent.**
- **`/follow-up`** — surfaces emails you sent that nobody replied to.

## Architecture in one paragraph

A Lead Agent built on the Claude Agent SDK orchestrates six specialist
sub-agents (Sorter, Researcher, Drafter, Scheduler, Critic, Briefer) that share
a custom Python MCP server. The MCP server wraps Gmail and Google Calendar
behind two OAuth credentials (read-only and write) so a token compromise on
the read side cannot mutate state. The signature pattern is the **Approval
Workflow**: every mutating tool call is gated by a `PreToolUse` hook that
requires a fresh user-approved plan, and every call is appended to a
PII-redacted JSONL audit log by `PostToolUse`.

See `docs/ARCHITECTURE.md` for the full diagram.

## Quickstart (10 minutes)

> **One-time rename after extracting the scaffold archive:**
> the bundle ships dotfile-prefixed folders as `dotclaude/` and `dotgithub/`.
> Rename them once: `mv dotclaude .claude && mv dotgithub .github`.

```bash
git clone <your-fork>
cd mailroom-starter
mv dotclaude .claude && mv dotgithub .github   # only after first extract
uv venv && source .venv/bin/activate
uv pip install -e ".[dev]"

cp .env.example .env  # add your Anthropic API key

# 1. mint two OAuth tokens (read-only + write) for your throwaway Gmail
python scripts/oauth_setup.py

# 2. seed the throwaway inbox with 200 synthetic labelled emails
python scripts/seed_throwaway_gmail.py

# 3. smoke test
make hello   # should print "You have N unread"
```

Then open the project in Claude Code and try `/morning-inbox`.

## Repo tour

```
.claude/        # mcp.json, hooks, commands, skills — everything Claude Code reads
src/mailroom/   # the Python package: agents, MCP server, schemas, audit log
evals/          # golden set, runner, judge, rubric
scripts/        # OAuth bootstrap, throwaway-inbox seeder
docs/           # architecture diagram, ADRs, exam-objective map
tests/          # smoke + hooks + critic
```

## House rules

- **Throwaway Gmail only.** Never point this at a real inbox during the sprint.
- **No solo merges.** Every PR has a co-author from your pair.
- **Drafts, never sends.** `mail.send` is deliberately absent from v1.
- **ADRs for the three hardest decisions.** Template in `docs/ADR-template.md`.

## License

MIT — see `LICENSE`.
