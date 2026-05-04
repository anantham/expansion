---
name: Handover
description: Graceful context transfer before session end or compaction. Commits work, documents pending threads, captures learnings, and prepares the next instance to continue seamlessly.
when_to_use: when user says "handover", "wrap up", "closing session", or when context is approaching 90% capacity and compaction is imminent
version: 1.1.0
---

# Handover

## Purpose

You are about to lose this context. Whether due to compaction, session end, or context limit — everything you've learned, every thread you're tracking, every insight you've gained will vanish unless you capture it NOW.

This skill ensures continuity across instances. The next Claude picking up this conversation should be able to continue as if no context was lost.

**Announce at start:** "Running handover — committing work, documenting threads, and capturing learnings for the next instance."

## When to Trigger

| Trigger | Context |
|---------|---------|
| Manual: `/handover` | User explicitly requests wrap-up |
| Manual: "wrap up", "closing session" | Natural language equivalents |
| Auto (hook): 90% context capacity | Pre-compaction preservation |
| Manual: Before switching tasks | Context pivot coming |

## The Four Phases

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

**Goal:** Document every open thread so the next instance knows what's pending.

**Scan for:**

1. **Explicit TODOs from this session**
   - User requests not yet completed
   - Errors encountered but not resolved
   - Features partially implemented

2. **Implicit threads**
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

### Deferred (Acknowledged but Parked)
1. **<thread name>**
   - Why deferred: <reason>
   - Revisit: <when/trigger>
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

## Session Summary
<2-3 sentences: what was accomplished, what's the current state>

## Commits This Session
- `<hash>` <message>
- `<hash>` <message>

## Pending Threads

### Continue Immediately
1. **<thread>** — <next step>

### Blocked
1. **<thread>** — waiting on <X>

### Deferred
1. **<thread>** — parked because <reason>

## Key Context
<Non-obvious information the next instance needs>
- <context item>
- <context item>

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

## Checklist

- [ ] Checked git status across all touched repos
- [ ] Committed all changes (or stashed with clear message)
- [ ] Pushed commits (or noted why not)
- [ ] Inventoried all pending threads
- [ ] Classified threads: active / blocked / deferred
- [ ] Captured session learnings
- [ ] Updated CLAUDE.md with project insights
- [ ] Updated MEMORY.md with cross-project learnings
- [ ] Noted any skill update opportunities
- [ ] Documented running background processes
- [ ] Wrote handover document
- [ ] Handover document is committed/persisted
- [ ] Provided specific resume instructions

## Example Handover

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
