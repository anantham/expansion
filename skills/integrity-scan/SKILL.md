---
name: integrity-scan
description: Audit whether the code tells the truth about itself — names that lie (validate_* that doesn't validate, get_* with side effects), docstrings and comments claiming behavior or guarantees the code no longer has, comments citing mechanisms that exist but are neutralized, one concept under many names, the same logic forked under different names, and band-aid layers each sensible in isolation that together form unnecessary complexity. Use after high-velocity or AI-assisted development, before onboarding someone, or when the code "reads fine" but keeps surprising people.
when_to_use: when user says "integrity scan", "inconsistent naming", "the docstrings lie", "comments are stale", "unnecessary complexity", "layers of patches", "this code reads fine but surprises me", "incoherent", or after a sprint of AI-generated code
version: 1.0.1
---

# Integrity Scan

## Purpose

Code carries claims about itself: a function's **name** claims what it does, its
**docstring** claims a contract, a **comment** claims why. When those claims drift from
behavior, every future reader — human or model — is confidently misled. Nothing fails; the
tests stay green; the wrong mental model propagates.

High-velocity and AI-assisted development produce this drift *characteristically*, not
randomly:

- A model writes a **plausible** docstring for code it half-implemented — plausible prose
  is what LLMs are best at, so the docstring is the *most* trustworthy-looking artifact and
  the least verified.
- Each session names the same concept slightly differently (`fetch`/`get`/`retrieve`/`load`;
  `user`/`account`/`owner`/`principal`), because a fresh context can't see the convention.
- Each fix adds a **defensive layer** rather than changing the layer that's wrong — every
  one justified in isolation, together an onion nobody can reason about.
- Comments accrete referencing mechanisms that have since been renamed, disabled, or
  silently neutralized.

This scan finds claims that are no longer true. It is read-only and produces a prioritized
report; the human decides whether to fix the code or the claim.

**Announce at start:** "Running integrity scan across 7 signatures: lying names, docstring
divergence, stale/neutralized comments, concept synonyms, forked logic, band-aid layering,
and stale scaffolding."

## When to Use

- After a sprint where much of the code was model-generated.
- Before onboarding anyone (they will trust the docstrings; find out if that's safe).
- When a bug's post-mortem is "but the comment said it was handled."
- When the codebase "reads fine" yet keeps surprising people — the signature of drifted
  claims rather than missing tests.

**Boundaries (avoid overlap):**
- `doc-audit` → **doc files** (README/ADR/module docs) vs code. This skill → **in-code**
  claims: names, docstrings, inline comments.
- `guard-sprawl` → competing *enforcement* of one invariant (config, guards, thresholds).
  This skill → whether the code's *self-description* is honest. They pair: sprawl usually
  leaves behind comments claiming guarantees the sprawl broke.
- `test-audit` → whether tests are meaningful. This skill doesn't judge tests, but a
  docstring claiming a guarantee **only** a test asserts is a finding here.
- `surface-tech-debt` → broad triage; run it to decide whether to run this.

## The Seven Signatures

### Signature 1: Names that lie

A name is an API. When it misdescribes behavior, callers write correct-looking wrong code.

```bash
# Predicate/getter names with side effects or mutation
grep -rnE "def (is_|has_|can_|should_|get_|check_|validate_|verify_|ensure_)[a-z_]*\(" \
  --include="*.py" . | grep -viE "test_|\.venv" | head -40
# For each hit, read the body and ask the questions below.
```

Confirm by reading, then judge:
- `is_*` / `has_*` / `can_*` → returns a bool and **mutates nothing**? (a predicate that
  writes state is the highest-severity lie in this signature)
- `get_*` → no I/O, no cache-fill, no side effects? (`get_user()` that creates a user is a
  trap)
- `validate_*` / `verify_*` / `check_*` → does it actually **reject** anything, or only log?
  A validator that never fails is a **no-op wearing a safety name** — cross-check whether
  callers treat its return value as a gate.
- `ensure_*` → idempotent?
- Negations: `disable_x = True` meaning enabled, `skip_y` that includes.

### Signature 2: Docstring / comment ↔ code divergence

The docstring is the contract. Check it against the body — especially **guarantee words**.

```bash
# Docstrings making strong claims: verify EACH against the body.
grep -rnE "guarantee|guaranteed|always|never|ensures|atomic|thread-safe|idempotent|\
sanitiz|validated|verified|cannot|must not|safe to" --include="*.py" . \
  | grep -viE "test_|\.venv" | head -50
```

For every guarantee word, find the code that implements it. Common divergences:
- "**never** returns None" — and there's a bare `return` on an error path.
- "**atomic**" — with two separate writes and no transaction.
- "**thread-safe**" — no lock anywhere in the module.
- "**validated/sanitized before X**" — the check covers a narrow subset (an allowlist of
  a handful of patterns) while the prose implies full coverage.
- Documented parameters/returns that no longer exist in the signature.

> **Real instance:** a module docstring said user data was *"mechanically verified before
> upload"*. The mechanical layer checked exact display-names plus a hardcoded 7-string
> allowlist; the real work was done by an LLM whose output was unverifiable. The prose
> implied a guarantee the code never had — and an operator-facing UI note repeated it.

**A documented limitation is NOT a finding.** A docstring that honestly says *"this tests
the redaction spec via a fake, NOT the production transformer"* is good engineering. Only
flag a claim that is **stronger** than the code. Verify the claim, not your first
impression of the file.

### Signature 3: Comments citing mechanisms that are gone or neutralized

The cruelest class: the comment names a real, existing safety net — which no longer works.

```bash
# Comments that point AT something. Then verify the target still does what's implied.
grep -rnE "#.*(handled by|see |guarded by|covered by|protected by|watchdog|fallback|\
retry|elsewhere|upstream|downstream|the .* will)" --include="*.py" . \
  | grep -viE "test_|\.venv" | head -40
```

For each: does the named mechanism (a) still exist, (b) still cover this case, (c) actually
run in production? Follow the reference — do not accept it.

> **Real instance:** a comment read *"the off-loop RAM watchdog handles the emergency >88%
> case independently."* The watchdog **did** exist and **did** trigger at 88% — but an
> exemption list excluded the exact model that caused the outage. The comment was true
> about existence and false about coverage, so every reader (including the agent
> diagnosing the incident) stopped looking there.

Also check `skip`/`xfail` markers whose comment promises a replacement that never landed —
**a skipped test is where a guard goes to die.**

### Signature 4: One concept, many names (synonym sprawl)

```bash
# Verb synonyms for the same operation
for v in get fetch retrieve load read lookup find query; do
  printf "%-9s %s\n" "$v" "$(grep -rlE "def ${v}_" --include='*.py' . 2>/dev/null | grep -vcE 'test_|\.venv')"
done
# Noun synonyms for the same entity — adjust the list per domain
for n in user account owner principal member profile; do
  printf "%-10s %s\n" "$n" "$(grep -rohE "\b${n}_(id|name|key)\b" --include='*.py' . 2>/dev/null | wc -l)"
done
```

Multiple live synonyms mean every reader must learn the mapping, and models will invent a
new synonym on the next edit. **Pick one, document it in `CONVENTIONS.md`, migrate the
rest.** Where a distinction is real (`fetch` = network, `get` = in-memory), the fix is to
*write that rule down*, not to merge them.

### Signature 5: Forked logic — the same behavior implemented N times

```bash
# Duplicate-ish helper names across modules
grep -rhoE "def [a-z_]{6,}\(" --include="*.py" . | sort | uniq -c | sort -rn | head -25
# Repeated literal patterns that suggest copy-paste (tune the regex to your domain)
grep -rnE "re\.(sub|match|search)\(r?[\"'][^\"']{12,}" --include="*.py" . \
  | grep -viE "test_|\.venv" | awk -F: '{print $3}' | sort | uniq -c | sort -rn | head -15
```

Ask which fork **production actually reaches**. The dangerous shape is a canonical, correct
implementation that is *dead*, while live callers each roll their own.

> **Real instance:** the one strictMode-aware validator in a codebase was reachable only
> from two dead provider files and a test re-export. Three shipped adapters each forked it —
> so the suite "covering" that logic exercised code production never ran, while every live
> path was untested and subtly wrong (one fork's over-escaped regex matched zero markers
> and silently duplicated every one). **Fix shape:** find the chokepoint all callers
> already flow through, promote the canonical implementation into it, delete the forks.

### Signature 6: Band-aid layering / abstraction inversion

Each layer was added for a real bug. Together they are unnecessary complexity.

```bash
# Wrappers around wrappers: functions that only delegate.
# NB use [(] [)] not \( \) — in -E mode BSD/macOS grep reads escaped parens as a group
# and dies with "parentheses not balanced"; and [[:space:]] not \s (\s is GNU-only).
grep -rnA3 "def [a-z_]*[(]" --include="*.py" . \
  | grep -B1 -E "^[[:space:]]*return [a-z_]+[(].*[)]$" \
  | grep -viE "test_|\.venv" | head -30
# Defensive re-checks: the same guard repeated down a call chain
grep -rnE "if .* is None:\s*return|if not .*:\s*return( None)?$" --include="*.py" . \
  | grep -viE "test_|\.venv" | wc -l
# "Just in case" / "shouldn't happen" markers — each marks a layer nobody understood
grep -rnE "#.*(just in case|shouldn't happen|should never|defensive|belt and braces|\
paranoia|safety net|double check|extra guard)" --include="*.py" . | head -25
```

For each defensive check, ask: **is the condition it guards actually reachable?** An
unreachable guard is dead weight that implies a danger that doesn't exist — future readers
preserve it forever. Prove reachability before keeping.

> **Real instance:** a proposed "hardening" turned out to guard a state that was
> structurally impossible (`s is None` where the callee never returns None), *and* the exit
> codes it set had **no consumer**. Retracted rather than shipped. Check for a consumer
> before adding a signal.

### Signature 7: Stale scaffolding

```bash
grep -rnE "TODO|FIXME|XXX|HACK|temporary|for now|will be removed|deprecated|legacy" \
  --include="*.py" . | grep -viE "\.venv|node_modules" | head -40
# Age them — a 2-year-old "temporary" is permanent, and its comment is a lie.
# (Set f=<path>; a literal <file> placeholder makes bash attempt a redirection.)
f=path/to/file.py; git log -1 --format="%ar" -S "TODO" -- "$f"
```

Also: version-stamped comments (`# as of v2.1`) long past, dated notes whose date has
passed, and "temporary" code older than the feature it patched. Either fix, delete, or
restate honestly with a current date and an owner.

## Process

1. **Scope** — pick the subsystem (whole-repo output is unreadable). Prefer where surprises
   happened recently, or the highest-traffic module.
2. **Collect candidates** with the greps above. They are *candidate* lists, never findings.
3. **Confirm by reading.** Every finding requires the claim and the contradicting code, both
   quoted. Never report from a grep hit alone — the false-positive rate is high by design.
4. **Never truncate a candidate list into a clean verdict.** If a list is too long, sort by
   relevance and state in the report **which files were not examined**. A subset-scoped
   check silently passes everything excluded.
5. **Triage** by reader-harm: does this claim cause someone to write wrong code?
6. **Report**; do not auto-fix.

## Severity

| Sev | Meaning | Example |
|---|---|---|
| **P0** | The claim causes unsafe action | "validated before upload" over a narrow subset; a validator that never rejects but is treated as a gate |
| **P1** | Comment cites a real but neutralized safety net | "the watchdog handles >88%" while an exemption excludes the culprit |
| **P2** | Name/docstring misleads about behavior | `get_*` that writes; "never returns None" that can |
| **P3** | Coherence tax | synonym sprawl, forked helpers, dead defensive layers, stale TODOs |

## Report format

```markdown
## Scope
<subsystem> — <N> files examined; NOT examined: <list or "none">

## P0 — claims that cause unsafe action
### <file:line> — <the claim, quoted>
**Code says:** <quoted contradicting code>
**Reader harm:** <the wrong code someone writes because of this>
**Fix:** change the code / change the claim — <which, and why>

## P1 / P2 / P3 — same shape

## Coherence summary
- Concept synonyms: <concept> appears as <a>/<b>/<c> — canonical: <proposal>
- Forked logic: <behavior> implemented in <N> places; live path: <which>; chokepoint: <where>
- Dead defensive layers: <count> guards on unreachable conditions
```

## Guidance

- **A claim is a testable artifact.** "Documentation" and "comments" sound soft; a docstring
  saying *never* is a falsifiable assertion. Treat it like one.
- **Fix the claim OR the code — decide deliberately.** Weakening prose to match sloppy code
  is sometimes right (honesty beats aspiration) and sometimes a cover-up. Say which you
  chose and why.
- **Don't flag honest disclosure.** A documented limitation is the *opposite* of this defect
  class. Read for whether the claim is stronger than the code, not for whether the code is
  imperfect.
- **Prefer deleting a layer to adding a comment explaining it.** Band-aid layering is not
  fixed by documenting the onion.
- **Dogfood your detection.** Before reporting a signature clean, run it against a place you
  already know has the defect and confirm it fires. An audit that can't reproduce a known
  instance is measuring nothing.
- **The greps here are instruments — verify them before trusting a zero.** Three silent
  failure modes, all of which shipped in this skill's first draft and were caught only by
  executing every block: a **quoted regex split across lines** needs a trailing `\` (else
  the newline joins the pattern, grep exits 2, and you get zero hits — the same pattern on
  one line found 603); **`\(`/`\)` inside `-E` are not portable** (BSD/macOS grep: "parentheses
  not balanced" — use `[(]`/`[)]`); **`\s` is GNU-only** (use `[[:space:]]`). A broken
  instrument does not error loudly — it returns a clean bill of health, which is the exact
  defect class this skill exists to find.

## Related Skills

- **`guard-sprawl`** — the *structural* sibling: N competing mechanisms for one invariant.
  Sprawl leaves behind exactly the Signature-3 comments this skill finds.
- **`doc-audit`** — doc **files** vs code (this skill: in-code claims).
- **`doc-writer`** — writes the honest replacement once findings are triaged; owns
  `CONVENTIONS.md`, where Signature-4 canonical names belong.
- **`test-audit`** — a guarantee asserted only by a vacuous test is a P0 here.
- **`surface-tech-debt`** — broad triage that points at this scan.
