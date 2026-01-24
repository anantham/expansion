---
name: postmortem
description: Use after discovering bugs that may have silently caused data loss or degradation. Traces root cause through git history, identifies human-agent miscommunication, then systematically searches for downstream damage.
when_to_use: when a bug is found that was silently failing, especially import errors, silent exceptions, or broken integrations that may have been losing data
version: 1.0.0
---

# Postmortem

## Purpose

Bugs that fail silently are the most dangerous — they don't crash, they just quietly lose data or skip work. This skill provides a structured investigation from **root cause** through **blast radius** to **recovery**.

**Announce at start:** "Running postmortem on [bug description] — tracing origin, assessing damage window, and checking for downstream data loss."

## When to Use

- A bug was found that has been silently failing (no crash, just skipped work)
- An import error, missing function, or broken integration was caught
- Data might have fallen through the cracks during the failure window
- A fix was applied but you need to understand what was lost

## The Five Phases

### Phase 1: Origin Tracing

**Goal:** Identify exactly when and how the bug was introduced.

1. **Find the introducing commit**
   ```
   git log --all -S "<broken_symbol_or_pattern>" -- <file>
   git blame <file> -L <line_range>
   ```

2. **Read the commit message and diff**
   - What was the stated intent?
   - Was this part of a larger feature?
   - Who authored it (human commit or Co-Authored-By agent)?

3. **Identify the miscommunication pattern**
   Common patterns:
   - **Partial refactor**: Symbol moved/renamed but not all import paths updated
   - **Try/except masking**: Error caught too broadly, real failure hidden
   - **Dual import paths**: Primary path works but misses new additions
   - **Feature flag gap**: New code added behind a path that's never reached
   - **Silent fallback**: `except Exception: pass` or `log.debug()` hiding real errors

4. **Document the timeline**
   ```
   INTRODUCED: <commit hash> (<date>)
   DISCOVERED: <today>
   WINDOW: <duration the bug was active>
   PATTERN: <which miscommunication pattern>
   AUTHOR_CONTEXT: <what the commit was trying to do>
   ```

### Phase 2: Root Cause Presentation

**Present findings to human for review:**

- The exact commit that introduced the bug
- The diff showing what went wrong
- The miscommunication pattern identified
- The time window the bug was active
- Ask: "Does this match your understanding? Any additional context?"

**Wait for human confirmation before proceeding.**

### Phase 3: Blast Radius Analysis

**Goal:** Determine what class of problems this bug represents and what data may be affected.

1. **Classify the failure mode**
   - What operation was silently skipped?
   - How often does that code path execute? (every message? once at startup? on specific triggers?)
   - What data would have been produced if it worked?

2. **Quantify the damage window**
   ```
   FAILURE_MODE: <what didn't happen>
   FREQUENCY: <how often the code path runs>
   WINDOW: <start_date> to <end_date>
   ESTIMATED_MISSED: <rough count of missed operations>
   ```

3. **Check for evidence of the gap**
   - Query the database for missing records in the time window
   - Check logs for the silent failure messages
   - Compare expected vs actual data for the period
   - Look for downstream processes that depended on this data

### Phase 4: Class-of-Problem Search

**Goal:** If this specific bug existed, what other similar bugs might exist?

1. **Generalize the pattern**
   - If the bug was a missing import: search for other files with the same dual-import pattern
   - If it was a silent except: search for other broad exception handlers
   - If it was a partial refactor: check all references to the refactored symbol

2. **Systematic search**
   ```
   # Example searches by pattern type:

   # Dual import paths with mismatched symbols
   grep -n "except ImportError" <codebase> → compare imports in try vs except blocks

   # Silent failures
   grep -n "except.*:" <codebase> → check for pass/debug-only handlers

   # Unreachable new code
   git diff <introducing_commit>..HEAD -- <file> → new symbols added only to one path
   ```

3. **Report findings**
   For each potential sibling bug found:
   - File and line
   - The pattern match
   - Whether it's actually broken or just looks similar
   - Severity if broken (data loss vs cosmetic)

### Phase 5: Recovery Plan

**Goal:** Determine if lost data can be recovered and how.

1. **Recoverability assessment**
   For each piece of missed data:
   - Is the source material still available? (messages still in API, files still on disk)
   - Can we re-run the missed operation retroactively?
   - Is there a time limit on recovery? (API pagination limits, data retention)

2. **Recovery actions**
   - List specific commands or operations to recover data
   - Note any that are time-sensitive
   - Flag any that are irreversible if done wrong

3. **Prevention measures**
   - What test would have caught this?
   - Should we add a health check for this code path?
   - Does the error handling need to be louder (warning vs debug)?

## Output Format

```
# POSTMORTEM: <brief title>

## Timeline
- INTRODUCED: <commit> on <date> by <author>
- DISCOVERED: <date>
- WINDOW: <duration>
- FIXED: <commit/uncommitted>

## Root Cause
<1-2 sentences on what went wrong>
Pattern: <miscommunication pattern name>

## Blast Radius
- <operation> was silently skipped for <duration>
- Estimated <N> missed operations
- Affected data: <description>

## Evidence
- <specific queries/logs showing the gap>

## Sibling Bugs Found
1. <file:line> - <description> - <status>

## Recovery
- [ ] <recovery action 1>
- [ ] <recovery action 2>

## Prevention
- [ ] <test or check to add>
```

## Anti-Patterns

| Don't | Do Instead |
|-------|-----------|
| Assume the fix is enough | Investigate what was lost during the failure window |
| Blame the author | Identify the communication pattern that failed |
| Only fix the specific instance | Search for the class of similar bugs |
| Skip the human review step | Always confirm root cause understanding before blast radius |
| Panic about data loss | Assess recoverability calmly — source data often still exists |
| Add defensive code everywhere | Target prevention at the specific failure mode |

## Checklist

- [ ] Found the introducing commit with `git log -S` or `git blame`
- [ ] Identified the miscommunication pattern
- [ ] Calculated the time window of silent failure
- [ ] Presented root cause to human for confirmation
- [ ] Quantified what operations were missed
- [ ] Searched for evidence of the gap in data/logs
- [ ] Generalized to the class of problem and searched for siblings
- [ ] Assessed recoverability of lost data
- [ ] Proposed prevention measures (tests, louder errors)
