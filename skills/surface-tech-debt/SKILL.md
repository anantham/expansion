---
name: Surface Tech Debt
description: Systematic codebase review for technical debt - pattern conflicts, test gaps, band-aid fixes, security issues, and documentation holes. Use after rapid development sprints or before major refactors.
when_to_use: when user says "tech debt", "review codebase", "what needs cleanup", "surface issues", "code health", or after a period of rapid development
version: 1.3.0
---

# Surface Tech Debt

## Purpose

High-velocity development accumulates debt. This skill performs a systematic sweep across five dimensions of technical health, surfacing issues that compound over time. The goal isn't to fix everything — it's to make the debt **visible** so you can prioritize.

**Announce at start:** "Running tech debt surface scan across 7 dimensions: patterns, tests, band-aids, security, documentation, monolith growth, and complexity spirals."

## When to Use

- After a rapid development sprint (shipped fast, need to assess damage)
- Before a major refactor (understand what you're building on)
- Onboarding to a new codebase (map the landmines)
- Periodic hygiene (monthly/quarterly review)
- After discovering a bug that "shouldn't have happened"

## The Seven Dimensions

### Dimension 1: Pattern Conflicts

**Goal:** Find where different parts of the system follow conflicting patterns, creating friction and cognitive load.

**What to look for:**

| Conflict Type | Example | Why It Matters |
|---------------|---------|----------------|
| Dual patterns | Some routes use `async/await`, others use callbacks | Developers must context-switch; bugs at boundaries |
| Naming inconsistency | `get_user()` vs `fetch_user()` vs `retrieve_user()` | Harder to discover existing code; duplicates created |
| Data format conflicts | Some APIs return `{ok, data}`, others return raw data | Error handling becomes inconsistent |
| Import path fragmentation | Same module imported 3 different ways | Refactoring becomes treacherous |
| State management split | Some state in DB, some in memory, some in files | Race conditions, stale data, hard to reason about |

**Search patterns:**
```bash
# Find async/await vs callback mixing
grep -rn "async def" --include="*.py" | head -20
grep -rn "\.then\(" --include="*.py" | head -20

# Find naming pattern variations
grep -rn "def get_" --include="*.py" | wc -l
grep -rn "def fetch_" --include="*.py" | wc -l

# Find error response format variations
grep -rn '"ok":' --include="*.py" | head -10
grep -rn "return {" --include="*.py" | head -10
```

**Output format:**
```
PATTERN CONFLICT: <type>
LOCATIONS:
  - <file:line> uses pattern A
  - <file:line> uses pattern B
FRICTION: <what breaks or confuses>
RESOLUTION: <which pattern should win, or need new abstraction>
EFFORT: trivial | moderate | significant
```

### Dimension 2: Test Coverage Gaps

**Goal:** Identify code that's risky to change because it lacks automated verification.

**What to look for:**

| Gap Type | Risk Level | Detection |
|----------|------------|-----------|
| No tests for critical path | HIGH | Core business logic with 0 test files |
| Tests exist but don't assert | MEDIUM | Test files with no `assert` statements |
| Mocked so heavily it tests nothing | MEDIUM | More mocks than real code exercised |
| Integration seams untested | HIGH | DB/API boundaries with no integration tests |
| Error paths untested | HIGH | `except` blocks never exercised |
| Edge cases undocumented | MEDIUM | No tests for boundary conditions |

**Search patterns:**
```bash
# Find files with no corresponding test
find . -name "*.py" -path "*/core/*" | while read f; do
  base=$(basename "$f" .py)
  if ! find . -name "test_${base}.py" -o -name "${base}_test.py" | grep -q .; then
    echo "UNTESTED: $f"
  fi
done

# Find test files with no assertions
grep -L "assert" tests/test_*.py

# Find heavily mocked tests
grep -c "@patch\|Mock\|MagicMock" tests/*.py | sort -t: -k2 -rn | head -10
```

**Output format:**
```
TEST GAP: <description>
FILE: <path>
RISK: high | medium | low
WHAT BREAKS IF WRONG: <consequence>
SUGGESTED TEST: <1-2 sentence description of what to test>
```

### Dimension 3: Band-Aid Fixes (Edge Solutions)

**Goal:** Find fixes that treat symptoms rather than root causes — code that should trigger architectural rethinking.

**The key question:** "Are we solving the same problem in multiple places?"

**What to look for:**

| Band-Aid Type | Signal | Real Fix |
|---------------|--------|----------|
| Repeated defensive checks | Same `if x is None` in 5 places | Fix the source that produces None |
| Format conversion layers | `str(x)` or `int(x)` scattered everywhere | Enforce types at boundaries |
| Retry wrappers everywhere | Every call wrapped in retry logic | Centralized retry middleware |
| String manipulation for IDs | Splitting/joining IDs to extract parts | Structured ID objects |
| Timeout increases | "Fixed by increasing timeout to 600s" | Fix the slow operation |
| Silent exception swallowing | `except: pass` or `except: log.debug()` | Handle or propagate properly |
| Feature flags as permanent fixtures | `if USE_NEW_X:` that's always True | Remove the old path |

**Search patterns:**
```bash
# Find repeated defensive patterns
grep -rn "if .* is None:" --include="*.py" | cut -d: -f3 | sort | uniq -c | sort -rn | head -10

# Find timeout band-aids
grep -rn "timeout" --include="*.py" | grep -i "increase\|bump\|raise"

# Find silent exception handling (exclude .venv)
grep -rn "except.*:" --include="*.py" -A1 | grep -v "\.venv" | grep -E "pass$|log\.(debug|info)"

# Find repeated string manipulation
grep -rn "\.split\(" --include="*.py" | cut -d: -f1 | sort | uniq -c | sort -rn | head -5
```

**The upstream question:** For each band-aid found, ask:
- Why does this condition exist?
- Who/what creates the bad state?
- Can we fix it there instead?

**Output format:**
```
BAND-AID: <what the code does>
LOCATIONS: <N occurrences across M files>
ROOT CAUSE: <why this keeps happening>
UPSTREAM FIX: <where to actually solve it>
ADR NEEDED: yes | no
EFFORT: trivial | moderate | significant | architectural
```

### Dimension 4: Security Review (Red Team)

**Goal:** Think like an attacker. What can be exploited?

**Attack surface checklist:**

| Vector | What to Check | Search Pattern |
|--------|---------------|----------------|
| Injection | User input reaching shell/SQL/eval | `subprocess`, `os.system`, `eval`, `exec`, raw SQL |
| Auth bypass | Missing auth checks on routes | Routes without `@requires_auth` or token validation |
| Secrets in code | Hardcoded API keys, passwords | `password=`, `api_key=`, `secret=`, `token=` |
| Path traversal | User input in file paths | `open(user_input)`, `Path(user_input)` |
| SSRF | User-controlled URLs fetched | `requests.get(user_url)`, `fetch(user_url)` |
| Sensitive data exposure | PII in logs, error messages | `log.*password`, `log.*token`, `log.*email` |
| Insecure defaults | Debug mode, permissive CORS | `DEBUG=True`, `CORS(*)`, `verify=False` |

**Search patterns:**
```bash
# Command injection vectors (exclude .venv)
grep -rn "subprocess\|os.system\|os.popen" --include="*.py" | grep -v "\.venv"

# SQL injection (raw queries with f-strings)
grep -rn 'execute.*f"' --include="*.py"
grep -rn "execute.*%" --include="*.py"

# Hardcoded secrets
grep -rn "password\s*=\s*['\"]" --include="*.py"
grep -rn "api_key\s*=\s*['\"]" --include="*.py"

# Sensitive data in logs
grep -rn "log.*password\|log.*token\|log.*secret" --include="*.py"

# Insecure defaults
grep -rn "verify=False\|DEBUG.*True" --include="*.py"
```

**Output format:**
```
SECURITY: <vulnerability type>
SEVERITY: critical | high | medium | low
FILE: <path:line>
VECTOR: <how an attacker exploits this>
REMEDIATION: <specific fix>
```

### Dimension 5: Documentation Gaps

**Goal:** Find missing ADRs, unclear tradeoffs, and unmarked shortcuts.

**What to look for:**

| Gap Type | Signal | Why It Matters |
|----------|--------|----------------|
| Feature without ADR | Major feature, no `docs/adr/` entry | Decisions lost, can't understand "why" |
| Orphaned TODOs | `TODO` comments older than 3 months | Either do it or delete it |
| Magic numbers | Hardcoded `42`, `1000`, `0.85` without explanation | Future devs can't tune safely |
| Implicit contracts | Function expects specific format, not documented | Breaks silently on bad input |
| Stale comments | Comment says X, code does Y | Worse than no comment |
| Unmarked shortcuts | Code that "works but shouldn't" | Time bomb for refactoring |

**Search patterns:**
```bash
# Find TODOs (potential debt markers)
grep -rn "TODO\|FIXME\|HACK\|XXX" --include="*.py" | head -30

# Find magic numbers
grep -rn "= [0-9]\{2,\}" --include="*.py" | grep -v "port\|timeout\|version"

# Find features without ADRs
ls docs/adr/*.md | xargs -I{} basename {} .md | sort > /tmp/adrs
ls grimoire/IndrasNet/agents/*.py | xargs -I{} basename {} .py | sort > /tmp/agents
comm -23 /tmp/agents /tmp/adrs

# Find stale comments (comments with old dates)
grep -rn "# .*202[0-4]" --include="*.py"
```

**ADR audit:**
- List all major features/subsystems
- Check if each has an ADR
- For existing ADRs, check if they're current (decisions may have evolved)

**Output format:**
```
DOC GAP: <what's missing>
TYPE: adr | todo | magic-number | stale-comment | implicit-contract
LOCATION: <file:line or feature name>
IMPACT: <what goes wrong without this doc>
ACTION: <write ADR | delete TODO | add constant | update comment>
```

### Dimension 6: Monolith Growth

**Goal:** Catch files that have grown too large and need splitting before they become unmaintainable.

**Why this matters:** Monoliths get split, then regrow. Without automated detection, you only notice when it's painful again.

**Thresholds:**

| LOC | Status | Action |
|-----|--------|--------|
| >300 | Warning | Consider splitting |
| >500 | Alert | Must split before adding more |
| >800 | Critical | Stop — split first |

**What to look for:**

| Pattern | Signal | Problem |
|---------|--------|---------|
| Growing server files | `web_server.py`, `app.py` at 500+ LOC | Should be bootstrap only |
| Agent monoliths | Single agent file doing client + processing + state | Split by responsibility |
| God components | React component at 1000+ LOC | Extract sub-components |
| Route sprawl | Routes mixed with business logic | Routes should be thin |
| Config accumulation | Config file doing validation + parsing + defaults | Split concerns |

**Search patterns:**
```bash
# Find large Python files (Unix/Mac)
find . -name "*.py" -not -path "*/test*" -not -path "*/__pycache__/*" -not -path "*/.venv/*" -not -path "*/node_modules/*" | \
  xargs wc -l | sort -rn | head -20

# Find large TypeScript/React files
find . -name "*.tsx" -o -name "*.ts" | grep -v node_modules | \
  xargs wc -l | sort -rn | head -20

# Cross-platform: Check specific known-risk files directly (works on Windows)
# Adapt paths to your project structure:
wc -l agents/web_server.py agents/beeper_ingestion_agent.py core/reprocessing_worker.py
wc -l indras-ui/src/App.tsx indras-ui/src/ExecutionPane.tsx

# Find routes defined outside route files (exclude .venv)
grep -rn "@app\.\|@router\." --include="*.py" | grep -v "/routes/" | grep -v "\.venv"
```

**IMPORTANT:** Always exclude `.venv/`, `node_modules/`, and `__pycache__/` from scans to avoid noise from dependencies.

**Route organization check:**
- New API endpoints should go in `agents/routes/<domain>.py`
- `web_server.py` should only do: app setup, middleware, mount routers
- If routing logic is in the main server file, flag it

**Output format:**
```
MONOLITH: <filename>
LOC: <count> (threshold: <which threshold exceeded>)
LAST SPLIT: <date if known, or "never">
GROWTH SINCE: <LOC added since last check, if trackable>
SUGGESTED SPLIT:
  - <component 1> → <new file>
  - <component 2> → <new file>
EFFORT: trivial | moderate | significant
```

**Tracking growth over time:**
If `docs/indrasnet/ISSUES_AND_GAPS.md` exists, check the "Monoliths to Split" section and compare current LOC to documented LOC.

**Growth Detection Protocol:**
For each file over threshold:
1. **If in ISSUES_AND_GAPS.md:** Compare current LOC to documented LOC
   - Flag if >20% growth (regression detected)
   - Update the documented LOC after scan
2. **If NOT in ISSUES_AND_GAPS.md:** Flag as "NEW MONOLITH - not tracked" (high priority)
   - Add to the tracking doc immediately

**Growth output format:**
```
GROWTH: beeper_ingestion_agent.py
DOCUMENTED: 1222 LOC (ISSUES_AND_GAPS.md)
CURRENT: 2704 LOC
DELTA: +1482 (+121% growth) ← 🔴 CRITICAL REGRESSION

NEW MONOLITH: reprocessing_worker.py
LOC: 1750 (threshold: 800 CRITICAL)
STATUS: Not in ISSUES_AND_GAPS.md — add to tracking!
```

### Dimension 7: Complexity Spirals

**Goal:** Find places where the original problem was simple, but instead of fixing the root cause, layers of compensating mechanisms accumulated. Each layer often defends against bugs in the previous layer. The codebase grows by addition where it should have grown by replacement.

**The diagnostic question:** *"If I deleted N of these layers, would anything actually break?"*

**The discipline:** Before adding any new defensive mechanism, try to **delete two existing ones** first. If you can't delete two, you don't understand the problem yet.

**Patterns to look for:**

| Spiral Type | Smell | Real Fix |
|---|---|---|
| Multiple recovery paths | Boot task + watchdog + auto-restart + port guard + retry loops, all "making sure X runs" | Pick one supervisor, delete the others |
| Dead module shims | `foo.py` (file) AND `foo/` (package) both exist; `.py` is "for compat" but every internal caller updated | Update imports, delete the shim |
| Dead replaced code | Old impl kept "in case" — zero importers, zero callers | `git rm` the file |
| Dual stores for same domain | JSON contacts AND SQLite contacts; settings in env AND yaml AND DB | Migrate callers to one, delete the other |
| Defensive checks for impossible states | `if x is None` scattered everywhere because upstream contract is unclear | Fix the upstream contract, delete the defenses |
| Wrappers around wrappers | A→B→C→D, each with try/except + default | Inline the layers, delete try/except where the failure can't actually happen |
| Config fallback chains repeated N times | `get_setting(X) or os.getenv(Y, "literal default")` copy-pasted in 6+ places | One helper function, called everywhere |
| Timeout band-aids | "Fixed by raising timeout to 600s" — multiple times, never fixing the slow thing | Profile and fix the slow thing |

**Search patterns:**

```bash
# Stub/shim files: a .py file AND a directory of the same name (likely a refactor that left scaffolding)
find . -name "*.py" -not -path "*/.venv/*" -not -path "*/__pycache__/*" | while read f; do
  base="${f%.py}"
  if [ -d "$base" ]; then
    lines=$(wc -l < "$f")
    if [ "$lines" -lt 200 ]; then
      echo "SHIM CANDIDATE: $f ($lines lines) coexists with package $base/"
    fi
  fi
done

# Repeated config fallback chains (same default copy-pasted)
grep -rn 'get_setting.*or os\.getenv' --include="*.py" | head -20
grep -rn 'os\.getenv.*"http://localhost' --include="*.py" | sort | uniq -c | sort -rn | head -10

# Multiple "make sure X runs" mechanisms — look for restart/watchdog/supervisor/recover keywords
grep -rln "watchdog\|restart_server\|auto_restart\|crash_count\|MAX_RESTART" --include="*.py" --include="*.bat" --include="*.ps1"

# Dead modules: count imports of modules that look "deprecated"
for module in privacy_filter privacy_engine privacy_transformer; do
  count=$(grep -rn "from.*$module\|import $module" --include="*.py" | grep -v "$module.py\|$module/" | wc -l)
  echo "$module: $count importers"
done

# Defensive None checks — by file, sorted (high count = upstream contract unclear)
grep -rn "if.*is None" --include="*.py" | cut -d: -f1 | sort | uniq -c | sort -rn | head -10
```

**The upstream question:** For each spiral, ask:
- What was the ORIGINAL problem (one sentence)?
- How many layers exist now?
- Which layer would break if removed? (often: none)
- What's the smallest delete that consolidates them?

**Output format:**
```
COMPLEXITY SPIRAL: <one-sentence description>
ORIGINAL PROBLEM: <what we were actually trying to solve>
LAYERS NOW (N):
  1. <file:line> — <what it does>
  2. <file:line> — <what it does, often defending against #1's bug>
  3. <file:line> — ...
DELETE PROPOSAL: <which N-1 layers to remove, what survives>
LOC RECLAIMED: ~<estimate>
RISK: low | medium | high (when in doubt, low — the layers were never load-bearing)
```

**Worked examples (real findings from the IndrasNet codebase):**

1. **5-layer server-restart spiral.** Original problem: "make sure web_server on port 7777 stays up." Layers: (a) Boot Task Scheduler entry, (b) every-5-min Watchdog Task with broken `Get-Process` cleanup, (c) `start_all.py` exponential-backoff auto-restart, (d) `start.bat` port-busy guard, (e) `web_server`'s own `check_single_instance` PID file. Reduced to 3 by deleting the watchdog (113 LOC + 1 task). The watchdog was "defending against" failures that didn't exist; its own bugs caused the duplicate-launch incident that prompted its rewrite.

2. **5+ shim files for refactored packages.** Original problem: "split monolith into a package." Resolution: re-export shim left in place (`web_server.py` → `web_server/`, `media_router.py` → `media_router/`, `context_assembler.py` → `context_assembler/`, `media_transcription.py` → `media_transcription/`, `auth.py` → `auth/`). Each shim says "DEPRECATED" but every internal caller still imports through it. Real fix: update imports, `git rm` the shim.

3. **3 dead privacy modules.** Original problem: "redact private content." `privacy_filter.py` (308 LOC, 0 callers), `privacy_engine.py` (rarely called), `privacy_transformer.py` (different concern entirely). All three exist because each "replaced" the previous without deleting it. Real fix: pick the one in active use, delete the rest.

4. **Beeper URL config repeated 6×.** Original problem: "where do we read the Beeper URL from?" Pattern `get_setting("beeper_api_url") or os.getenv("BEEPER_URL", "http://localhost:23374")` copy-pasted in 6 files. If the default changes, you edit 6 places. Real fix: one `get_beeper_config()` helper, ~20 LOC of duplication eliminated.

5. **Dual contact stores.** JSON-backed `core/contacts.py` (privacy paths) and SQLite `core/db/contacts.py` (everything else). Privacy decisions and prayer routing operate on potentially divergent contact data — same name, different tier in each store. Real fix: migrate the 3 privacy callers to the SQLite version, delete `core/contacts.py` and `config/contacts.json`.

**Anti-pattern to recognize in yourself when designing fixes:**

You discover bug X in layer 3. Tempting fix: add layer 4 to detect bug X and work around it. **Stop.** Better questions:
- Why does layer 3 exist? Does it solve a problem, or just defend against layer 2?
- If layers 2 and 3 disappeared, what would actually break?
- Can the bug be eliminated by deleting layers, not adding them?

**Often the right answer is the smallest one:** delete the broken thing rather than fix it. If `start_all.py`'s built-in restart is sufficient for crash recovery, the watchdog's reason to exist disappears — fixing the watchdog's CIM detection misses the real question.

---

## Process

### Phase 1: Automated Scan

Run searches across all seven dimensions. Use grep, find, and static analysis to surface candidates. This should take 5-10 minutes.

**Don't analyze everything — collect raw signals.**

### Phase 2: Triage

For each category, sort findings by:
1. **Severity** — How bad if this bites us?
2. **Frequency** — How often does this code path run?
3. **Effort** — How hard to fix?

Focus report on top 3-5 items per dimension.

### Phase 3: Report

Present findings in priority order with actionable next steps.

```
# TECH DEBT SURFACE SCAN

## Executive Summary
- **Pattern Conflicts:** N issues (M critical)
- **Test Gaps:** N untested critical paths
- **Band-Aids:** N fixes that need upstream resolution
- **Security:** N issues (M high/critical)
- **Documentation:** N missing ADRs, M stale TODOs
- **Monoliths:** N files over threshold
  - 🔴 CRITICAL (>800 LOC): M files
  - 🟡 ALERT (>500 LOC): N files
  - ⚪ WARNING (>300 LOC): O files
- **Complexity Spirals:** N spirals (M with 4+ layers — top priority for deletion)

## Critical Items (Fix This Sprint)
1. <issue> — <1-line description> — <effort>
2. ...

## High Priority (Fix This Month)
1. ...

## Tech Debt Backlog (Track for Later)
1. ...

---

## Detailed Findings

### Pattern Conflicts
<detailed findings>

### Test Gaps
<detailed findings>

### Band-Aid Fixes
<detailed findings>

### Security Issues
<detailed findings>

### Documentation Gaps
<detailed findings>

### Monolith Growth
<detailed findings>

### Complexity Spirals
<detailed findings — for each spiral list the layers + delete proposal>

---

## Recommended Actions
1. <action> — <owner> — <effort>
2. ...
```

## Integration with Development Workflow

### When to Run

| Trigger | Focus |
|---------|-------|
| After feature sprint | Band-aids, test gaps for new code |
| Before major refactor | Pattern conflicts, implicit contracts |
| Quarterly review | Full scan, all dimensions |
| After security incident | Security dimension deep dive |
| New team member onboarding | Documentation gaps |

### Output Artifacts

- **GitHub Issues:** Create issues for critical/high items
- **ADR drafts:** For architectural band-aids that need design discussion
- **Test plan:** List of tests to add for coverage gaps
- **Refactor proposals:** For pattern unification

## Anti-Patterns

| Don't | Do Instead |
|-------|-----------|
| Try to fix everything found | Prioritize ruthlessly, track the rest |
| Report without severity | Every item needs severity + effort |
| Flag style issues as debt | Focus on correctness and maintainability |
| Ignore "it works" code | Working code with debt is still debt |
| Run only once | Make this a regular hygiene practice |
| Skip the upstream question | Every band-aid deserves root cause analysis |

## Checklist

- [ ] Ran automated scans across all 6 dimensions
- [ ] Triaged findings by severity and effort
- [ ] Identified top 3-5 items per dimension
- [ ] For each band-aid, asked "where's the upstream fix?"
- [ ] Checked ADR coverage for major features
- [ ] Flagged security issues with severity ratings
- [ ] Checked file sizes against LOC thresholds (300/500/800)
- [ ] Compared monolith sizes to ISSUES_AND_GAPS.md (if exists)
- [ ] Produced prioritized report with actionable items
- [ ] Created tracking issues for items to address

## Example: The Telemetry Fix

**What happened:** Telemetry messages were being truncated/broken because Telegram has message size limits, but the system was sending unbounded content.

**Band-aid detected:** Multiple places doing string truncation or catching send failures.

**Upstream analysis:** The real fix is in the Beeper actuation agent — handle large messages gracefully by chunking them *before* they hit the Telegram API, not by catching failures after.

**This is the mental model:** When you see the same defensive code in multiple places, trace it back to the source and fix it there.
