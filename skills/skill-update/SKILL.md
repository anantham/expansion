---
name: Skill Update
description: Meta-skill for learning from skill usage. Tracks uncertainties, human interventions, and gaps discovered during use, then proposes concrete skill patches.
when_to_use: when actively using another skill and noticing ambiguities, gaps, or places where the skill's instructions were insufficient or wrong
version: 1.1.0
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

As you work with the primary skill, note observations openly in the conversation when you encounter them. The context window IS the scratchpad — reasoning traces are already visible, so there's no need for a separate hidden phase. Just call out friction as you hit it:

```
SKILL_UPDATE observation: [TYPE] <what happened>
```

Types:
- `[AMBIGUITY]` — skill was unclear, had to interpret
- `[GAP]` — skill doesn't cover this, but should
- `[WRONG]` — skill's guidance produced a bad result
- `[TIGHT]` — constraints too restrictive for the task
- `[LOOSE]` — more structure was needed
- `[ASKED_HUMAN]` — question asked that skill should have answered
- `[JUDGMENT]` — decision made that the skill should encode
- `[PATTERN]` — reusable pattern discovered during work

**Keep delivering the primary task.** Observations are inline notes, not interruptions.

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
| Stop working to write long reflections | Brief inline note, keep delivering |
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
Use skill → hit friction → note inline → keep working
    ↓
Deliver primary output
    ↓
Distill observations → draft patches
    ↓
Propose to user → apply approved changes
    ↓
Next use benefits from improvements
    ↓
Repeat (skills converge toward robustness)
```

## Example Session

**Primary skill**: canvas-diagrams (L3 data flow)
**Task**: Map Beeper API fields → storage decisions

**Inline observations as they happened:**

> Building Beeper canvas... the skill says L3 is "5-10 nodes" but I've got 24 fields to map.
> `SKILL_UPDATE observation: [TIGHT] L3 node count too restrictive for field-level mapping`

> Need to show discarded fields in red but the skill only has component-type colors.
> `SKILL_UPDATE observation: [GAP] No color convention for retained vs discarded data fate`

> Using horizontal layout — vertical spine doesn't work for source→destination comparison.
> `SKILL_UPDATE observation: [PATTERN] Data-mapping needs 3-column horizontal, not vertical spine`

**Resulting patches (after delivery):**
- Expand L3 to include "data-field mapping" variant (L3b, 15-30 nodes)
- Add data-fate edge colors (green=retained, red=discarded)
- Add horizontal 3-column layout for mapping canvases
- Add "Discarded" group concept for explicit non-storage

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
