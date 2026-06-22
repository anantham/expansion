# Affordances

Hand-curated tools across `~/Documents/Ongoing Local/` projects that solve a real recurring problem. Agents: grep here before reinventing. To add an entry, get user approval first.

> **Canonical home:** `~/Documents/Ongoing Local/expansion-skills/AFFORDANCES.md` (the `expansion-skills` repo, git remote `anantham/expansion` — the `expansion` plugin's source; ships with the plugin, and `expansion:handover` references it). A relative symlink `~/Documents/Ongoing Local/AFFORDANCES.md → expansion-skills/AFFORDANCES.md` exposes it at the top-level path — but **Syncthing does NOT sync symlinks, so recreate it per-device**: `ln -s expansion-skills/AFFORDANCES.md "~/Documents/Ongoing Local/AFFORDANCES.md"`. Edit the canonical file; agents can also just grep `~/Documents/Ongoing Local/` for `AFFORDANCES.md`.

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

## Multi-account browser IMAGE generation + diegetic lyric keying (Lyra)

The image-gen counterpart to the entry above: drive Gemini / ChatGPT / Grok image models in their web UIs (free, on the user's subs), with account rotation + failover, plus a chroma-key recipe that turns model-rendered text into word-synced glowing light (lyrics made of the scene's own light, not captions-on-top). The richest, most battle-tested browser-image-gen in the tree.

- **Pattern home:** `~/Documents/Ongoing Local/Lyra/pipeline/providers/` — `browser.py` (Gemini `gemini-3-pro-image`: `generate(prompt, out, ref=)`, `_chat` prose, `_locate_gemini` vision word-boxes), `grok_browser.py` (Grok Imagine: aspect-lock, all-variations capture, `GrokLimit`/`GrokContent` rate/refusal signals), + a ChatGPT lane (Story 9:16, copyright-refusal split+retry). Uses the SAME GEO/pramana Playwright runner + `~/.atlas/browser-state/<site>/` sessions as the entry above — it's a superset for images.
- **Multi-account rotation/failover:** N profiles per provider (`GEMINI_PROFILES`, `GROK_ACCOUNTS`); round-robin by gen-count, mark-account-down on limit/refusal, fall through providers. Pick the lane via `LYRA_IMG_PROVIDER` = `gemini_browser` | `grok_browser` | `browser` (chatgpt) | `openrouter` (paid API).
- **Diegetic keying (the novel bit):** prompt the model to render the words AS a magenta light source IN the scene → `pipeline/render_engine.html` chroma-keys the magenta, recolors it, and word-syncs a glow (auto-detects word bands; no detect boxes needed); `pipeline/capture_frames.py` captures frames headless via the runner.
- **Per-subject world:** `pipeline/stages/research.py` derives a `world.json` (theme/style/palette/lens/grain) from research so each subject renders in ITS own setting — dodges the generic-stock-image trap.
- **Gotchas (hard-won):** Gemini needs explicit "Create image" mode selection or it answers in TEXT; run HEADED (`LYRA_BROWSER_HEADLESS=0`); poll ≤280s + click "Answer now" to skip its thinking-mode; chat apps won't emit strict JSON — use `claude -p` / an API for structured fields (cf. union-store `free-llm-fallback`); if the text-LLM brain (agy / claude -p / OpenRouter) is down, field-gen silently falls onto a Gemini text-chat per line = a slow "bombardment."
- **When to use:** generating images at volume on free frontier subs; any diegetic-text-in-image / lyric-as-light effect.
- **When NOT to use:** one-off images (the paid `openrouter` lane is simpler); strict-JSON needs (`claude -p` / an API).

Last verified: 2026-06-22 (Lyra — House of Memories reel, all lanes)

---

## Scheduled recurring tasks (per-user macOS)

Plugin-based runner for periodic local jobs — backups, vault maintenance, metric collection, journal carryover, queue workers. One LaunchAgent + one orchestrator + many operation plugins. Config-driven schedules (`interval:2h`, `daily:11:30`).

- **Pattern home:** `~/Documents/Ongoing Local/TemporalCoordination/scheduled_tasks/` — `runner.py` + `operations/<task>.py` + `config.yaml`. ADR-001 documents the architecture.
- **Adding a new task:** drop a file in `operations/`, inherit `ScheduledOperation`, register in `config.yaml` with a schedule. Runner picks it up.
- **In flight examples:** Claude/Gemini/IndrasNet backups; Obsidian vault gardening; journal carryover (daily Obsidian task migration); disk/RAM/clutter metrics; retrieval index refresh; media captioning queue worker.
- **When to use:** anything that should run periodically on this machine without spinning up a dedicated daemon.
- **When NOT to use:** event-driven work (use a queue), or anything that needs to run on a different machine.

Last verified: 2026-05-18
