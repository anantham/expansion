---
name: guard-sprawl
description: Find scattered enforcement of a single invariant — N competing guards for one rule, exemption flags that defeat several at once, one setting under several env-var names/defaults, one resource with several definitions, overlapping thresholds owned by different components, and settings whose meaning depends on which process reads them. Use when config feels duplicated, when a documented setting mysteriously didn't apply, after an incident where "the safety net didn't fire", or before refactoring a subsystem that has accumulated guards.
when_to_use: when user says "competing configurations", "why are there so many ways to do this", "band aid after band aid", "complexity spiral", "the guard didn't fire", "config drift", "duplicate guards", "this setting didn't take effect", or before consolidating a subsystem
version: 1.0.0
---

# Guard Sprawl

## Purpose

A system accumulates guards. Each is added after a real incident, at the site where the
pain appeared, by whoever was on it. **Nothing gets deleted**, because removing a guard
requires proving it's subsumed — expensive and scary — while adding one is cheap and feels
responsible.

The result is not defense in depth. It is **defense in breadth**: N mechanisms each
covering ~80% of an invariant, with the gaps in *different* places, and one exemption flag
that punches through several of them at once.

This skill finds that pattern *before* the next incident, and it finds the sharpest form
of it: **a setting whose meaning depends on which process reads it.**

**Announce at start:** "Running guard-sprawl audit across 6 signatures: mechanism count,
exemption flags, setting aliases, resource definitions, threshold triangulation, and
config-visibility-by-ancestry."

## When to Use

- An incident where "the watchdog/guard/limit didn't fire" — especially if it *logged*
  something reassuring while the system was failing.
- A documented setting that provably had no effect.
- Before consolidating or refactoring a subsystem that has grown guards over time.
- The user says any variant of *"why do we have competing configurations"* or
  *"band aid after band aid"*.

**Distinct from `surface-tech-debt`:** that skill sweeps broadly across many debt
dimensions. This one goes deep on exactly one failure class — scattered enforcement of a
single invariant — and produces a consolidation plan. Run this when you already suspect
sprawl; run that one to discover what to suspect.

## Step 0 — Name the invariant in ONE sentence

Everything depends on this. Write the rule the system is supposed to uphold, as a single
declarative sentence, before scanning:

> "A local model must never exhaust host RAM."
> "A request must never reach the DB without an authenticated user."
> "Money must never move twice for one order."

If you cannot write it in one sentence, the sprawl is worse than you think — the invariant
itself has drifted. Write the two or three sentences it has become, and audit each.

## The Six Signatures

### Signature 1: Mechanism count — how many things enforce one rule?

Enumerate every mechanism that exists to uphold the Step-0 sentence: timeouts, thresholds,
watchdogs, admission checks, retry caps, circuit breakers, cleanup jobs, cron tasks.

```bash
# Guards usually name themselves. Sweep for the vocabulary, then read each hit.
grep -rnE "watchdog|guard|_limit|threshold|max_|timeout|evict|reap|prune|circuit|
backoff|admission|quota|throttle" --include="*.py" --include="*.ts" --include="*.go" . \
  | grep -viE "test_|_test\.|\.min\.|node_modules|\.venv" | head -60

# Then: which of these run as SEPARATE processes? (cron/systemd/Task Scheduler/launchd)
grep -rnE "crontab|systemd|schtasks|launchd|LaunchAgent|CronJob|celery.beat|APScheduler" \
  --include="*.py" --include="*.ps1" --include="*.sh" --include="*.yaml" . | head -20
```

**Record a table:** mechanism · file:line · what triggers it · what it covers · what it
*cannot* see. That last column is where incidents live.

**Verdict:** more than ~3 mechanisms for one invariant is sprawl. More than 6 means the
next incident is already scheduled.

### Signature 2: Exemption flags — the multiplier

An exemption (allowlist / keep-warm / skip / pinned / protected / ignore) is the highest-
leverage defect in a sprawled system, because **one exemption can defeat every mechanism
at once** — and it usually reads as a thoughtful optimisation.

```bash
grep -rnE "keep_warm|keepalive|KEEP_|allowlist|allow_list|whitelist|skip_|_skip|
exempt|exclude|ignore_|never_|pinned|protect|no_evict|do_not|permanent" \
  --include="*.py" --include="*.ts" . | grep -viE "test_|node_modules|\.venv" | head -40
```

For each exemption found, answer in writing:
1. **Which mechanisms does it defeat?** (check every one from Signature 1)
2. **Is it a safety guarantee or a performance preference?** A *preference* that outranks
   a *safety* mechanism is a bug, always. Latency is recoverable; an exhausted host is not.
3. **Is its default value the dangerous one?** An exemption that is opt-*out* rather than
   opt-*in* applies on hosts nobody tuned.

> **Real instance:** `LMSTUDIO_KEEP_WARM_MODELS` defaulted to a ~12 GB vision model. That
> single default made the model permanently resident (a 317-year idle timeout) *and* exempt
> from two independent pressure-eviction paths. A 33 GB host wedged at 94% RAM with every
> relief mechanism structurally unable to touch it.

### Signature 3: Setting aliases — one concept, many names

```bash
# Same conceptual setting reached by different env var names.
grep -rhoE "getenv\(\"[A-Z_]+\"|environ\[\"[A-Z_]+\"\]|process\.env\.[A-Z_]+" \
  --include="*.py" --include="*.ts" . | grep -oE "[A-Z_]{3,}" | sort -u > /tmp/envnames.txt
# Then cluster by shared token — aliases share a stem (LMS_CLI / LMS_CLI_PATH / LMS_BIN).
awk '{n=split($0,p,"_"); for(i=1;i<=n;i++) if(length(p[i])>2) print p[i]"\t"$0}' \
  /tmp/envnames.txt | sort | awk '{c[$1]++; v[$1]=v[$1]" "$2} END{for(k in c) if(c[k]>1) print c[k], k":"v[k]}' \
  | sort -rn | head -25
```

Also check **defaults disagreeing across call sites** — the more dangerous half:

```bash
# same-ish name, different fallback value
grep -rnE "getenv\(\"[A-Z_]+\",\s*[\"r]" --include="*.py" . | grep -viE "test_|\.venv" | head -40
```

> **Real instance:** one CLI binary was reached via `LMS_CLI_PATH`, `LMS_CLI`, `LMS_BIN`,
> and a hardcoded literal — four call sites, three env names, two defaults (two of them a
> bare command name, PATH-dependent). Setting any one of them fixed exactly one call site,
> and the bare ones failed during the incident when PATH differed.

### Signature 4: Resource definitions — one endpoint/path/table, many literals

```bash
# Pick the resource (a port, base URL, DB path, bucket, queue name) and count its definitions.
RES="1234"   # <- change per resource
grep -rn "$RES" --include="*.py" --include="*.ts" --include="*.yaml" --include="*.json" . \
  | grep -viE "test_|node_modules|\.venv|lock" | head -30
```

Look for **shape drift**, not just count: `localhost` vs `127.0.0.1`, with/without a
`/v1` suffix, with/without the route appended, trailing slash present/absent. Shape drift
means a migration must find *all* variants, and normalisation bugs hide in the difference.

> **Real instance:** one local inference endpoint had six definitions across five files,
> mixing `localhost`/`127.0.0.1`, with and without `/v1`, one with the full chat route.

### Signature 5: Threshold triangulation — several numbers, one physical resource

```bash
# Numeric thresholds on the same resource, owned by different components.
grep -rnE "(PCT|PERCENT|THRESHOLD|_GB|_MB|LIMIT|MAX_|HIGH_|CRIT|PANIC|WARN)[A-Z_]*\s*=" \
  --include="*.py" . | grep -viE "test_|\.venv" | head -40
```

Group the hits by the resource they measure. Three thresholds on one resource owned by
three components means no component knows the whole picture, and the *gaps between them*
are unmonitored.

> **Real instance:** RAM/memory pressure had a load-admission threshold at 90%, a watchdog
> at 88%, and a panic-kill at 94% — three owners, one resource. The incident sat at 94% RAM
> / 80% commit: the panic gate keyed on *commit*, so it never fired, and the watchdog that
> keyed on *RAM* was neutralised by the Signature-2 exemption.

### Signature 6: 🎯 Config visibility by process ancestry — the root class

**This is the signature most audits miss, and the one that produces "but I set that!"**

A config file (`.env`, `settings.yaml`) is usually loaded by *one* entry point. Anything
that entry point imports inherits the values. Anything that runs as a **separate process**
— cron job, scheduled task, systemd unit, sidecar, CLI tool — does **not**, unless it loads
the config itself. Both call `getenv` identically, so the code looks correct in both.

```bash
# 1. Who actually LOADS config?
grep -rn "load_dotenv\|dotenv\|readFileSync(.*\.env\|config.load\|Settings()" \
  --include="*.py" --include="*.ts" . | grep -viE "test_|\.venv|node_modules"

# 2. Which files are ENTRY POINTS? (separate processes — cron/task/service/CLI)
grep -rln "__main__\|argparse\|click\.command\|#!/usr/bin" --include="*.py" . \
  | grep -viE "test_|\.venv" | head -30

# 3. THE GAP: entry points that read config but never load it.
#    NOTE: do NOT pipe the candidate list through `head` — see the warning below.
for f in $(grep -rl "__main__" --include="*.py" . | grep -viE "test_|\.venv|node_modules"); do
  if grep -q "getenv" "$f" && ! grep -qE "load_dotenv|dotenv|_load_env|config\.load" "$f"; then
    echo "GAP: $f — $(grep -c getenv "$f") getenv calls, loads config: NO"
  fi
done
```

**⚠️ Never truncate the candidate list in this step.** A `| head -N` on the entry-point
list means everything past N is silently unexamined and the check reports clean — a
subset-scoped check hands a **free pass** to the excluded subset. This happened while
dogfooding this very skill: a `head -12` excluded the exact watchdog whose
config-blindness caused the incident, and the audit printed "no gaps". If the list is
genuinely too long to review, sort it by relevance and say in the report **which files
were not examined** — never let truncation masquerade as a clean result.

For every gap, ask: **does this setting mean the same thing in every process?** If not,
that is the bug — regardless of whether it has bitten yet.

> **Real instance, the sharpest finding of the whole audit:** the config file explicitly
> set the exemption list to *empty* ("pin nothing"). But the watchdog ran as a standalone
> scheduled task, never loaded that file, and fell back to its own hardcoded default —
> pinning a 12 GB model. **The watchdog protected the model against the operator's own
> configuration**, and the config looked correct to every human who read it.
>
> Corollary: for a standalone process, **the code default is the only value you can rely
> on reaching it.** Fixing such a bug via the config file is a no-op.

## ⚠️ The trap: you will try to add mechanism N+1

Mid-audit, the obvious fix for "the guard didn't fire" is to **add a guard**. Resist it.

> **Real instance:** while diagnosing the incident above, the auditing agent added a
> RAM-percentage check to one watchdog — not knowing a *second* watchdog already did
> exactly that, at the same 88% threshold. It would have shipped mechanism #11, overlapping
> #7, still missing the exemption that caused the outage. Caught only by a cross-family
> code review.

**The spiral reproduces itself during its own diagnosis.** Before writing any new guard:

1. Search for an existing mechanism that already covers this trigger (Signature 1 table).
2. If one exists, **fix it** rather than adding beside it.
3. If you truly must add one, name in the commit message **what it subsumes — and delete
   that.**

## Report format

```markdown
## Invariant
"<the one sentence from Step 0>"

## Mechanisms (Signature 1)
| # | Mechanism | Location | Triggered by | Covers | Cannot see |

## Exemptions (Signature 2)  ← highest leverage
| Flag | Default | Mechanisms it defeats | Safety or preference? | Verdict |

## Duplication (Signatures 3–5)
- Setting aliases: <name> reached as <A>, <B>, <C> — defaults <x> vs <y>
- Resource definitions: <resource> defined N times; shape drift: <...>
- Thresholds: <resource> gated at <a>% / <b>% / <c>% by <owners>

## Config visibility (Signature 6)
| Entry point | Separate process? | Loads config? | Settings it silently ignores |

## Consolidation plan
1. One owner: <module> owns the invariant; everything else calls it.
2. Delete: <mechanisms subsumed by the owner>
3. One name per concept: <canonical env var / constant>
4. Config parity: <how standalone processes get the same values>
5. Enforce at the boundary, not per-call-site: <the chokepoint>
```

## Guidance

- **Surface, don't shred.** This is a read-only audit that produces a plan. Deleting a
  guard needs proof of subsumption; the human decides.
- **Prefer removing an exemption over adding a mechanism.** The 2-line change that deletes
  an exemption usually beats the 200-line guard.
- **Prove each fix red.** A consolidation guard that passes both before and after your
  change tests nothing — run it against the pre-fix code and watch it fail first.
- **Don't consolidate during an incident.** Land the minimal fix that kills the class, then
  do the boundary refactor in its own PR with tests. Rushing the refactor while production
  is down is how the spiral deepens.
- **Chokepoint pattern:** find the one path every caller *already* flows through, promote
  the canonical implementation into it, and delete the forks. If no such path exists,
  creating it is the refactor.
- **Dogfood your own detection.** Before reporting "no findings" for a signature, run that
  signature against a place you *already know* has the defect and confirm it fires. An
  audit that cannot reproduce a known instance is measuring nothing — and a silent `head`,
  a stale path, or a too-narrow `--include` will produce a confident clean bill of health.
  Expected-red first, then trust the greens.

## Related Skills

- **`surface-tech-debt`** — broad multi-dimension sweep; run it to *discover* sprawl.
- **`postmortem`** — when sprawl already caused an incident, trace the damage.
- **`doc-audit`** — sprawled subsystems almost always have docs claiming guarantees the
  code no longer provides (a comment referencing a watchdog that exists but is neutralised
  is the classic tell).
