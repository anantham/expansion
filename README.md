# expansion

A [Claude Code](https://docs.claude.com/en/docs/claude-code) plugin: a small collection of
**skills** that extend Claude Code with documentation, code-health, testing, and
session-workflow capabilities. Each skill is a self-contained `SKILL.md` that Claude loads
on demand when the task matches — or that you invoke explicitly with `/expansion:<name>`.

- **Repository:** https://github.com/anantham/expansion
- **License:** MIT

## Skills

| Skill | Invoke | What it does |
|-------|--------|--------------|
| **doc-audit** | `/expansion:doc-audit` | Read-only audit that compares docs against code — surfaces stale docs, undocumented modules, naming conflicts, invalidated proofs, and convention violations. |
| **doc-writer** | `/expansion:doc-writer` | Writes module docs with staleness markers, design rationale, and bottom-up decision capture; can create `CONVENTIONS.md`. Pairs with doc-audit. |
| **doc-prover** | `/expansion:doc-prover` | Creates formal `ASSERTION` proof files that cite specific code lines for invariants, security boundaries, privacy, or correctness — and re-verifies them when code changes. |
| **surface-tech-debt** | `/expansion:surface-tech-debt` | Systematic codebase review across seven dimensions: pattern conflicts, test gaps, band-aid fixes, security, docs, monolith growth, and complexity spirals. |
| **test-audit** | `/expansion:test-audit` | Audits the *quality* of an existing test suite (not just its existence) — vacuous/change-detector/over-mocked/unfocused/unreadable tests, weak values, non-hermetic tests, and risk gaps. Grounded in the Google Testing Blog. |
| **postmortem** | `/expansion:postmortem` | After a silent bug, traces root cause through git history, identifies human–agent miscommunication, and searches for downstream data damage. |
| **canvas-diagrams** | `/expansion:canvas-diagrams` | Generates Obsidian Canvas (`.canvas`) files for UX flows, data flows, and testing-infrastructure visualization. |
| **handover** | `/expansion:handover` | Graceful context transfer before session end or compaction — commits work, documents pending threads, captures learnings, prepares the next instance. |
| **skill-update** | `/expansion:skill-update` | Meta-skill: learns from skill usage, tracks uncertainties and human interventions, and proposes concrete patches to the skills themselves. |
| **meta-update** | `/expansion:meta-update` | The `/meta-update` ritual: promotes a session's durable cross-project learnings to a shared union store — a tagged mistakes ledger, a pulled-on-demand library with an index, and a journal — plus per-project auto-memory, guarding against silent sync-conflict loss. (Distinct from `/mu`, which aliases skill-update.) |

The doc skills are designed to compose: **doc-audit** finds gaps → **doc-writer** fills them →
**doc-prover** locks down the invariants that matter. **surface-tech-debt** and **test-audit**
are complementary code-health passes (the former asks *"is there a test?"*, the latter *"is the
test any good?"*).

## Install

In Claude Code, add this repo as a plugin marketplace and install the plugin:

```
/plugin marketplace add anantham/expansion
/plugin install expansion@expansion
```

`/plugin marketplace add` accepts the `owner/repo` shorthand for GitHub (or the full
`https://github.com/anantham/expansion.git` URL). Once installed, the skills register
automatically; restart or reload if a newly added skill doesn't appear.

## Use

Skills trigger two ways:

1. **Automatically** — Claude reads each skill's `description` and invokes the matching one
   when your request fits (e.g. "audit the docs", "review tech debt", "are these tests any
   good?").
2. **Explicitly** — type `/expansion:<name>`, e.g. `/expansion:test-audit`.

Most skills are **read-only diagnostics by default**: they produce a prioritized report and
leave the fixes to you (or to a follow-up skill) unless you ask them to apply changes.

## Repository layout

```
.
├── .claude-plugin/
│   ├── plugin.json         # plugin manifest (name, version, metadata)
│   └── marketplace.json    # marketplace manifest pointing at this repo
├── skills/
│   └── <skill-name>/
│       └── SKILL.md        # one self-contained skill (frontmatter + body)
├── AFFORDANCES.md          # catalog of reusable cross-project tools
├── .gitattributes          # normalizes line endings to LF (Syncthing-safe)
└── README.md
```

Each `skills/<name>/SKILL.md` has YAML frontmatter (`name`, `description`, optional
`when_to_use` and `version`) followed by the skill instructions in Markdown. The skill's
**invocation name is its directory name**, so `skills/test-audit/` is `/expansion:test-audit`.

## Developing a skill

1. **Edit here, in this repo** — `skills/<name>/SKILL.md`. (Do *not* edit the installed copy
   under `~/.claude/plugins/cache/...`; it is overwritten on every plugin update.)
2. To add a skill, create `skills/<new-name>/SKILL.md` with frontmatter and a `description`
   that clearly states *when* Claude should reach for it — that description is the only thing
   the auto-trigger sees.
3. **Bump the version** in `.claude-plugin/plugin.json` (and keep `marketplace.json` in sync)
   so installs pick up the change.
4. Commit and push; users re-run `/plugin install expansion@expansion` (or update) to get it.

### Versioning

`plugin.json` and `marketplace.json` both carry a `version`; keep them equal. Bump the minor
version when adding a skill, the patch version for fixes within a skill. Individual skills may
also carry their own `version:` in frontmatter for finer-grained tracking.

## License

MIT — see `plugin.json`.
