---
name: Handover
description: Graceful context transfer before session end or compaction. Commits work, documents pending threads, captures learnings, and prepares the next instance to continue seamlessly.
when_to_use: when user says "handover", "wrap up", "closing session", or when context is approaching 90% capacity and compaction is imminent
version: 1.6.1
changelog:
  1.6.1 (2026-05-16): patch — announce-at-start now includes the version string for self-identification on invocation. Users had no easy way to verify which skill version was loaded vs cached. Reads the `version:` field above; replace `<version>` literally with that value.
  1.6.0 (2026-05-16): mandate verbatim user-quote capture — 5 surgical edits (Phase 0 triage matrix row, Phase 2 EXHAUSTIVENESS CHECKLIST row, Phase 4 template required section, anti-patterns table row, final checklist row). The conversation JSONL is local-only and /compact paraphrases lossily; verbatim quotes in the handover .md are the only durable grounding for "what the user wanted."
  1.5.0 (2026-05-14): exhaustiveness checklist + name 'Silent Omission via Conciseness' anti-pattern. Added Phase 2 EXHAUSTIVENESS CHECKLIST (9 binding rows), explicit Carry-forward-from-prior-handover scan-for item, named anti-pattern, split Phase 4 template's Session Summary (narrative) from Pending Threads (enumeration). Plus 'Operator Cleanup' section in template.
  1.4.0 (2026-05-14): Phase 0 — Triage by Marginal Value. Identify what THIS dying context can produce that future sessions cannot; propose captures to user before executing.
  1.3.0 (2026-05-13): related-skills cross-ref (meta-update + skill-update), ADR section in template, overwrite guidance for existing HANDOVER.md, push-auth rule.
  1.2.0 (2026-05-13): require exhaustive deferred-thread scan — read prior HANDOVER, ADRs, roadmap, architecture docs, audit reports, README "candidate" lists. Don't only document this-session TODOs.
  1.1.0 (2026-05-13): empirical patches from LexiconForge load-test.
  1.0.0 (2026-02-11): initial — graceful context transfer before compaction; PreCompact hook integration.
---

# Handover

## Purpose

You are about to lose this context. Whether due to compaction, session end, or context limit — everything you've learned, every thread you're tracking, every insight you've gained will vanish unless you capture it NOW.

This skill ensures continuity across instances. The next Claude picking up this conversation should be able to continue as if no context was lost.

**Announce at start:** "Running handover **v\<version\>** — committing work, documenting threads, and capturing learnings for the next instance." Replace `<version>` literally with the value from this skill's frontmatter `version:` field above (currently `1.6.1`) so the user can verify which skill version is actually loaded.

## Related Skills

Handover focuses on session-state preservation. Two adjacent skills handle downstream concerns:

- **`/meta-update` (or `/mu`)** — Updates `~/.claude/CLAUDE.md` with cross-project protocol learnings extracted from this session. Run AFTER handover when you've noticed corrections, new patterns, or protocol gaps.
- **`/expansion:skill-update`** — Updates THIS or any other skill based on friction encountered while using it. Run when the skill's instructions didn't quite cover your situation.

Typical end-of-session flow: `/handover` → `/mu` → `/expansion:skill-update` (only the first is always relevant; the latter two are conditional).

## When to Trigger

| Trigger | Context |
|---------|---------|
| Manual: `/handover` | User explicitly requests wrap-up |
| Manual: "wrap up", "closing session" | Natural language equivalents |
| Auto (hook): 90% context capacity | Pre-compaction preservation |
| Manual: Before switching tasks | Context pivot coming |

## The Phases

### Phase 0: Triage by Marginal Value

**Goal:** Before running the mechanical checklist, identify what THIS
dying context can produce that future sessions cannot.

Conversation logs (JSONL) are local — they don't survive into next
session. The reasoning trajectory in your head — *why* decisions were
made, *what was considered and rejected*, *which patterns emerged* —
vanishes hardest. Some of that synthesis genuinely cannot be reproduced
by a fresh agent reading the diff.

**Three triage questions:**

1. **What can I produce *only* with the session-specific reasoning
   trace I still have in head?**
   (Principles distilled from the work, patterns spotted, decisions
   considered-and-rejected, rationale narratives, project-memory
   entries, the *why* behind specific commits)

2. **Of those, which are high-leverage** — generalize beyond this
   session, inform multiple future decisions, capture an insight
   nobody else has yet, or compound across sessions?

3. **What's worth the marginal context-spend?**
   Context is finite and dropping fast. Spend it on captures that
   will be expensive or impossible to reproduce later; defer anything
   mechanical that next session can do from clear instructions.

**Triage matrix:**

| Do NOW (this dying context) | Defer to next session |
|---|---|
| Principles + anti-patterns ratified by lived experience | Routine commits, branch cleanups |
| Patterns / methods discovered during work | Code the next agent can write from instructions |
| Rich handover narrative with the *why* | Single-file refactors with clear specs |
| Project / cross-project memory entries | Continuation of the work you're handing over |
| Decision rationale that won't survive diff alone | Anything blocked on user input |
| **Verbatim user quotes** with timestamps — directives in the user's own words; the JSONL is local-only and the /compact summary paraphrases lossily | Lossy paraphrases / "user wanted X" summarizations |

**Anti-patterns in this phase:**
- Continuing the work you're handing over (that's what handover ends)
- Speculative captures ahead of demonstrated need (Phantom Consumer)
- Captures that need more from the user than they can give right now

**Propose your captures to the user before executing** so they can
prune or redirect. The user often has better marginal-value judgment
than the exhausted-context model running on fumes. A brief proposal
("Here's what I think is worth doing with remaining context, what
should I drop?") respects both their attention and the residual
context budget.

After triage and user approval, proceed with Phases 1-4 — those are
the mechanical scaffold. **Phase 0 is what makes the handover
signal-dense rather than merely complete.**

### Phase 1: Commit Checkpoint

**Goal:** Get all work safely committed. Nothing uncommitted survives compaction.

**Steps:**

1. **Check git status**
   ```bash
   git status
   git diff --stat
   ```

2. **Scope check — separate YOUR changes from ambient state.**

   Before committing anything, identify which dirty files are from THIS
   session vs which were already dirty when you started. Multi-agent
   repos and human-co-edited repos always have ambient dirty state.

   Useful checks:
   - `git stash list` — anything you stashed earlier in the session?
   - Compare `git status` output to your tool-call history (Edit / Write
     calls) — files you didn't touch are not yours to commit.
   - When uncertain, leave the file untouched and note it in the
     handover doc under "Ambient dirty state (not committed)".

   Only commit files you affirmatively edited this session. Other
   agents, IDE plugins, or the user may have in-progress work in the
   same checkout. Wrongly committing their state is worse than leaving
   it dirty.

3. **If your uncommitted changes exist:**
   - Group related changes into logical commits
   - Use conventional commit format
   - Don't batch unrelated changes
   - Each commit should be atomic and reviewable

4. **DO NOT push by default.** Pushing requires **explicit authorization
   from the user for THIS session**. Acceptable signals:
   - User said "push" or "push to remote" earlier in the session
   - The project's CLAUDE.md / AGENTS.md explicitly allows autopush
   - User invoked handover with "push" in the args (e.g. `/handover push`)

   If any of those:
   ```bash
   git log --oneline origin/main..HEAD
   git push
   ```
   Verify with `git log origin/main..HEAD` (should show 0 commits after).

   Otherwise: list unpushed commits in the handover doc and explicitly
   note "PUSHED: no — awaits user authorization." That's a normal,
   expected end-state for a session.

5. **Document commit summary:**
   ```
   COMMITS THIS SESSION:
   - <hash> <message>
   - <hash> <message>
   PUSHED: yes | no — awaits user authorization | N/A
   AMBIENT DIRTY (not committed):
   - <file> — <why I left it: not mine, user WIP, etc.>
   ```

**If work is incomplete and shouldn't be committed:**
- Stash with descriptive message: `git stash push -m "WIP: <description>"`
- Or create WIP commit: `git commit -m "WIP: <description> [skip ci]"`
- Document what's incomplete and why

### Phase 2: Thread Inventory

**Goal:** Document **every open thread** the next instance might pick up — not just session-local TODOs. A handover doc that only captures "things we discussed today" leaves the next instance blind to anything the project has been carrying for weeks. Long-lived projects accumulate deferred state; surface it.

**Scan for (in this order):**

1. **Explicit TODOs from this session**
   - User requests not yet completed
   - Errors encountered but not resolved
   - Features partially implemented

2. **Implicit threads from this session**
   - Investigations started but not concluded
   - Patterns noticed but not addressed
   - Questions raised but not answered

3. **Blocked threads**
   - Waiting on user input
   - Waiting on external service
   - Waiting on human decision

4. **Background tasks**
   - Running processes (check with `tasklist` or background task IDs)
   - Scheduled operations pending

5. **Documented deferred state from prior sessions** — the bulk of a long-running project's pending work. Read these before declaring the inventory complete:
   - **Prior `docs/HANDOVER.md`** (if it exists) — anything in its "Deferred" section that hasn't been resolved is still deferred.
   - **`docs/adr/*.md`** — ADR headers often list "deferred sections" (e.g., "§F2.6 Aperture controls — not built"). These are commitments the project has acknowledged but not delivered.
   - **`docs/roadmap.md`** — phase items still listed as "not started" or "deferred."
   - **`docs/architecture/*.md`** — "open code-debt items" and "what's deferred" sections.
   - **Prior audit reports** (e.g., from `doc-audit` skill) — P2/P3 findings that weren't acted on.
   - **README candidates lists** (`docs/proofs/README.md`, `docs/modules/README.md`) — "candidates worth writing" stubs.
   - **`canvas.md`-style "open questions" sections** in any architecture doc.

6. **Explicit decisions NOT to do** — items considered and deliberately skipped. Capture these separately so a future instance doesn't waste a turn re-proposing them.

**Long lists: categorize.** If the inventory has more than ~10 items, a flat list becomes unscannable. Group by axis the next instance actually cares about — typical categories:

   - Out-of-scope / blocked by architecture
   - Magical features (1-2 day each, real UX impact)
   - Small wins (30 min – 1h each, friction-removing)
   - Test-coverage gaps
   - Architecture cleanup (low urgency)
   - Documentation gaps
   - Vision / roadmap items
   - ADR-deferred sections
   - Cross-cutting open questions
   - Explicit decisions NOT to do

A short table per category beats one mega-list of 30 rows.

**EXHAUSTIVENESS CHECKLIST** (binding — Phase 2 is NOT complete until each row is honestly answered):

This is a SECONDARY auditing pass *after* the scan-for items + categorization. The scan-for list catches categories; this checklist catches the specific failure modes that bypass categories.

- [ ] Every file modified this session → what unresolved follow-up does it imply? (e.g., new dependency without consumer, modified config without doc update, partial feature with a stub)
- [ ] Every new dependency / API key / env var introduced → matched consumer in committed code? Or orphan? Orphans become threads.
- [ ] Every doc file (README, CONVENTIONS, design docs, ADRs) touched-adjacent → still accurate, or now stale relative to the new code?
- [ ] Every operator-manual workflow exercised (not just shipped code) → operator burden made explicit for next session?
- [ ] Every "we discussed but didn't ship" idea → captured as a thread, not silently dropped because it didn't make it to code?
- [ ] Every commit message containing "TODO" / "XXX" / "FIXME" → promoted to a thread?
- [ ] Every test added with `pytest.mark.skip` / `xfail` / similar → captured as a thread with the unblock condition?
- [ ] Every external account / cookie / persistent context modified by automation → operator cleanup step documented?
- [ ] Every prior-handover thread in the "Deferred" section → resolved this session, still deferred, or obsoleted? Never silently drop.
- [ ] Every user decision arc this session (scope-setting, redirect, ratification, "go ahead"-style authorization, "why" rationale) → captured as a **verbatim quote** with timestamp in Phase 4 doc? Paraphrase is not sufficient — the JSONL is local-only and the /compact summary strips cadence and specificity. The user's exact words are the grounding for every claim about what they wanted.

If any row is unchecked, Phase 2 is incomplete. **Optimize the thread list for operational completeness, NOT for prose quality.** The narrative arc lives in the Session Summary in Phase 4; the threads list lives for the next instance's worklist. See anti-pattern "Silent Omission via Conciseness" below.

**Output format:**
```
## Pending Threads

### Active (Continue These)
1. **<thread name>**
   - Status: <where we left off>
   - Next step: <specific action>
   - Files: <relevant files>
   - Context: <any non-obvious context>

### Blocked (Waiting)
1. **<thread name>**
   - Blocked on: <what we're waiting for>
   - Resume when: <condition>

### Deferred (Acknowledged but Parked) — exhaustive
<If ≤10 items: flat list. If more: categorize.>

#### <Category, e.g. "Small wins">
| Item | Why deferred / sketch |
|---|---|
| ... | ... |

#### <Category, e.g. "Vision modes">
| ... | ... |

### Explicit Decisions NOT to Do
| Item | Why skipped |
|---|---|
| ... | <so future instances don't re-litigate> |
```

### Phase 3: Session Learnings

**Goal:** Capture insights that should persist beyond this session.

**Categories of learnings:**

| Type | What to Capture | Where to Write |
|------|-----------------|----------------|
| **Codebase insight** | "The beeper API uses linkedMessageID not replyTo" | CLAUDE.md or relevant ADR |
| **User preference** | "User prefers X over Y" | CLAUDE.md |
| **Bug pattern** | "This class of bug keeps recurring" | ISSUES_AND_GAPS.md or new ADR |
| **Skill gap** | "Skill X didn't cover situation Y" | Skill update proposal |
| **Architecture insight** | "These components are coupled in non-obvious way" | CODEBASE_MAP.md or ADR |

**Capture format:**
```
## Session Learnings

### For CLAUDE.md (Project Memory)
- <insight that helps future instances>

### For MEMORY.md (Personal/Cross-Project)
- <pattern or preference that applies broadly>

### Potential Skill Updates
- **<skill name>:** <what was missing or wrong>

### ADR Candidates
- **<topic>:** <why this needs architectural documentation>
```

**Write the learnings — routing priority (check in order):**

1. **Auto-memory infrastructure**: if `~/.claude/projects/<encoded-cwd>/memory/`
   exists, this is the project's structured memory directory. Each
   memory is its own `.md` file with frontmatter (`name`, `description`,
   `type` ∈ {user, feedback, project, reference}); `MEMORY.md` is the
   one-line index. Write here for project-specific facts that should
   persist across sessions. The encoded cwd replaces non-alphanumeric
   chars with `-` (e.g. `/Users/me/proj` → `-Users-me-proj`).
2. **Project CLAUDE.md / AGENTS.md**: if no auto-memory dir, append to
   the project's CLAUDE.md or AGENTS.md (repo root). Project-checked-in
   instructions for any future instance working in this repo.
3. **Cross-project ~/.claude/MEMORY.md**: only for patterns that
   genuinely apply across projects. Don't pollute it with project-local
   facts.
4. **ADR**: if the insight is architectural and warrants ratification
   (not just a fact), draft an ADR rather than a memory.

If unsure which applies, **check whether the auto-memory dir exists
first** — its presence indicates the user has explicit infrastructure
and writes there are first-class.

- Note skill updates for later (or apply if time permits via the
  `expansion:skill-update` meta-skill)

### Phase 4: Handover Document

**Goal:** Create a single artifact that the next instance can read to get up to speed instantly.

**Location:** Write to a predictable location that survives compaction:
- Option A: Append to project's CLAUDE.md under `## Latest Handover`
- Option B: Write to `docs/HANDOVER.md` (git-tracked)
- Option C: Write to `.claude/handover/<timestamp>.md` (local)

**Template:**
```markdown
# Handover: <date> <time>

## Session Summary (narrative — for humans skimming)
<2-3 sentences: what was accomplished, what's the current state.
Optimize for readability; this is the elevator pitch.>

## Commits This Session
- `<hash>` <message>
- `<hash>` <message>

## Verbatim user quotes (chronological — REQUIRED, not optional)
*The conversation JSONL is local-only and will NOT survive into the next
session. The /compact summary paraphrases lossily — it loses cadence,
specific terminology, and the precise force of the user's redirects. The
next instance has no way to verify a paraphrased "user wanted X" claim
against what the user actually said. Capture the user's own words here,
with timestamps, so every downstream decision in this handover has a
grounded source.*

*Scope: every decision arc — initial scope-setting, mid-session redirects,
ratifications ("yes do that"), authorizations ("yep go ahead"), explicit
NOT-to-do ("don't bother with X"), and any "why" rationale the user
volunteered in their own framing. Group by arc if it helps readability.
Extract from `~/.claude/projects/<encoded-cwd>/*.jsonl` if the session
has been compacted.*

- `<YYYY-MM-DDTHH:MM>` *"<exact verbatim quote, no editing>"* — <what this directed / ratified / blocked>

## ADRs Written / Updated This Session
- **ADR-NNN: <title>** — <one-line summary>

(Omit section if none.)

## Pending Threads (enumeration — for the next instance's worklist)
*EVERY pending item from this session AND every unresolved item from the
prior handover / ADRs / roadmap. Aim for completeness over compression.
The narrative lives above; this is the operational inventory. If this
section feels short relative to the session length, re-run the Phase 2
EXHAUSTIVENESS CHECKLIST before declaring complete.*

### Continue Immediately
1. **<thread>** — <next step + relevant files + non-obvious context>

### Blocked
1. **<thread>** — waiting on <X>; resume when <condition>

### Deferred (exhaustive — see Phase 2 for scan sources)
<If ≤10 items keep flat; otherwise categorize.>

### Carried forward from prior handover
1. **<thread>** — original handover line; current status (resolved / still pending / obsoleted by <X>)

## Key Context
<Non-obvious information the next instance needs>
- <context item>
- <context item>

## Operator Cleanup (manual steps for the human)
*Anything the user needs to do manually that automation didn't handle —
account state to clear, env vars to rotate, persistent browser contexts
to wipe, files to delete, external services to check, etc. If automation
modified state outside the repo, name it here.*

- <e.g., "Clear Custom Instructions in chatgpt.com/settings/Personalization on the avalokai account — currently set to <persona> for trajectory work">
- <e.g., "Wipe ~/.atlas/browser-state/<site>/ if you want to start fresh">

(Omit section if no manual steps required.)

## Learnings Captured
- [x] Added to CLAUDE.md: <what>
- [x] Added to MEMORY.md: <what>
- [ ] Skill update needed: <skill> — <issue>

## Running Processes
- <process> — PID <X> — purpose: <what it does>

## Resume Instructions
1. <First thing to do>
2. <Second thing to do>

## Calibration moments (optional but recommended)

A compressed table of "things that surprised me" from this session, each
mapped to a one-line lesson. Future-you (or the next instance) can scan
it in 30 seconds and internalize what NOT to do. Skip if the session was
routine maintenance with no surprises.

| Moment | Lesson |
|---|---|
| <specific thing that went sideways or required course-correction> | <one-line takeaway> |

Example rows from real sessions:
| Tried to fix bug X without live repro | §2-not-TBD is a hard rule, not a suggestion |
| Re-read claim mid-investigation, found I'd drifted | Re-read verbatim claim after every architectural decision |
| Auto-mode classifier blocked a novel script | Show contents inline (cat) before running unfamiliar scripts |

---
*Handover by Claude instance at <context_usage>% context*
```

### If HANDOVER.md already exists

The git-tracked option (Option B) is a single-document-per-repo pattern: each session's handover REPLACES the previous one. The previous handover's value lives in git history; the file always reflects the latest session.

Before overwriting:
- Skim the previous handover for items still pending (carry them forward into the new doc's "Continue Immediately" or "Deferred" sections)
- Don't lose context that's still relevant — bring it forward explicitly

## Automatic Hook Integration

Claude Code has a `PreCompact` hook that fires at ~83.5% context (before automatic compaction).

**Already configured in `~/.claude/settings.json`:**
```json
{
  "hooks": {
    "PreCompact": [
      {
        "matcher": "auto",
        "hooks": [
          {
            "type": "command",
            "command": "echo '[HANDOVER TRIGGER] Context at compaction threshold - run /handover'",
            "timeout": 5
          }
        ]
      }
    ],
    "SessionStart": [
      {
        "matcher": "compact",
        "hooks": [
          {
            "type": "command",
            "command": "echo '[POST-COMPACTION] Check docs/HANDOVER.md for context'",
            "timeout": 5
          }
        ]
      }
    ]
  }
}
```

**How it works:**
1. When context hits ~83.5%, `PreCompact` hook fires with reminder to run `/handover`
2. You run `/handover` to preserve state
3. Compaction happens, summarizing conversation
4. `SessionStart` with `compact` matcher reminds next instance to check handover docs

**Behavior when triggered:**
- Announce: "Context at compaction threshold — running handover before compaction."
- Run all four phases
- Be concise — context is precious
- Prioritize: commits > threads > learnings > document

## Edge Cases

### Nothing to Hand Over
If the session was purely exploratory or all work is committed:
```
HANDOVER: Clean state
- All changes committed and pushed
- No pending threads
- Session was: <brief description>
```

### Urgent Incomplete Work
If there's critical incomplete work:
1. Create WIP commit with clear message
2. Document exact resume point
3. Flag as URGENT in handover doc
4. Consider notifying user: "Incomplete critical work — see handover doc"

### Background Processes Running
If long-running tasks are in progress:
1. Document task IDs and how to check status
2. Note expected completion
3. Describe what to do when complete
4. Example:
   ```
   BACKGROUND: Beeper ingestion agent (PID 12345)
   - Started: 15:30
   - Status: Check with `tail -20 /path/to/output`
   - On complete: Verify items in DB, then can be killed
   ```

### Multiple Projects Touched
If session spanned multiple repos:
1. Run commit phase for each
2. Note cross-project dependencies
3. Document in each project's handover location

## Anti-Patterns

| Don't | Do Instead |
|-------|-----------|
| Skip handover because "it's all in git" | Git doesn't capture context, threads, or learnings |
| Write vague threads ("finish the thing") | Specific: file, line, exact next step |
| Batch all changes into one mega-commit | Logical atomic commits |
| Forget background processes | Always check and document |
| Skip learnings phase | This is how you improve over time |
| Write handover doc but don't commit it | Uncommitted docs don't survive |
| Document only this-session TODOs | Scan documented deferred state too — prior HANDOVER, ADRs, roadmap, architecture docs, audit reports. A new instance shouldn't have to re-discover what the project has been carrying for weeks. |
| Re-propose items the project decided to skip | Capture "explicit decisions NOT to do" in a separate section so future instances see the prior reasoning |
| **Silent Omission via Conciseness**: write a "clean" handover that under-enumerates (3 named wins, 8 named threads, looks crisp and professional, drops items because they didn't fit the narrative) | Default to enumeration over prose. The Session Summary is for human skim; the thread list is for the next instance's worklist. Better to be ugly-and-complete than crisp-and-incomplete. Run the EXHAUSTIVENESS CHECKLIST (Phase 2) before writing Phase 4 — if it surfaces items, include them even if they break the narrative flow. |
| **Paraphrasing the user's voice** ("user wanted X", "we agreed Y", "the directive was Z") | Quote verbatim with timestamp. Paraphrase strips the cadence and specific terminology that carries the WHY (e.g. user saying *"forcing function"* vs. summary saying *"a strict rule"* — different intents). Future instances can't verify a paraphrase against the original. JSONL is local-only; verbatim capture in the handover .md is the only durable grounding. |

## Checklist

- [ ] Checked git status across all touched repos
- [ ] Committed all changes (or stashed with clear message)
- [ ] Pushed commits (or noted why not)
- [ ] Inventoried all pending threads — scanned BOTH session-local TODOs AND documented deferred state (prior HANDOVER, ADRs, roadmap, architecture docs, audit reports, README "candidate" lists)
- [ ] Classified threads: active / blocked / deferred; categorized deferred if >10 items
- [ ] Captured "explicit decisions NOT to do" so they don't get re-proposed
- [ ] **Captured verbatim user quotes** (timestamp + own words) for every decision arc — scope-setting, redirects, ratifications, authorizations, NOT-to-do, "why" rationale. JSONL is local-only and /compact paraphrases lossily; this is the only durable grounding for "what the user wanted."
- [ ] Captured session learnings
- [ ] Updated CLAUDE.md with project insights
- [ ] Updated MEMORY.md with cross-project learnings
- [ ] Noted any skill update opportunities
- [ ] Documented running background processes
- [ ] Wrote handover document
- [ ] Handover document is committed/persisted
- [ ] Provided specific resume instructions

## Example Handover

*This example is deliberately short — a quiet session with a few session-local items. Real handovers on long-running projects often have 20+ deferred items pulled from prior HANDOVERs, ADRs, roadmap, and audit reports; in that case the Deferred section should be categorized (see Phase 2 → "Long lists: categorize") rather than a flat list.*

```markdown
# Handover: 2026-02-11 21:00

## Session Summary
Fixed reply-to-image workflow (3 bugs: beeper extraction, DB upsert, workflow angel).
Added Browser Automation angel. Created /surfaceTechDebt skill.

## Commits This Session
- `e4bdbf7` feat(voices): add voice profiles for cross-recording speaker ID
- `7f0c5b8` feat(angels): add Browser Automation angel with Chrome extension
- `5c1f603` fix(db): persist participants on item upsert
- ... (10 total, all pushed)

## Pending Threads

### Continue Immediately
1. **ADR-021 Feedback Dashboard** — Plan exists at ~/.claude/plans/partitioned-sparking-river.md, not started

### Blocked
None

### Deferred
1. **Workflow capability manifest** — User asked about meme routing (retrieve vs generate vs edit), parked for reply-to-image fix

## Key Context
- Beeper uses `linkedMessageID` for replies (not `replyTo`)
- Workflow angel timeout is 600s (was 300s)
- Web server and beeper ingestion running as background tasks

## Running Processes
- Web server — task bc66aee — check with `tail ~/.../bc66aee.output`
- Beeper ingestion — task b6c3958 — actively processing messages

## Resume Instructions
1. Verify reply-to-image works: send image to Telegram, reply with "p ud <prompt>"
2. If working, continue with ADR-021 Feedback Dashboard implementation

---
*Handover by Claude at ~85% context*
```

## Post-Handover

After handover completes:
1. User should see: "Handover complete. Safe to close session or continue."
2. If auto-triggered by hook: Compaction proceeds, next instance reads handover doc
3. Handover doc serves as "boot sequence" for next instance
