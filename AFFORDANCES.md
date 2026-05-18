# Affordances

Hand-curated tools across `~/Documents/Ongoing Local/` projects that solve a real recurring problem. Agents: grep here before reinventing. To add an entry, get user approval first.

> **Canonical home:** `~/Documents/Ongoing Local/expansion/AFFORDANCES.md`. A symlink at `~/Documents/Ongoing Local/AFFORDANCES.md` exposes it at the user-facing path that `expansion:handover` references. Edit the canonical file; the symlink picks it up.

---

## Browser automation against logged-in frontier-model accounts

Drive Gemini / ChatGPT / Claude.ai / Grok in their real web UIs using a Playwright persistent context. Inherits the user's authenticated session, so account features (Deep Research, image generation, custom instructions, memory) are available.

- **Pattern home:** `~/Documents/Ongoing Local/GEO/runner/atlas_runner/browser_chat.py` — `BrowserChatProvider` base class + subclasses for ChatGPT, Gemini, Grok. Anti-automation flags (`channel="chrome"`, `--disable-blink-features=AutomationControlled`) handled.
- **Persistent state:** `~/.atlas/browser-state/<site>/` — one Chromium user-data-dir per site. Treat like `~/.ssh/`.
- **Concrete example:** `~/Documents/Ongoing Local/LexiconForge/scripts/gemini_research.py` — minimal CLI that takes a prompt file, writes Gemini's response to disk. ~250 lines, no GEO dependency.
- **When to use:** anything needing the user's account on a frontier model — Deep Research runs, image generation, paid-tier features, custom instructions side effects.
- **When NOT to use:** unauthenticated read-only scraping (WebFetch / direct Playwright is lighter).

Last verified: 2026-05-18

---

## Scheduled recurring tasks (per-user macOS)

Plugin-based runner for periodic local jobs — backups, vault maintenance, metric collection, journal carryover, queue workers. One LaunchAgent + one orchestrator + many operation plugins. Config-driven schedules (`interval:2h`, `daily:11:30`).

- **Pattern home:** `~/Documents/Ongoing Local/TemporalCoordination/scheduled_tasks/` — `runner.py` + `operations/<task>.py` + `config.yaml`. ADR-001 documents the architecture.
- **Adding a new task:** drop a file in `operations/`, inherit `ScheduledOperation`, register in `config.yaml` with a schedule. Runner picks it up.
- **In flight examples:** Claude/Gemini/IndrasNet backups; Obsidian vault gardening; journal carryover (daily Obsidian task migration); disk/RAM/clutter metrics; retrieval index refresh; media captioning queue worker.
- **When to use:** anything that should run periodically on this machine without spinning up a dedicated daemon.
- **When NOT to use:** event-driven work (use a queue), or anything that needs to run on a different machine.

Last verified: 2026-05-18
