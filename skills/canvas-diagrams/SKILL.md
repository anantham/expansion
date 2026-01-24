---
name: Canvas Diagrams
description: Generate Obsidian Canvas (.canvas) files for UX flows, data flows, and testing infrastructure visualization
when_to_use: when user asks to visualize architecture, flows, data pipelines, testing infrastructure, or says "canvas", "diagram", "flow chart", "visualize"
version: 1.0.0
---

# Canvas Diagrams

## Overview

Generate Obsidian `.canvas` files that visually represent system architecture, user flows, data pipelines, and testing infrastructure. These are collaborative artifacts — Claude generates them, the user can manually adjust layout in Obsidian, and Claude can read them back to understand current architecture.

**Announce at start:** "Using Canvas Diagrams skill to generate a visual [type] canvas."

## Target Directory

```
/Users/aditya/Library/CloudStorage/GoogleDrive-adityaprasadiskool@gmail.com/My Drive/Exocortex/Research/Projects/TemporalCoordination
```

**Naming convention:** `<type>-<scope>[-detail].canvas`

Examples at each resolution:
- `data-flow-overview.canvas` (system level)
- `data-flow-prayer-pipeline.canvas` (subsystem level)
- `data-flow-context-assembly.canvas` (function level)

## Resolution Levels

**Generation order: BOTTOM-UP.** Create detailed subsystem canvases first, then aggregate into an overview that embeds them.

### Level 1: System Overview
- **5-8 nodes max** — major subsystems as single nodes (not groups with children)
- **Purpose:** Skimmable at a glance — "what are the moving parts?"
- **Node content:** Title + 1 line max. No code references, no function names.
- **Edges:** Major data/control flows only (3-8 edges total)
- **Embeds:** Each node links to its detail canvas via `![[filename.canvas]]` in the node text
- **Generated LAST** after all Level 2 canvases exist

### Level 2: Subsystem Detail
- **8-15 nodes** — functions, tables, agents within one subsystem
- **Purpose:** "How does this subsystem work internally?"
- **Node content:** Function names, file:line refs, in/out types
- **Edges:** Internal data transformations, branching logic
- **Generated FIRST** — this is the primary working canvas

### Level 3: Function/Step Detail
- **5-10 nodes** — individual steps within one function or process
- **Purpose:** "What happens inside this function?"
- **Node content:** Code snippets, parameter details, branching conditions
- **Edges:** Sequential steps, conditional branches, error paths
- **When:** A function is complex enough to warrant decomposition

### Cross-linking (Canvas embedding)

**Use `type: "file"` nodes** to embed sub-canvases — NOT `![[...]]` syntax in text nodes (that's for markdown notes only).

```json
{"id": "embed_subsystem", "type": "file", "file": "vault/relative/path/to/sub-canvas.canvas", "x": 0, "y": 100, "width": 400, "height": 300}
```

**Pattern for overview canvases:**
- Text label node (title + 1-line description) positioned above
- File embed node (the sub-canvas preview) positioned below the label
- Edges connect between the embed nodes

**Vault-relative paths** (from Exocortex vault root):
- `Research/Projects/TemporalCoordination/<filename>.canvas`

## Canvas Types

### 1. UX Flow (`ux-flow-*.canvas`)
Maps user journeys through the system.

**Node categories & colors:**
| Element | Color | Meaning |
|---------|-------|---------|
| User action | `"4"` (green) | Something the user does |
| System response | `"5"` (cyan) | What the system shows/does |
| Decision point | `"2"` (orange) | Branch in the flow |
| Error/edge case | `"1"` (red) | Failure states |
| External trigger | `"6"` (purple) | Incoming webhook, notification |

### 2. Data Flow (`data-flow-*.canvas`)
Maps data sources, transformations, API calls, and storage.

**Node categories & colors:**
| Element | Color | Meaning |
|---------|-------|---------|
| Data source | `"4"` (green) | External APIs, webhooks, user input |
| Transform/function | `"5"` (cyan) | Processing logic |
| Storage | `"6"` (purple) | Database, file system |
| API endpoint | `"2"` (orange) | Exposed routes |
| LLM call | `"3"` (yellow) | AI processing step |
| Error/retry | `"1"` (red) | Failure handling |

### 3. Testing Infrastructure (`testing-infra-*.canvas`)
Maps what's tested, how, and where gaps exist.

**Node categories & colors:**
| Element | Color | Meaning |
|---------|-------|---------|
| Tested (passing) | `"4"` (green) | Has tests, they pass |
| Tested (failing) | `"1"` (red) | Has tests, they fail |
| Untested | `"2"` (orange) | No test coverage |
| Test file | `"5"` (cyan) | The test file itself |
| Mock/fixture | `"6"` (purple) | Test infrastructure |
| CI/runner | `"3"` (yellow) | How tests are executed |

## Canvas JSON Format

**Important Obsidian behaviors:**
- Obsidian reorders nodes (groups first) and compacts JSON on save
- Omit fields with null/default values — Obsidian strips them anyway
- Edge `fromEnd`/`toEnd` default to no arrow; only specify when you want arrows
- When reading back a user-edited canvas, trust Obsidian's reformatted positions

```json
{
  "nodes": [
    {"id": "group_id", "type": "group", "x": -50, "y": -50, "width": 700, "height": 400, "color": "5", "label": "Group Label"},
    {"id": "unique_snake_case_id", "type": "text", "text": "# Node Title\n\nDescription.\n\n`file_path:line`", "x": 0, "y": 0, "width": 300, "height": 150, "color": "4"}
  ],
  "edges": [
    {"id": "edge_unique_id", "fromNode": "source_node_id", "fromSide": "bottom", "toNode": "target_node_id", "toSide": "top", "color": "3", "label": "verb"}
  ]
}
```

**Node field order** (matches Obsidian's output): `id, type, text|label, x, y, width, height, color`
**Edge field order**: `id, fromNode, fromSide, toNode, toSide, color, label`

## Layout Rules

### Spacing Philosophy
**Breathe.** Nodes need white space to be skimmable. Err on the side of too spread out — the user can zoom in Obsidian but can't un-overlap nodes easily.

### Grid System
- **Column width:** 450px (300px node + 150px gap)
- **Row height:** 400px (180px node + 220px gap)
- **Group padding:** 50px around contained nodes
- **Between-group gap:** 400-600px vertically (Obsidian needs breathing room)
- **Between-group gap (horizontal branch):** 500-800px when subsystem branches off

### Spatial Layout
- **Main pipeline:** flows top-to-bottom (vertical spine)
- **Branching subsystems:** extend horizontally (to the right or left of spine)
- **Don't stack everything vertically** — if a subsystem is its own domain, give it its own spatial region
- **Example:** Ingestion flows down, prayer processing branches right, execution flows back down

### Node Sizing

**Title must never wrap.** The `# heading` line determines minimum width. Body text can be clipped (user scrolls in Obsidian), but the title must be fully visible on one line.

**Width calculation (title-driven):**
- Obsidian heading font: ~14px per character at `#` level
- `width = max(200, len(title_text) * 14 + 40)`
- Example: `# beeper_ingestion_agent` (24 chars) → min 376px wide
- Example: `# Cache Check` (13 chars) → min 222px wide
- Cap at 420px — if title is longer, abbreviate it

**Height calculation (body-driven):**
- Obsidian renders ~30px per line of markdown (header line is ~40px)
- `height = 50 + (num_body_lines * 28)`
- Body text below the fold is OK — user can scroll inside the node
- But the title + first 2 lines should always be visible

**Practical sizes:**
| Content | Width | Height |
|---------|-------|--------|
| Title only | 250-350w | 60h |
| Title + 1-2 lines | 280-380w | 100-120h |
| Title + 3-5 lines + code ref | 320-420w | 160-200h |
| Title + bullet list (6-8 items) | 350-420w | 220-280h |

### Color as Primary Visual Affordance

**Colors should be scannable at a glance.** A viewer should understand the system's structure from colors alone, before reading any text.

**Principles:**
1. **Color = role, not decoration.** Every color choice must answer "what category is this?"
2. **Consistent across all canvases** in a domain (same color for "storage" everywhere)
3. **Groups inherit color** from their dominant node type
4. **Edges get colored** to show data type or flow category — not left as default gray
5. **High contrast pairs** for connected elements (green source → yellow processing → purple storage)

**Edge coloring:**
| Edge meaning | Color |
|--------------|-------|
| Primary data flow | `"4"` (green) |
| LLM/AI processing | `"3"` (yellow) |
| Storage write | `"6"` (purple) |
| API/HTTP call | `"2"` (orange) |
| Error/retry path | `"1"` (red) |
| Control flow (triggers) | `"5"` (cyan) |

**Group coloring:**
- Groups should use the same color as their dominant node category
- This creates visual "zones" that are identifiable even when zoomed out

### Edge Crowding (validated)

**#1 rule: Drop obvious labels.** If node names imply the relationship, use color-only edges.

**Strategies:**
1. **Color-only edges** — color communicates category (green=data in, cyan=processing, yellow=dispatch, purple=storage). No label needed.
2. **Fan-in spread** — when N nodes connect to 1 target, use different `toSide` values (`left`, `top`, `right`) so edges arrive at different points.
3. **Fan-out spread** — when 1 node connects to N targets, use different `fromSide` values to radiate edges instead of bundling.
4. **Perimeter routing** — for edges that would cross the whole canvas, connect from sides that route around content (e.g., `fromSide: "right"` → `toSide: "left"` to go around the right edge).
5. **Only label non-obvious edges** — if you must label, max 1-2 words, active verb.

### Visual Verification

After generating a canvas, run the PNG preview script to check for:
- Title text exceeding node width
- Edge label collisions (shared midpoints)
- Nodes outside their containing group bounds
- Color distribution — are zones visually distinct?

```python
# Quick overlap/preview check (run from project root)
.venv/bin/python3 scripts/canvas_preview.py <canvas_file>
```

### Edge Sides
Use logical connection points:
- Top-to-bottom flow: `fromSide: "bottom"`, `toSide: "top"`
- Left-to-right flow: `fromSide: "right"`, `toSide: "left"`
- Bidirectional: Use two edges or `fromEnd: "arrow"`, `toEnd: "arrow"`

## Node Content Format

### Standard Node
```
# Short Title

Brief description (1-2 sentences).

`path/to/relevant/file.py:42`
```

### Decision Node
```
# Decision: [Question]

**Yes →** [outcome]
**No →** [outcome]
```

### API/Function Node
```
# function_name()

**In:** param types
**Out:** return type
**Side effects:** what it changes

`path/to/file.py:123`
```

## Process (Bottom-Up)

### When creating a new domain's canvases:

1. **Explore** the relevant codebase area (use Task agents, read files, check existing docs)
2. **Identify subsystems** — what are the 3-6 major components? Each becomes a Level 2 canvas.
3. **For each subsystem** (Level 2):
   a. Identify nodes: functions, tables, agents, API calls
   b. Determine relationships (data flow, triggers, dependencies)
   c. Layout with generous spacing, branching subsystems horizontally
   d. Generate and write the `.canvas` file
4. **After all Level 2s exist**, generate the Level 1 overview:
   a. One node per subsystem, title + 1 line only
   b. Embed link: `![[subsystem-canvas.canvas]]` in each node
   c. Minimal edges showing major flows between subsystems
5. **Report** what was generated, list all files created

### When updating an existing canvas:

1. Read the `.canvas` file (respect user's manual layout adjustments)
2. Identify what changed in the codebase
3. Add/modify/remove specific nodes and edges
4. Preserve user-adjusted positions for unchanged nodes

## Reading Existing Canvases

When asked to update or discuss an existing canvas:
1. Read the `.canvas` file from the target directory
2. Parse the JSON to understand current state
3. Propose additions/changes as specific node/edge modifications
4. Write the updated canvas

## Collaborative Workflow

- **Claude generates** → User adjusts layout in Obsidian → Claude reads back
- **Version tracking**: Git tracks the vault, so canvas evolution is preserved
- **Incremental updates**: Don't regenerate from scratch — read existing canvas and add/modify nodes
- **Comments in nodes**: Use `> [!note]` callouts in node text for Claude-to-human notes

## Edge Labels

Use short, active verbs:
- `triggers`, `calls`, `returns`, `stores`, `reads`, `transforms`
- `on success`, `on error`, `if true`, `if false`
- `via webhook`, `via API`, `via cron`

## Checklist

- [ ] Explored relevant code to identify all components
- [ ] Chose correct canvas type and color scheme
- [ ] Nodes have meaningful IDs (snake_case, descriptive)
- [ ] Every node has a title and brief description
- [ ] Code references use `file:line` format
- [ ] Edges have descriptive labels
- [ ] Layout follows grid system (no overlapping nodes)
- [ ] Groups logically cluster related nodes
- [ ] Written to correct target directory with proper naming
- [ ] Reported to user what was generated
