---
name: Skill Update
description: Meta-skill for learning from skill usage. Tracks uncertainties, human interventions, and gaps discovered during use, then proposes concrete skill patches.
when_to_use: when actively using another skill and noticing ambiguities, gaps, or places where the skill's instructions were insufficient or wrong
version: 1.0.0
---

# Skill Update (Meta-Skill)

## Purpose

A container for **empirical skill improvement**. When you use any skill and encounter friction — ambiguity, missing guidance, wrong assumptions, or needed human clarification — this meta-skill structures those observations into actionable patches.

**Announce at start:** "Also running Skill Update — I'll track learnings for [skill name] as I work."

## When to Invoke

Invoke this **alongside** another skill when ANY of these occur:
- The skill's instructions don't cover your current situation
- You had to ask the user something the skill should have answered
- The skill's constraints feel wrong for the task (too tight, too loose)
- You made a judgment call the skill doesn't encode
- The user corrected your interpretation of the skill
- You discovered a pattern worth codifying for next time

## Process

### Phase 1: Observation (During Skill Use)

As you work with the primary skill, maintain a running log in your working memory:

```
SKILL_UPDATE_LOG:
- skill: <skill being used>
- task: <what you're trying to accomplish>
- observations:
  1. [AMBIGUITY] <where the skill was unclear>
  2. [GAP] <what the skill doesn't cover but should>
  3. [WRONG] <where the skill's guidance produced a bad result>
  4. [TIGHT] <where constraints were too restrictive for the task>
  5. [LOOSE] <where more structure was needed>
  6. [ASKED_HUMAN] <question asked + why skill didn't answer it>
  7. [JUDGMENT] <decision made that the skill should encode>
  8. [PATTERN] <reusable pattern discovered during work>
```

**Don't interrupt the primary task.** Collect observations silently. The learning phase comes after delivery.

### Phase 2: Reflection (After Delivery)

Once the primary skill's output is delivered, review your log and categorize:

#### 2a. Impact Assessment

For each observation, assess:
- **Frequency**: Will this come up again? (one-off vs recurring)
- **Severity**: How much did it slow you down or risk a wrong output?
- **Generality**: Does this apply to all uses of the skill, or just this task type?

Only propose patches for observations that are **recurring + moderate-to-high severity + general**.

#### 2b. Patch Drafting

For each qualified observation, draft a concrete patch:

```
PATCH: <short title>
TYPE: [addition | modification | removal | restructure]
LOCATION: <which section of the skill to change>
CURRENT: <what the skill currently says (quote or summarize)>
PROPOSED: <exact new text or structural change>
RATIONALE: <why, grounded in the specific experience>
EXAMPLE: <concrete example from this session that illustrates the need>
```

### Phase 3: Proposal (Present to User)

Present patches grouped by type:

1. **Critical** — Skill produced wrong output or required human rescue
2. **Enhancement** — Skill worked but missed an opportunity or common case
3. **Clarification** — Skill was ambiguous; adding precision prevents future confusion

For each patch, show:
- The observation that triggered it
- The proposed change (diffable if possible)
- Confidence that this generalizes (not just this one task)

### Phase 4: Application (After Approval)

On user approval:
1. Read the current skill file
2. Apply approved patches
3. Bump the skill's version (patch increment for fixes, minor for new capabilities)
4. Commit with message: `skill(<name>): <summary of changes>`
5. If the skill is in a plugin repo (like expansion), push the update

## Anti-Patterns

| Don't | Do Instead |
|-------|-----------|
| Interrupt primary work to discuss skill gaps | Note silently, discuss after delivery |
| Propose patches for one-off edge cases | Only patch recurring patterns |
| Rewrite the entire skill based on one use | Targeted patches with rationale |
| Add complexity for hypothetical futures | Only encode patterns you've actually hit |
| Conflate "I did it differently" with "skill is wrong" | Ask: would the skill's way have been better? |
| Skip examples | Every patch needs a concrete example from this session |

## What Makes a Good Patch

- **Grounded**: Comes from a real moment in the task, not abstract thinking
- **Specific**: Points to exact section, proposes exact text
- **Minimal**: Changes only what's needed, preserves working guidance
- **Tested**: You experienced the gap and can describe what would have helped
- **General**: Applies beyond just this one task instance

## Integration with Skill Lifecycle

```
First use of skill
    ↓
Notice friction → SKILL_UPDATE_LOG
    ↓
Complete primary task
    ↓
Phase 2: Reflect on observations
    ↓
Phase 3: Propose patches to user
    ↓
Phase 4: Apply approved changes
    ↓
Next use benefits from improvements
    ↓
Repeat (skills converge toward robustness)
```

## Example Session

**Primary skill**: canvas-diagrams (L3 data flow)
**Task**: Map Beeper API fields → storage decisions

**Observations during work:**
1. [TIGHT] L3 defined as "5-10 nodes, individual steps within one function" — but data-field mapping needs 15-20 nodes across an API boundary
2. [GAP] No color guidance for "kept vs discarded" — only component-type colors exist
3. [GAP] No node type for "raw API field" — only transforms, storage, sources
4. [AMBIGUITY] "Function/Step Detail" framing doesn't fit data-mapping tasks
5. [PATTERN] Data-mapping canvases need a "what comes in" column, "decision" column, and "where it goes" column — a 3-column layout, not top-to-bottom flow

**Resulting patches:**
- Expand L3 definition to include "data-field mapping" as a valid L3 type
- Add "kept/discarded" edge color convention (green=retained, red=discarded)
- Add "API field" node type with its own color
- Add layout variant: "3-column comparison" for mapping tasks

## Checklist

- [ ] Identified the primary skill being used
- [ ] Maintained observation log during work (without interrupting)
- [ ] Completed primary task delivery first
- [ ] Assessed frequency/severity/generality of each observation
- [ ] Drafted patches only for qualifying observations
- [ ] Each patch has: location, current, proposed, rationale, example
- [ ] Grouped by priority (critical/enhancement/clarification)
- [ ] Presented to user for approval before applying
- [ ] Applied changes and bumped version on approval
