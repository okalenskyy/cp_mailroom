# Welcome to Mailroom

---

## What you are about to build

**Mailroom** is a privacy‑conscious, human‑in‑the‑loop AI inbox triage assistant. Over 15 working days, with a partner, you will build a public‑GitHub‑grade Python project that:

- triages your unread mail every morning,
- drafts replies in your own writing voice,
- never sends, archives, or deletes anything without your explicit approval, and
- demonstrates every Architect Essential exam objective in working code.

By the end you will own a portfolio repo with: a custom MCP server, six specialist sub‑agents, five Skills, hooks that gate every mutation, a 50‑item eval harness, and a 90‑second demo recording.

---

## Workflows

/morning-inbox — a 1‑page brief at the top of every day: who needs you today, what you owe people, what you're waiting on, and "ghost meetings" — proposed times buried in threads with no calendar event yet.
/triage — walks the unread inbox and produces a structured action plan: classify (urgent / action‑required / FYI / newsletter / social), extract action items, propose archive / label / snooze, surface "must reply today". Nothing is applied until the user runs /triage --apply, which goes through a confirmation Hook.
/draft <email-id> — produces a reply grounded in (a) the full thread, (b) your sent‑mail style profile, (c) your personal knowledge (notes, calendar, prior conversations with that contact). The draft lands in Gmail's drafts folder; never auto‑sent.
/follow-up (emails you sent that nobody replied to in N days), 
/unsubscribe-audit (newsletters you never open — generates a checklist, never auto‑clicks), 
/calendar-conflicts (proposed times in threads vs. your calendar).

----

## What you need

### 1. A throwaway Gmail account

Mailroom is destructive to play with. Do **not** point it at your real inbox. Create a brand‑new Gmail account today, named something like `mailroom-<your-first-name>@gmail.com`. Enable 2FA on it. We will seed it with 200 synthetic emails on Day 1.

### 2. Software installed

| Tool | Why | Install |
|---|---|---|
| Python 3.11 or 3.12 | runtime | https://www.python.org or `pyenv` |
| `uv` | venv & dependency manager | `pip install uv` (or `brew install uv`) |
| Git + GitHub CLI | source control | https://cli.github.com |
| Claude Code | the agent IDE | https://claude.com/claude-code |
| A laptop with ≥ 16 GB RAM | local embeddings work smoothly | (verify yours) |

### 3. API keys

- **Anthropic API key.** Create one at console.anthropic.com. Daily development uses your own key (~$25 in API spend across the 15 days; the trainer covers eval CI separately).
- **Google OAuth client.** Create one.


## What to read (≈ 3 hours total)

1. **Anthropic engineering — *Building effective agents*.** The conceptual frame for the entire sprint.
2. **Claude Agent SDK quickstart** at docs.claude.com. The agent loop, tool use, streaming.
3. **Model Context Protocol overview** at modelcontextprotocol.io, plus the **Python SDK quickstart**.
4. **Claude Code docs — Skills, Hooks, slash commands.** The surfaces you will author.

---

## House rules

- **Throwaway Gmail only** during the sprint. We will gently audit.
- **No solo merges to `main`.** Every PR has a co‑author from your pair.
- **Drafts, never sends.** `mail.send` is *deliberately not implemented* in v1. You unlock it post‑sprint.
- **ADRs (Architecture Decision Record) for the three hardest decisions.** One page each, in `docs/adr/`. 
- **Daily 3‑line standup** in `#mailroom-cohort`: `done / next / blocked`.
- **Office hours** Tue and Thu evenings, 30 minutes each, bookable via Calendly. Bring code, not slides.

---

## Prerequisites checklist — tick all of these before start

- [ ] Gmail created, 2FA enabled
- [ ] Python 3.11+ installed and `python --version` works
- [ ] `uv --version` works
- [ ] `claude --version` works
- [ ] `gh auth status` shows you logged in
- [ ] Anthropic API key minted and exported as `ANTHROPIC_API_KEY` in your shell
- [ ] You can `git clone` from the cohort GitHub
- [ ] You have skimmed the four required reading items
- [ ] 90 minutes/day blocked on your calendar for the sprint

If any item is unticked, ping Oleg.


