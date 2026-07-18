---
name: Meta Update
description: Promote a session's durable cross-project learnings to the shared union store (~/.claude-sync — the MISTAKES.md ledger, LIBRARY.md and its Index in SHARED-MEMORY.md, and the SHARED-MEMORY.md Journal) and to per-project auto-memory. This is the /meta-update ritual.
when_to_use: when the user asks to "update memory" / "promote learnings" / runs /meta-update, or at session close or handover when the work produced cross-project learnings, mistakes, or reusable techniques worth persisting for the next agent on any machine. Run AFTER handover.
version: 1.0.0
---

# Meta-Update (`/meta-update`)

Durably capture what this session learned so the next agent — any machine, any
project — inherits it. Do it deliberately, not mechanically. If the user named a
focus (e.g. "just the codex trap", "project only"), scope to that; otherwise cover
everything worth keeping.

**Announce at start:** "Running Meta-Update — promoting this session's learnings to the union store."

> **Not to be confused with `/mu`.** In current fleet environments `/mu` aliases
> `expansion:skill-update` (learning from how a *skill* was used and patching the
> skill). This skill is the separate, cross-project **memory-promotion** ritual that
> `/meta-update` has long referred to. If they run together, order is
> `/handover` → `/mu` (skill-update) → `/meta-update` (this).

This skill assumes the union store at `~/.claude-sync/` (`SHARED-MEMORY.md`,
`LIBRARY.md`, `MISTAKES.md`), imported by `~/.claude/CLAUDE.md`. If that store isn't
present, tell the user and fall back to editing the relevant project `CLAUDE.md` /
auto-memory directly.

## 0. Guard first — the store silently loses edits
`SHARED-MEMORY.md` is `sendreceive` across devices and the fleet's highest-centrality
file, so concurrent edits resolve last-writer-wins and park the loser in an unread
conflict copy.
- `ls ~/.claude-sync/*sync-conflict* 2>/dev/null` — if any conflict copy exists,
  **UNION-merge** it into the live file (diff the Index + Journal, keep both sides),
  then delete the copy. Never last-writer-wins by hand.
- Prefer append-only edits; re-run this check again after writing (step 4).

## 1. Decide what is worth keeping
Three buckets. Skip anything the repo/git already records, or that only mattered to
this one conversation.
- **Cross-project / cross-machine learnings** → the union store (step 2).
- **Project-specific facts** → this project's auto-memory
  (`~/.claude/projects/<encoded-cwd>/memory/` + its `MEMORY.md` index) and/or the
  project's `CLAUDE.md` (step 3).
- **Mistakes** (yours or ones you caught) → the ledger (step 2).

## 2. Promote to the union store (`~/.claude-sync/`)
- **`MISTAKES.md`** — one tagged line per mistake (format + fixed tag vocabulary live
  at the file top; date = when it OCCURRED, not when logged). Escalation rule: when a
  tag reaches 3 instances still fixed only by notes, stop writing notes — build the
  tool/guard that kills the class, and update those lines' fix-state.
- **`LIBRARY.md`** — append durable, reusable knowledge under the right topic (create
  one if none fits). The Library is NOT auto-loaded, so if the topic or sub-point is
  new, add a pointer to it in the **Index** inside `SHARED-MEMORY.md` — the Index is
  the only part loaded into a session, so an unlisted topic is invisible forever.
- **`SHARED-MEMORY.md` Journal** — add ONE entry at the top, directly under the
  `<!-- newest -->` marker:
  `### [NNNN] YYYY-MM-DD HH:MM TZ · <model-id> · <project> · <device-hostname>`
  then one dense line, cross-referencing related entries by `[NNNN]`. `NNNN` =
  previous top index + 1, zero-padded. **Never edit another writer's entry** (moving a
  clearly-misplaced one verbatim to its correct slot is fine). Keep this file lean
  (Index + Standing facts + Journal only); archive old Journal entries to
  `SHARED-MEMORY-archive.md` when it grows.

Get date/time/host from the shell, don't guess: `date '+%F %H:%M %Z'` and `hostname`.
Tag each promotion with its source project.

## 3. Project auto-memory
For project-specific facts, write/update one file per fact under this project's memory
directory (one fact per file, with frontmatter), and add/update its one-line pointer
in that directory's `MEMORY.md`. Update an existing file rather than duplicating;
delete memories that turned out wrong. Link related memories with `[[name]]`.

## 4. Verify
- Re-run the sync-conflict check (a peer may have written while you did).
- Sanity-check Journal order (newest first) and that any Index pointer you added
  resolves to a real `LIBRARY.md` topic.

Report concisely what you promoted and where. Do not commit or push anything unless
the user asks.
