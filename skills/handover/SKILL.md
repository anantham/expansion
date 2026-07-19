---
name: Handover
description: Graceful context transfer before session end or compaction. Commits work, documents pending threads, captures learnings, and prepares the next instance to continue seamlessly.
when_to_use: when user says "handover", "wrap up", "closing session", or when context is approaching 90% capacity and compaction is imminent
version: 1.15.1
changelog:
  1.15.1 (2026-07-19): Anti-pattern added — unbounded "verified green" claims.
  A verification claim must name the commands run, the SHAs they ran against,
  AND the known not-verified surface. Grounded in a trial-merge verdict that
  named tsc+tests but not the never-run vite build, where the real blocker
  (an unresolvable gitignored import) lived. Subtraction: Background-Processes
  example compressed (restated Principle 4's bolded guidance).
  1.15.0 (2026-07-04): Background Processes Running edge case now prescribes making
  long jobs SURVIVE the session — a Claude Code background-bash job is killed at its
  ~10-min tool timeout, so multi-hour work must be a detached OS process (Start-Process
  -WindowStyle Hidden / nohup) with a PID file + resumable state journal, and the PID
  must be verified alive before a handover claims "still running" (a dead/reused PID
  misleads worse than silence). Grounded in a Beeper backfill sweep that only survived
  once detached + resumable.
---

# Handover

## Purpose

You are about to lose this context. Whether due to compaction, session end, or context limit — everything you've learned, every thread you're tracking, every insight you've gained will vanish unless you capture it NOW.

This skill ensures continuity across instances. The next Claude picking up this conversation should be able to continue as if no context was lost.

**Announce at start:** "Running handover **v\<version\>** — committing work, documenting threads, and capturing learnings for the next instance." Replace `<version>` literally with the value from this skill's frontmatter `version:` field above so the user can verify which skill version is actually loaded.

## Related Skills

Handover focuses on session-state preservation. Two adjacent skills handle downstream concerns:

- **`/meta-update`** — Updates `~/.claude/CLAUDE.md` with cross-project protocol learnings extracted from this session. Run AFTER handover. NOTE: **`/mu` is NOT this** — in current environments `/mu` aliases `expansion:skill-update`; invoke meta-update by its full name.
- **`/expansion:skill-update`** — Updates THIS or any other skill based on friction encountered while using it. Run when the skill's instructions didn't quite cover your situation.

Typical end-of-session flow: `/handover` → `/mu` (= skill-update) → `/meta-update` (only the first is always relevant; the latter two are conditional).

## Available cross-project affordances

Before writing or recommending a new utility, check **`~/Documents/Ongoing Local/AFFORDANCES.md`** — the hand-curated registry of tools that already exist across the user's projects (browser automation on logged-in frontier accounts, scheduled recurring tasks, and more — grep it for the live list; it drifts). If a session-relevant affordance was used or could have been used, add a one-line "next instance: see `<path>` for `<problem>`" pointer to the handover's **Key Context**. Don't duplicate the registry's content — the pointer is enough.

## Ask the human when blocked (binding)

Handover itself is a context-spending act. Don't burn dying context trying to recover from blockers solo.

When you encounter ANY of the following during handover, **stop and ask the user** in one sentence rather than fabricating, guessing, or paving over:

- **Permission denied** during a file write, tool invocation, or browser action (do NOT write a placeholder file with fabricated content; report the block).
- **Missing file or directory** at a path the protocol expects (e.g., `docs/HANDOVER.md` location, an ADR directory).
- **Ambiguous prior handover** — if `Carried forward from prior handover` items are vague and the current state doesn't disambiguate, ask which interpretation is current.
- **Conflicting signals** — e.g., a thread is "Active" in your head but the commit history shows it was abandoned; verify with the user before re-listing it.
- **Tool unavailability** — if a deferred tool you need can't be loaded via ToolSearch, surface that explicitly rather than skipping the step silently.

The pattern: *"Blocked on X. One sentence I'd write if I knew the answer: '…'. What's the actual answer?"*

This is consistent with the broader anti-pattern named in `~/.claude/CLAUDE.md` ("don't burn dying context fabricating output when asking would resolve in 30 seconds"). Verbatim user quotes captured in Phase 4 are the durable grounding; one extra exchange to disambiguate is cheaper than a wrong-but-confident handover.

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

**BINDING GATE — do this BEFORE any mechanical phase.** Your FIRST
user-facing message in a handover MUST be the Phase-0 capture proposal:
the specific high-leverage, hot-context-only synthesis you intend to
produce, **named concretely AND triaged per item**. For each named item,
give a **Do-NOW vs Defer** verdict against the one-question test — *"could
a fresh agent reproduce this from the final diff + instructions?"* (No →
Do-NOW: synthesis, rationale, user-voice, a cross-project pattern; Yes →
Defer to a pointer the next agent can act on). Then name **which items you'd
still do if context were critically tight** — that forces a ranking, not a
flat list. Format: *"Here's what only this dying context can produce: \<X\>
[Do-NOW — irreproducible synthesis], \<Y\> [Do-NOW], \<Z\> [Defer — next
agent re-runs the sweep] — what should I drop?"* A bare enumeration of
candidates WITHOUT the per-item Do-NOW/Defer verdict is NOT a valid
proposal: surfacing the list but handing the triage to the user is the
"treat Phase 0 as a checkbox" anti-pattern one level up — the triage matrix
above is the whole point, so APPLY it per item, don't just reference it.

Before you present, run ONE adversarial completeness pass: **what
high-leverage, hot-context-only capture am I NOT naming?** Recurring misses:
a user-working-preference you lived this session; a reusable method or
tool-usage discovered through failure; a cross-decision posture. If the user
still has to ask "is that exhaustive?", this self-check was skipped.

You may NOT proceed to Phase 1 (commit checkpoint) until that triaged
proposal is made and the user has pruned/redirected it.

The proposal must contain real synthesis — a cross-decision pattern, a
posture spanning multiple choices this session, a rationale that won't
survive the diff alone, a project/cross-project memory only this trace
can write. It is NOT a restatement of the mechanical scaffold ("I'll
commit, list threads, write the doc"). If, after honest reflection, you
truly cannot name anything only this context can produce, say so
explicitly and justify it — don't silently skip to the scaffold.

The user has better marginal-value judgment than the exhausted-context
model running on fumes — and the model's strong default is to fall into
the comfortable mechanical scaffold (Phases 1-4) and treat Phase 0 as a
checkbox. Resist that. After the proposal is pruned/approved, proceed
with Phases 1-4 (the mechanical scaffold). **Phase 0 is what makes the
handover signal-dense rather than merely complete; the user should never
have to ask "is that all?"**

### Phase 1: Commit Checkpoint

**Goal:** Get all work safely committed. Nothing uncommitted survives compaction.

**Steps:**

1. **Check git status — and confirm you're on the intended branch.**
   ```bash
   git status
   git branch --show-current   # shared checkouts get switched by parallel sessions
   git diff --stat
   ```
   In a checkout shared with parallel agents, the branch can be switched out from
   under you mid-session (it can even switch **twice**) — so a later `git commit`
   silently lands on someone else's branch and `git push` fails (no upstream /
   non-fast-forward). If the current branch isn't the one you've been committing to
   all session:
   - Do NOT switch the shared checkout back (clobbers their uncommitted WIP), and
     do NOT commit onto their branch (pollutes their PR).
   - Land your commit on the target branch via a SEPARATE worktree:
     `git worktree add <tmp> <target>; git -C <tmp> cherry-pick <sha>` (or write the
     artifact there) `; git -C <tmp> push origin <target>; git worktree remove <tmp> --force`.
   - After a prod-restart-from-session, the running app serves whatever branch the
     checkout sits on — verify it's the intended one.

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

   **The index itself can drift between `git add` and `git commit`.** In a
   checkout shared with parallel agents, `git add <specific files>` does not
   lock the index — another agent's concurrent `git add` can insert its own
   staged files into yours in the seconds before you commit. Checking
   `git status`/`git diff --stat` once, right after `git add`, is not
   enough. Re-run `git diff --cached --stat` again *immediately before* the
   actual `git commit` call, every time, and `git restore --staged <path>`
   anything that isn't yours. This has been observed to happen twice in one
   session. **This rule is unconditional — it applies in PRIVATE worktrees
   too**: your own earlier commands stage things silently (`git mv`
   auto-stages; setup scripts may `git add`), and hours-old staged renames
   have ridden into an unrelated commit in a single-agent worktree with no
   other session involved.

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

   **If the push fails:** an auth/connection message ("Please make sure you have
   the correct access rights and the repository exists") is often **transient** —
   retry once before treating it as failure, then re-verify with
   `git log origin/<branch>..HEAD`. If instead it's rejected *non-fast-forward*,
   `git fetch` + rebase onto `origin/<branch>`; never `--force` a shared branch.

   **If `git rebase` fails on a fully-dirty-tree error, try `git merge` instead
   before assuming you're blocked.** `git rebase` requires the *entire* working
   tree clean — it blocks on ANY uncommitted change anywhere, even files
   unrelated to the incoming commits, which is too blunt for a shared
   checkout that always has other agents' WIP sitting around. `git merge` is
   surgical: it only blocks on the specific files that would actually
   collide ("local changes would be overwritten by merge: <files>"), giving
   you a much more actionable signal. When merge does block on specific
   files: do NOT stash, touch, or overwrite someone else's uncommitted file.
   Instead, create a scratch worktree at your current HEAD on a throwaway
   branch, merge origin there (a clean tree has nothing to collide with),
   and push from there:
   ```bash
   git worktree add <tmp> -b <temp-branch> HEAD
   git -C <tmp> merge origin/<branch> --no-edit
   git -C <tmp> push origin <temp-branch>:<branch>
   git worktree remove <tmp> --force
   git branch -D <temp-branch>
   ```
   Zero risk to anyone else's in-progress state. If the merge itself hits a
   real *content* conflict (not a collision with someone's uncommitted
   file — an actual textual conflict, e.g. two sessions both added an entry
   at the same line in an index file like `docs/HANDOVER.md`), resolve it
   normally in the scratch worktree (it's just a merge conflict like any
   other) rather than treating it as another instance of this failure mode.

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

### Phase 1a: Cruft Census (parallel-session hygiene)

**Goal:** Surface accumulated parallel-session debris — stale branches, dangling worktrees, forgotten stashes — so the operator can decide what dies now vs. lives one more session.

**Why this exists:** With parallel Claude sessions a normal pattern, branches and worktrees accumulate fast. Each session creates artifacts; few sessions clean up. Without a forced confrontation at handover time, cruft compounds until cleanup becomes daunting. The handover is the natural checkpoint — operator is already context-switching out, the cost of confronting cruft is amortized.

This phase pairs with Phase 1's "scope check" (separate YOUR changes from ambient state): scope-check prevents wrongful commits IN this session; cruft census prevents debt accumulation ACROSS sessions.

**Run the census:**

```bash
# 1. Active worktrees (count + paths)
git worktree list

# 2. Unmerged local branches (excluding current + main)
git branch --no-merged main | grep -v "^\*"

# 3. Merged-but-undeleted local branches (sweep candidates — EXCLUDING worktree-*:
#    those back active/locked worktrees, so deleting them breaks the worktree)
git branch --merged main | grep -vE "^\*|main$|worktree-"

# 4. Stashes
git stash list

# 5. Stale branches (>14 days since last commit, on any branch)
for b in $(git for-each-ref --format='%(refname:short)' refs/heads/); do
  age_days=$(( ($(date +%s) - $(git log -1 --format=%ct "$b")) / 86400 ))
  [ "$age_days" -gt 14 ] && echo "$b ($age_days days)"
done
```

**Silent on clean state:** If `worktrees ≤ 1` AND `unmerged branches = 0` AND `stashes = 0` AND `stale branches = 0`, skip the rest of this phase. No output, no operator interrupt. Solo work on a single branch stays frictionless.

**Decision rules when cruft exists:**

| Finding | Action |
|---------|--------|
| Merged-but-undeleted branch (NOT `worktree-*`) | Offer to sweep: `git branch --merged main \| grep -vE "^\*\|main$\|worktree-" \| xargs -r git branch -d`. **Never** sweep `worktree-*` branches — they back active/locked worktrees; deleting breaks them. |
| Stash >7 days old with no recent reference | Surface in handover; ask: review, apply, or drop |
| Unmerged branch >14 days, no recent commits | Name in handover's **Deferred** section with explicit expiry condition (must merge, become ADR, or get `git branch -D`'d by date X) |
| Worktree with no recent commits AND its branch is merged | Offer to remove: `git worktree remove <path>` |
| Worktree with uncommitted changes | DO NOT auto-clean. Flag in handover; operator must triage |
| The worktree this session is STANDING IN | Cannot be removed from inside the session — if its branch is merged, name it as next-session cruft (with any hazards, e.g. a junctioned node_modules that must be deleted non-recursively first) rather than attempting removal |

**Surface, don't shred:** Auto-cleans only merged-undeleted branches (provably safe — no work is lost). Unmerged branches, worktrees-with-uncommitted-work, and stashes get *named* in the handover with expiry conditions. The operator decides; you report.

**Otherwise output to operator + handover doc:**

```
PARALLEL-SESSION CRUFT:
- 3 active worktrees: ../tc-share, ../tc-norm, ../tc-vocab
- 5 unmerged branches: session-share, session-norm, session-vocab, session-priv, session-audio
- 2 merged-but-undeleted: session-old-1, session-old-2  ← sweep candidates
- 1 stash: "WIP: vocab refiner — 2026-05-22"  ← review or drop
- Stale (>14 days): session-old-1 (32 days)
```

Then ask the operator:
- "Sweep N merged branches now?" (one-liner above)
- "Drop M-day-old stash?" (`git stash drop stash@{N}`)
- For stale unmerged branches: prompt them to add to the handover's Deferred section with an explicit expiry date.

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
- Option D (**background / isolation-guarded sessions — often the only one that works**):
  write `~/.claude/projects/<encoded-cwd>/memory/_session-handover-<date>.md` and add a
  one-line pointer at the TOP of that dir's `MEMORY.md` (which auto-loads next session).
  Reach for this when A/B/C are unwritable: a background session's Write to the shared
  checkout is rejected until `EnterWorktree` (so A, B, and C all fail), **and** `.claude/`
  is commonly gitignored, so even a worktree copy of C can't be committed → lost. This is
  consistent with Phase 3 (which already routes *learnings* to auto-memory); the
  auto-memory doc is durable **and** auto-loaded, so here it is the PREFERRED fallback,
  not a last resort. (The `MEMORY.md` pointer is what makes it discoverable — a bare file
  in the memory dir is not auto-surfaced.)

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
with timestamps when cheaply available (the offline JSONL holds them),
so every downstream decision in this handover has a grounded source.
In-context messages carry NO timestamps — when they aren't cheaply
recoverable, preserve strict chronological order and mark the list
"times approximate" instead. NEVER fabricate a timestamp to satisfy the
template; the verbatim words are the load-bearing part.*

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

## Parallel-Session Cruft
*Omit this section entirely if Phase 1a was silent (clean state). Otherwise:*
- Worktrees: <count + paths>
- Unmerged branches: <list>
- Merged-but-undeleted (sweep candidates): <list>
- Stashes: <list, with age>
- Stale (>14 days): <branch (age)>
- Decisions made this session: <e.g. "swept 2 merged branches", "dropped stash@{0}">
- Deferred for next session: <e.g. "session-vocab — expires 2026-06-10 if not merged, else `git branch -D`">

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

Two hooks in `~/.claude/settings.json` (already configured — read that file for the exact JSON): a `PreCompact(auto)` hook fires at ~83.5% context with "[HANDOVER TRIGGER] … run /handover", and a `SessionStart(compact)` hook reminds the next instance to check the handover docs after compaction.

**Behavior when auto-triggered:**
- Announce: "Context at compaction threshold — running handover before compaction."
- Run all phases, concisely — context is precious.
- Prioritize: commits > threads > learnings > document.

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
4. **Make it survive the session, and verify it's alive before claiming so.** In
   Claude Code a *background-bash* job is killed when its tool call times out
   (~10 min) — it will NOT outlive the session. For work that must keep running
   (multi-hour jobs), launch it as a **detached OS process** (`Start-Process
   -WindowStyle Hidden` on Windows, `nohup … &` elsewhere), write its **PID to a
   file**, and make it **resumable from a state journal** so the next instance can
   poll and resume. Before writing "still running (PID X)", confirm the PID is
   actually alive — a documented-but-dead process, or a reused PID pointing at an
   unrelated program, misleads the next instance worse than saying nothing.
5. Example:
   ```
   BACKGROUND: Beeper backfill sweep — detached, PID 26452 (PID file + state journal)
   - Status: poll DB row count + agents/state/*_state.json
   - Resume: re-run the same command — skips complete units, re-walks stragglers
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
| **Mechanical-scaffold-first**: running Phases 1-4 (commit, thread-list, write doc) and treating Phase 0 as a checkbox → a complete-but-thin handover; the cross-session synthesis only this dying context can produce gets silently dropped | Open the handover with the Phase-0 capture proposal (BINDING gate) — name the hot-context-only items concretely BEFORE the scaffold. If the user has to ask "nothing worth doing with hot context?", Phase 0 was skipped. The synthesis (a posture across decisions, a cross-ADR/cross-file pattern, a "why" that won't survive the diff) is the highest-value thing a dying context produces — it can't be reconstructed by a fresh agent reading the commits. |
| **Write the handover doc, then keep acting** (merging, pushing, deleting, deploying) without updating it | The doc is stale before the session ends and boots the next instance into wrong state. Handover is the LAST state-changing act of a session; if the user asks for more actions after the doc is written, re-open it and reconcile every claim those actions changed (PR states, service status, cruft lists) before ending. A doc that said "3 PRs open, awaiting merge" was wrong within minutes when the same session then merged all three. |
| **Unbounded "verified green" claims** ("trial merge was clean — tsc 17, tests green") | A verification claim names the commands run, their key numbers, the SHAs they ran against, AND the known not-verified surface ("vite build not run"). The successor must be able to tell whether the verdict still binds and where its edge is. A 2026-07-18 trial-merge verdict named tsc+tests; the real blocker (a gitignored import unresolvable on fresh checkouts) lived precisely in the never-run build gate, and the successor read "green" as one gate wider than measured. |
| Skip the Phase 1a cruft census thinking "the next session will clean up" | The next session won't either. Every handover confronts what's accumulated. Sustainable parallel-session work depends on it. |
| Auto-delete unmerged branches or worktrees-with-uncommitted-work during cruft census | Surface, don't shred. The operator decides; you report. Only merged-undeleted branches are provably safe to auto-sweep. |

## Checklist

- [ ] Checked git status across all touched repos
- [ ] Committed all changes (or stashed with clear message)
- [ ] Pushed commits (or noted why not)
- [ ] Ran Phase 1a cruft census (worktrees, unmerged branches, merged-undeleted, stashes, stale >14d). If cruft found: offered sweep of merged-undeleted, surfaced stale ones to operator with expiry conditions.
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

A quiet session's handover can be ~25 lines: Session Summary (2 sentences), Commits
(hash+message list, "all pushed"), Pending Threads (1 active item with file+next-step,
"Blocked: none", 1 deferred), Key Context (2–3 non-obvious facts like "Beeper uses
`linkedMessageID` not `replyTo`"), Running Processes (task id + check command), Resume
Instructions (2 numbered steps). Long-running projects instead often carry 20+ deferred
items pulled from prior HANDOVERs/ADRs/roadmap — categorize those (see Phase 2) rather
than flattening. Match the doc's weight to the session's actual state; completeness of
enumeration matters, prose volume doesn't.

## Post-Handover

After handover completes:
1. User should see: "Handover complete. Safe to close session or continue."
2. If auto-triggered by hook: Compaction proceeds, next instance reads handover doc
3. Handover doc serves as "boot sequence" for next instance
