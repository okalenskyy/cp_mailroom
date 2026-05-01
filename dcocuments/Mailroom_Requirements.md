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

## What you need before Day 1

### 1. A throwaway Gmail account

Mailroom is destructive to play with. Do **not** point it at your real inbox. Create a brand‑new Gmail account today, named something like `mailroom-<your-first-name>@gmail.com`. Enable 2FA on it. We will seed it with 200 synthetic emails on Day 1.

### 2. Software installed

| Tool | Why | Install |
|---|---|---|
| Python 3.11 or 3.12 | runtime | https://www.python.org or `pyenv` |
| `uv` | venv & dependency manager | `pip install uv` (or `brew install uv`) |
| Git + GitHub CLI | source control | https://cli.github.com |
| Claude Code | the agent IDE | https://claude.com/claude-code |
| Node 20+ | only for some MCP CLI helpers | `nvm install 20` |
| A laptop with ≥ 16 GB RAM | local embeddings work smoothly | (verify yours) |

### 3. API keys

- **Anthropic API key.** Create one at console.anthropic.com. Daily development uses your own key (~$25 in API spend across the 15 days; the trainer covers eval CI separately).
- **Google OAuth client.** Use the trainer's shared OAuth client (instructions on Day 1) — you do **not** need your own Google Cloud project.

### 4. Calendar block

Block 90 minutes per working day for the sprint. The work decomposes badly into 20‑minute chunks; uninterrupted time matters.

### 5. Your partner

You will be paired with one other participant (junior + senior). Pairs are announced at kickoff. Plan to do a 5‑minute pair sync at the start of every working day, camera on.

---

## What to read (≈ 3 hours total)

Skim these in order. You don't need to memorise anything; you need vocabulary.

1. **Anthropic engineering — *Building effective agents*.** The conceptual frame for the entire sprint.
2. **Claude Agent SDK quickstart** at docs.claude.com. The agent loop, tool use, streaming.
3. **Model Context Protocol overview** at modelcontextprotocol.io, plus the **Python SDK quickstart**.
4. **Claude Code docs — Skills, Hooks, slash commands.** The surfaces you will author.

If you finish early, skim the "Choose Gmail API scopes" page in Google's docs — we will be very deliberate about scopes on Day 1.

---

## What Day 1 looks like

A 3‑hour live session. Agenda:

- 45 min: Architect Essential primer (agent loop, sub‑agents, MCP, Skills, Hooks, evals, the *Approval Workflow*).
- 30 min: Pair formation, environment smoke test.
- 90 min: First MCP tool implemented; first agent call returns real data from your seeded throwaway inbox.
- 15 min: Day‑2 planning.

You leave Day 1 with `python -m mailroom hello` printing how many unread messages are in your throwaway Gmail.

---

## How the 15 days break down at a glance

- **Days 1–5 — Foundations & the read path.** OAuth, MCP read tools, the Sorter sub‑agent, the first Skill.
- **Days 6–10 — Sub‑agents, drafts, hooks.** Researcher, Drafter, Critic, Scheduler. The *Approval Workflow* gates every write. Day 10 ends with a structured pen‑test.
- **Days 11–15 — Evals, observability, polish, showcase.** A 50‑item golden set, a CI eval gate, prompt caching, the demo recording, the Day 15 showcase.

---

## What you walk away with

- a public GitHub repo that demonstrates production AI architecture end‑to‑end,
- a 90‑second demo recording suitable for LinkedIn or your portfolio,
- a clean revision path for the Architect Essential exam — *every objective is something you personally implemented*, and
- a tool you will actually use every morning to triage your real inbox (after Day 15, by pointing v2 of your build at it).

---

## House rules

- **Throwaway Gmail only** during the sprint. We will gently audit.
- **No solo merges to `main`.** Every PR has a co‑author from your pair.
- **Drafts, never sends.** `mail.send` is *deliberately not implemented* in v1. You unlock it post‑sprint.
- **ADRs for the three hardest decisions.** One page each, in `docs/adr/`. Format is in the scaffold.
- **Daily 3‑line standup** in `#mailroom-cohort`: `done / next / blocked`.
- **Office hours** Tue and Thu evenings, 30 minutes each, bookable via Calendly. Bring code, not slides.

---

## Day‑0 checklist — tick all of these before kickoff

- [ ] Throwaway Gmail created, 2FA enabled
- [ ] Python 3.11+ installed and `python --version` works
- [ ] `uv --version` works
- [ ] `claude --version` works
- [ ] `gh auth status` shows you logged in
- [ ] Anthropic API key minted and exported as `ANTHROPIC_API_KEY` in your shell
- [ ] You can `git clone` from the cohort GitHub org
- [ ] You have skimmed the four required reading items
- [ ] 90 minutes/day blocked on your calendar for the sprint
- [ ] Joined `#mailroom-cohort`

If any item is unticked at kickoff, ping the trainer in `#blocker-of-the-day` *before* the session starts so we can fix it without slowing the room.

---

*See you on Day 1. Read the trainer's master plan if you want the full architecture upfront — otherwise it will unfold in the right order through the sprint.*
