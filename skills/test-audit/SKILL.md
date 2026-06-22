---
name: test-audit
description: Audit the QUALITY of an existing test suite (not just its existence) — finds tests that can't fail, change-detector/over-mocked tests, unfocused tests, unreadable tests, unactionable failures, weak test values, non-hermetic tests, and risk/coverage gaps. Grounded in the Google Testing Blog / "Testing on the Toilet" canon. Use after a test sprint, before trusting a suite, or when green tests still ship bugs.
when_to_use: when user says "audit tests", "test quality", "are these tests any good", "review the tests", "why did green tests miss this bug", "test smells", or before relying on a suite for a refactor
version: 1.0.0
---

# Test Audit

## Purpose

A passing test suite is not the same as a *good* test suite. Tests can be green and still
worthless — or worse, **negative value**: they break on every refactor without catching a
single real bug. This skill audits the **quality** of the tests that already exist, finds
the ones that lie, and tells you which to rewrite, narrow, split, or delete.

It is the test-focused complement to `surface-tech-debt` (whose Dimension 2 only asks *"is
there a test?"*). This skill asks the harder question: **"if the code broke, would this test
notice — and would the failure tell me what broke?"**

Every dimension below is grounded in a specific Google Testing Blog / Testing-on-the-Toilet
(TotT) post; sources are listed at the bottom so findings are citable, not opinion.

**Announce at start:** "Running test-quality audit across 8 dimensions: vacuous tests, brittle/change-detector tests, unfocused tests, unreadable tests, unactionable failures, weak values, non-hermetic tests, and risk/coverage gaps."

## When to Use

- After a test-writing sprint (coverage jumped — but is it real coverage?)
- A green suite shipped a bug it "should have" caught → find the other lying tests
- Before relying on the suite to refactor safely (are tests brittle change-detectors?)
- Flaky CI you suspect is the tests' fault, not the code's
- Onboarding to an unfamiliar codebase — judge how much to trust its tests
- Periodic hygiene, same cadence as `doc-audit` / `surface-tech-debt`

**Not for:** writing tests or fixing the findings (this is read-only diagnostic — propose,
don't auto-fix unless asked); pure coverage-gap hunting where no tests exist yet (use
`surface-tech-debt` Dimension 2); production-code smells unrelated to testability (use
`surface-tech-debt`).

## The Core Lens

From *Test Failures Should Be Actionable* — the unifying meta-principle: a test earns its
keep only if **(a)** it fails when the behavior breaks (not before), and **(b)** its
name + failure message alone tell you what broke. Most dimensions below are a specific way a
test violates (a) or (b).

```
                 Does it fail when, and only when, the BEHAVIOR breaks?
                 /                                                    \
              NO                                                      YES
             /   \                                                    /  \
   never fails    fails on refactor              when it fails, is the reason clear?
   (D1 Vacuous)   (D2 Brittle)                        /                      \
                                                     NO                       YES
                                          (D5 Unactionable failures)      ← keep it
```

## Detection Strategy (first 5 minutes)

Before scoring individual tests, get the lay of the land:

```bash
# 1. Find the test files (adapt globs to the stack)
#    pytest/unittest:  test_*.py  *_test.py        jest/vitest: *.test.ts *.spec.ts
#    JUnit:            *Test.java  Test*.java       Go: *_test.go    gtest: *_test.cc
find . \( -path '*/node_modules/*' -o -path '*/.venv/*' -o -path '*/vendor/*' \) -prune -o \
  \( -name 'test_*.py' -o -name '*_test.py' -o -name '*.test.*' -o -name '*.spec.*' \
     -o -name '*Test.java' -o -name '*_test.go' -o -name '*_test.cc' \) -print

# 2. Detect the frameworks/assert idioms in use (drives the grep patterns below)
grep -rhoE '(import pytest|from unittest|@Test|testing\.T|gtest|describe\(|it\(|test\()' \
  --include='*.py' --include='*.java' --include='*.go' --include='*.ts' --include='*.cc' . | sort | uniq -c

# 3. Per test file, count tests vs assertions vs mock-setup — the cheap triage signal
#    (low asserts-per-test → D1; high mock/verify density → D2/D6)
```

Then walk the 8 dimensions. **Read a representative sample** of flagged tests — grep finds
candidates, but a human/agent read confirms the smell. Don't report a grep hit as a finding
without reading it.

## The Eight Dimensions

### D1 — Vacuous Tests (can't fail)

**The worst kind: a green test that asserts nothing real.** It exists to pad coverage and
lull you. The canonical trap is the *default-value* coincidence — a test that passes because
its expected value happens to equal the type's default (`0`, `""`, `false`, `null`, enum[0]),
so a method that stores nothing still "passes."

| Smell | Signal to grep / look for |
|-------|---------------------------|
| No assertion at all | Test function body with zero `assert`/`expect`/`EXPECT_`/`require.` calls |
| Default-value coincidence | Expected value is `0` / `""` / `false` / `None`/`null` / enum index 0 — and so is the type default |
| Tautology | `assertEqual(x, x)`, `assertTrue(True)`, asserting a literal you just set |
| Asserts the mock, not the code | Only assertion is on a stubbed return value the test itself configured |
| `assert` with no effect | Python bare `assert` under `-O`; an `expect()` with no matcher; a Promise assertion never awaited |
| Exception-swallowing test | `try: ...; except: pass` around the act, so a thrown error passes silently |

```bash
# Python: test functions with no assert (approx — review hits)
grep -rLZ 'assert' --include='test_*.py' --include='*_test.py' . | xargs -0 -I{} echo "NO-ASSERT FILE: {}"
# Default/zero expectations
grep -rnE 'assert(Equal|Equals|Is)?\(.*, *(0|0\.0|""|''|False|None|null)\)' --include='*.py' --include='*.ts'
# Un-awaited async expectations (JS) — expect inside a promise without await/return
grep -rnE '\.then\(.*expect\(' --include='*.test.*' --include='*.spec.*'
# Bare except around the act
grep -rn -A2 'except' --include='test_*.py' . | grep -E 'pass$|continue$'
```
**Source:** *Choosing Values for Robust Tests*. **Severity:** usually **P0/P1** — a vacuous
test is worse than no test because it reads as coverage.

### D2 — Brittle / Change-Detector / Over-Mocked Tests

A test that breaks on a **pure refactor** (no behavior change) is coupled to implementation,
not behavior. The extreme is the *change-detector*: it mocks every collaborator and only
`verify`s that they were called in the order the production method calls them — it is a
restatement of the code, has **negative value**, and should be rewritten or deleted.

| Smell | Signal |
|-------|--------|
| `verify`-heavy, assert-light | High ratio of interaction checks (`verify`, `toHaveBeenCalled`, `assert_called_with`) to value/state assertions |
| One mock per collaborator + ordered verify | Mocks mirror the method body 1:1 |
| Over-mocking | A test mocks >1–2 classes, or one mock stubs >1–2 methods; setup longer than act+assert |
| Whole-object equality | `assertEqual(expectedObj, actual)` / `EXPECT_EQ(proto, expected)` — adding an unrelated field breaks many tests (prefer narrow, field-subset assertions) |
| Reaches into internals | Asserts on private fields/helpers, not the public return/behavior |
| Tests an implementation-detail class | A dedicated suite for a package-private/internal helper with a single caller, duplicating public-API coverage |
| Co-churn with refactors | Test files always change in lockstep with behavior-preserving refactor commits |

```bash
# Mock/verify density per test file (Python / JS / Java)
grep -rcE 'mock\(|@patch|MagicMock|when\(|thenReturn|\.toHaveBeenCalled|assert_called|verify\(' \
  --include='test_*.py' --include='*.test.*' --include='*Test.java' . | sort -t: -k2 -rn | head -20
# Whole-object equality assertions (brittle)
grep -rnE 'assert(Equal|Equals)\([A-Za-z_]+, *(expected|kExpected)[A-Za-z_]*\)' --include='*.py' --include='*.java'
grep -rnE 'EXPECT_EQ\([a-z].*, *[a-z].*\)' --include='*_test.cc'
# Stub density inside a single test (overuse-of-mocks signal)
grep -rnE 'when\(.*\)\.thenReturn|\.mockReturnValue|\.return_value *=' --include='*' .
```
**Sources:** *Change-Detector Tests Considered Harmful*; *Test Behavior, Not Implementation*;
*Don't Overuse Mocks*; *Prefer Testing Public APIs Over Implementation-Detail Classes*;
*Prefer Narrow Assertions in Unit Tests*. **Severity:** **P1** (P0 if it blocks a needed
refactor or is a pure change-detector).

### D3 — Unfocused Tests (more than one behavior)

A test should exercise **one behavior** (one scenario → one expected outcome). Tests are not
1:1 with methods. A test that calls the system, asserts, then calls it again and asserts
again is testing multiple scenarios — when it fails you can't tell which broke.

| Smell | Signal |
|-------|--------|
| Multiple act→assert cycles | More than one call to the system under test interleaved with assertions in one test |
| Named after a method | `testProcessTransaction` (method echo) instead of a behavior + outcome |
| Many unrelated assertion clusters | One test asserts UI text *and* email sent *and* DB write |
| Mid-test reconfiguration | Mutating config/limits partway through (e.g. `setOverdraftLimit`) to test a second case |

```bash
# Tests with many assertions (candidate unfocused) — count asserts per test region is hard
# with grep alone; use as a coarse signal then read the long ones:
grep -rcE '(assert|expect\(|EXPECT_)' --include='test_*.py' --include='*.test.*' . | sort -t: -k2 -rn | head -20
# Method-echo names (no scenario/outcome)
grep -rnE 'def test_?[A-Z][a-zA-Z]+\(self' --include='*.py'      # testCamelCaseMethodName
grep -rnE '(it|test)\(["'\'']test[A-Z]' --include='*.test.*'
```
**Sources:** *Test Behaviors, Not Methods*; *Keep Tests Focused*. **Severity:** **P2**
(P1 if the test is large and its failures are routinely ambiguous).

### D4 — Unreadable Tests (logic, over-DRY, hidden cause→effect)

Tests have no tests of their own, so they must be **obvious by inspection**. Prefer DAMP
(Descriptive And Meaningful Phrases) over DRY. Two failure modes: logic inside the test
(loops/conditionals/arithmetic can hide bugs), and cause separated from effect (setup buried
in shared `setUp`/fixtures far from the assertion).

| Smell | Signal |
|-------|--------|
| Logic in the test | `if`/`switch`/`for`/`while` inside a test body; expected value *computed* (string concat, arithmetic) instead of a literal |
| Over-DRY abstraction | Tests driven by shared `setUp` fields + private `_helper`s so the actual inputs aren't visible at the test site |
| Cause far from effect | `assertEqual(9, tally.get("k"))` whose `9` is produced by increments 200 lines away in `setUp` |
| Magic expected values | Hardcoded expected number/string with no locally visible reason |

```bash
# Loops/conditionals inside test bodies (read hits — some are legit table-driven setup)
grep -rnE '^[[:space:]]+(for |while |if |elif |switch ).*' --include='test_*.py' --include='*_test.go' --include='*.test.*'
# Computed expected values (string concat / arithmetic in an assertion)
grep -rnE 'assert.*(\+ *"|" *\+|\* [0-9]|/ [0-9])' --include='*.py' --include='*.ts'
# Heavy shared setup (cause hidden from effect)
grep -rncE '(def setUp|@(Before|BeforeEach)|beforeEach\()' --include='*' . | sort -t: -k2 -rn | head
```
**Sources:** *Tests Too DRY? Make Them DAMP!*; *Don't Put Logic in Tests*; *Keep Cause and
Effect Clear*. **Severity:** **P2**.

### D5 — Unactionable Failures (bad names + collapsed assertions)

When a test fails, name + message should be enough to start debugging. Two killers:
non-descriptive names, and **boolean-collapsing** assertions that throw away the actual value.

| Smell | Signal |
|-------|--------|
| Name lacks outcome | `testAdd`, `isUserLockedOut_invalidLogin` — scenario or method only, no expected outcome (`should…`, `…returns…`, `…throws…`) |
| Collapsed assertion | `assertTrue(a.equals(b))`, `assertTrue(x == y)`, `EXPECT_TRUE(foo().ok())` — failure says only "expected true, got false," hiding the value/error |
| Bare boolean on rich value | `assertTrue(list.contains(x))` instead of a containment matcher with a diff |
| No domain matchers | Equality/status checked via `assertTrue` instead of `assertThat`/Truth/`EXPECT_OK` |

```bash
# Collapsed boolean assertions hiding values
grep -rnE 'assert(True|False)\(.*(==|!=|\.equals\(|\.ok\(\)|in )' --include='*.py' --include='*.java'
grep -rnE 'EXPECT_(TRUE|FALSE)\(.*(==|\.ok\(\)|\.contains)' --include='*_test.cc'
# Test names with no expected-outcome clause (heuristic: no should/return/throw/when/then)
grep -rnE '(def test_|(it|test)\(["'\''])' --include='test_*.py' --include='*.test.*' \
  | grep -viE 'should|return|throw|raise|reject|when|then|given|_to_|_is_|_not_'
```
**Sources:** *Test Failures Should Be Actionable*; *Writing Descriptive Test Names*.
**Severity:** **P2** (P1 for collapsed assertions in critical paths — they cost real
debugging time on every failure).

### D6 — Weak Test Values (defaults, reused literals, no boundaries)

Even a well-structured test is blind if its **values** can't expose bugs. Pairs with D1.

| Smell | Signal |
|-------|--------|
| Default/zero values | The meaningful input/expected is the type default (`0`, `""`, `false`, `null`, enum[0]) |
| Same literal for distinct inputs | `insert(1, 1)` / same value for `key` and `value` → swapped/reused-arg bugs invisible |
| Single value per behavior | No boundary / empty / null / negative / large variants; no special-case coverage |
| No parameterization/fuzzing | Large input domains exercised with one hand-picked happy value |

```bash
# Same literal reused across args (review hits)
grep -rnE '\(\s*([0-9]+)\s*,\s*\1\s*\)' --include='test_*.py' --include='*.test.*'
# Meaningful zeros/empties as the value under test
grep -rnE '(insert|put|set|add|get|expect|assert)\w*\(.*(, *0\)|, *""\)|, *None\)|, *null\))' --include='*'
# Is parameterization used at all?
grep -rcE '@pytest.mark.parametrize|@parameterized|it.each|test.each|@RunWith\(Parameterized' --include='*' .
```
**Sources:** *Choosing Values for Robust Tests*. **Severity:** **P1/P2**.

### D7 — Non-Hermetic / Mis-Sized Tests (flaky-prone)

Classify tests by **how they run**, not by what they're called. A "unit" test that hits the
real network/DB/filesystem, sleeps, spawns threads, or depends on execution order is mis-sized
and a flake waiting to happen. Small = no network/DB/FS/sleep/threads; Medium = localhost +
DB/FS; Large = full system. All tests must be hermetic and order-independent.

| Smell | Signal |
|-------|--------|
| Hidden I/O in a "unit" test | Real socket/HTTP client, DB connection, file open, in a small test |
| Sleeps / wall-clock waits | `sleep`, `Thread.sleep`, `setTimeout` waits, polling for time |
| Non-determinism | Uses real `now()`/`Date.now()`/`rand`/UUID without injection/seed |
| Order dependence | Shared mutable/persistent test DB or global state; fails when run alone or reordered |
| Missing size tags | No small/medium/large (or equivalent) annotation to gate I/O |

```bash
grep -rnE '(time\.sleep|Thread\.sleep|sleep\(|await new Promise.*setTimeout)' --include='test_*.py' --include='*.test.*' --include='*Test.java'
grep -rnE '(requests\.(get|post)|http\.client|socket\.socket|urlopen|fetch\(|new Socket|jdbc:|psycopg|mysql\.connector|open\([^)]*[wra])' --include='test_*.py' --include='*_test.go' --include='*.test.*'
grep -rnE '(datetime\.now|time\.time\(\)|Date\.now|System\.currentTimeMillis|random\.|Math\.random|uuid4\()' --include='test_*.py' --include='*.test.*'
```
**Source:** *Test Sizes*. **Severity:** **P1** (flaky tests erode trust in the whole suite).

### D8 — Risk & Coverage Gaps

Tests are a means, not an end. Brainstorm the project's **key risks** and check the cheapest
tests actually cover them — rather than admiring a uniform pyramid. Coverage % is one lossy
signal: low coverage guarantees untested area, but high coverage guarantees nothing. Use it
to find gaps, never as the goal.

| Smell | Signal |
|-------|--------|
| High-blast-radius code thinly tested | Data integrity, money, auth, irreversible/destructive ops with only incidental coverage |
| Error paths untested | `except`/`catch`/`if err != nil` branches never exercised |
| Covered but unasserted | Lines run by tests with no meaningful assertion (mutation testing reveals this) |
| Coverage-padding | Many near-duplicate tests that execute lines without checking behavior |
| Misallocated effort | Expensive E2E on short-lived/low-risk features; none on the actual key risk |
| Coverage as a mandate | Uniform "every team hits X%" CI gate treated as the quality bar |

```bash
# Error branches vs tests that assert on errors (mismatch = untested error paths)
grep -rcE '(except|catch ?\(|if err != nil|rescue)' --include='*.py' --include='*.go' --include='*.java' . | sort -t: -k2 -rn | head
grep -rcE '(assertRaises|pytest.raises|toThrow|assert.*[Ee]rror|expectError|\.Error\()' --include='test_*.py' --include='*.test.*' --include='*_test.go' .
# If a coverage report exists, read it for the gaps (don't generate one unasked)
find . -name 'coverage.xml' -o -name 'lcov.info' -o -name '.coverage' -o -name 'coverage-final.json' 2>/dev/null
```
**Sources:** *Risk-Driven Testing*; *Code Coverage Best Practices*. **Severity:** **P0/P1**
for untested high-blast-radius paths; **P3** for coverage cosmetics.

## Cross-Cutting Checks

- **Testability (production-code design).** Tests forced into heavy mocking often signal a
  design problem: classes that create/fetch their own collaborators (vs injecting them) or
  take per-call *work* in the constructor. If D2 over-mocking is everywhere, the fix may be
  *Construct with Collaborators, Call with Work* (constructor for lifetime deps, method params
  for per-call data), not just the tests. Flag as a root cause, not a per-test finding.
- **TDD hygiene (optional, git-based).** Tests that were written *after* the code they cover
  and shaped to match it tend toward change-detectors (D2). Heuristic signals: tests landing
  in separate, later commits than their code; commit messages like "add tests for existing X";
  tests that demonstrably never failed first. Use only as supporting evidence, never as a
  standalone finding. **Sources:** *The Way of TDD*; *Construct with Collaborators, Call with Work*.

## Process

### Phase 1 — Scope
Ask the user (skip if scope passed by an automated caller):
> What should I audit? (1) full suite (2) a directory/module's tests (3) a specific test file
> (4) tests touched by the current diff/branch

For large suites, start with the highest-risk subsystem to avoid context overflow.

### Phase 2 — Detect frameworks & collect candidates
Run the Detection Strategy block, identify the stack, then run each dimension's greps.
**Collect raw candidates — don't analyze yet.**

### Phase 3 — Confirm by reading
For each candidate, open and read the test. Grep flags suspects; reading confirms the smell
and rules out false positives (a `for` loop may be legitimate table-driven setup; a `0` may be
a genuine boundary). Only confirmed smells become findings.

### Phase 4 — Triage
| Severity | Criteria |
|----------|----------|
| **P0 — Critical** | Vacuous test on a critical path; pure change-detector blocking a refactor; high-blast-radius behavior with no real assertion |
| **P1 — High** | Over-mocked/brittle tests; weak values that hide bugs; non-hermetic/flaky; collapsed assertions in critical paths; untested error paths on important code |
| **P2 — Medium** | Unfocused tests; unreadable (logic/over-DRY); non-descriptive names; minor brittleness |
| **P3 — Low** | Coverage cosmetics; style-level naming; impl-detail-class tests that are harmless |

### Phase 5 — Report
Present findings in the format below. **Read-only by default** — propose fixes, don't apply
them unless the user asks (then offer to `--fix` one dimension at a time, or hand off to TDD).

## Report Format

```markdown
# TEST AUDIT REPORT
**Scope:** <what was audited>   **Stack:** <frameworks detected>   **Date:** <date>
**Test files:** <N>   **Tests:** <~M>   **Coverage (if known):** <%>

## Summary
| Dimension | Findings | P0 | P1 | P2 | P3 |
|-----------|----------|----|----|----|----|
| D1 Vacuous | | | | | |
| D2 Brittle/over-mocked | | | | | |
| D3 Unfocused | | | | | |
| D4 Unreadable | | | | | |
| D5 Unactionable failures | | | | | |
| D6 Weak values | | | | | |
| D7 Non-hermetic | | | | | |
| D8 Risk/coverage gaps | | | | | |

## Critical (P0) — fix or delete now
### [D1 Vacuous] <test::name>  — <file:line>
- **Why it's worthless:** <e.g. expected value is the int default 0; method storing nothing still passes>
- **Proof:** <the line, or "mutate the impl to `return` and this still passes">
- **Fix:** <non-default value / real assertion / delete>

## High (P1)
### [D2 Change-detector] <test::name> — <file:line>
- **Coupled to:** <what implementation detail / which mocks>
- **Refactor that breaks it without a behavior change:** <example>
- **Fix:** <assert on public behavior; replace mock with fake/real; narrow the assertion>

## Medium (P2) / Low (P3)
<grouped, terser>

## Root-Cause Notes
- <e.g. "Over-mocking is pervasive in payments/* — the classes fetch their own DB handles.
   Consider constructor injection (Construct with Collaborators) before rewriting tests.">

## Recommended Next Steps
1. Delete/rewrite the N P0 vacuous tests (they read as coverage but catch nothing).
2. ...
```

## Anti-Patterns (for this skill itself)

| Don't | Do instead |
|-------|------------|
| Report a grep hit without reading the test | Confirm every finding by reading the test body |
| Flag every mock as bad | Mocks are fine in moderation; flag *over*-mocking and pure change-detectors |
| Flag every `for`/`0` as a smell | Table-driven setup and genuine boundary values are legitimate — read context |
| Treat coverage % as the verdict | Coverage finds gaps; quality is whether tests would *fail on a real bug* |
| Auto-rewrite tests | Read-only by default; propose, then fix on request |
| Confuse "no test exists" with "bad test" | Missing tests → `surface-tech-debt`; this skill judges tests that *do* exist |

## Integration with Other Skills

- **vs `surface-tech-debt` (Dim 2):** that asks *"is there a test?"*; this asks *"is the test
  any good?"*. Run tech-debt for gaps, test-audit for quality.
- **After audit → fix:** hand confirmed findings to your normal TDD loop, or `/code-review
  --fix` style application, one dimension at a time.
- **`postmortem`:** when a green suite shipped a silent bug, postmortem traces the damage;
  test-audit finds the *other* tests with the same blind spot so it doesn't recur.
- **`doc-prover`:** for invariants that must never break, a formal ASSERTION proof is stronger
  than a unit test — escalate the highest-risk D8 items there.

## Checklist

- [ ] Asked scope (or received it from caller)
- [ ] Detected the test frameworks/assert idioms in use
- [ ] Ran candidate greps for all 8 dimensions
- [ ] **Read** each candidate to confirm the smell (no unread grep-hit findings)
- [ ] Checked vacuous/default-value traps (D1, D6) — would a no-op impl still pass?
- [ ] Checked mock/verify density and whole-object equality (D2)
- [ ] Checked focus, readability, names, collapsed assertions (D3–D5)
- [ ] Checked hermeticity / sleeps / order-dependence (D7)
- [ ] Mapped key risks and untested error/high-blast-radius paths (D8)
- [ ] Noted testability root causes (DI) where over-mocking is pervasive
- [ ] Triaged P0–P3; produced the report; left fixes for the human unless asked

## Sources (Google Testing Blog / Testing on the Toilet)

| Dimension | Post | URL |
|-----------|------|-----|
| Core lens, D5 | Test Failures Should Be Actionable | https://testing.googleblog.com/2024/05/test-failures-should-be-actionable.html |
| D1, D6 | Choosing Values for Robust Tests | https://testing.googleblog.com/2026/06/choosing-values-for-robust-tests.html |
| D2 | Change-Detector Tests Considered Harmful | https://testing.googleblog.com/2015/01/testing-on-toilet-change-detector-tests.html |
| D2 | Test Behavior, Not Implementation | https://testing.googleblog.com/2013/08/testing-on-toilet-test-behavior-not.html |
| D2 | Don't Overuse Mocks | https://testing.googleblog.com/2013/05/testing-on-toilet-dont-overuse-mocks.html |
| D2 | Prefer Testing Public APIs Over Implementation-Detail Classes | https://testing.googleblog.com/2015/01/testing-on-toilet-prefer-testing-public.html |
| D2 | Prefer Narrow Assertions in Unit Tests | https://testing.googleblog.com/2024/04/prefer-narrow-assertions-in-unit-tests.html |
| D3 | Test Behaviors, Not Methods | https://testing.googleblog.com/2014/04/testing-on-toilet-test-behaviors-not.html |
| D3 | Keep Tests Focused | https://testing.googleblog.com/2018/06/testing-on-toilet-keep-tests-focused.html |
| D4 | Tests Too DRY? Make Them DAMP! | https://testing.googleblog.com/2019/12/testing-on-toilet-tests-too-dry-make.html |
| D4 | Don't Put Logic in Tests | https://testing.googleblog.com/2014/07/testing-on-toilet-dont-put-logic-in.html |
| D4 | Keep Cause and Effect Clear | https://testing.googleblog.com/2017/01/testing-on-toilet-keep-cause-and-effect.html |
| D5 | Writing Descriptive Test Names | https://testing.googleblog.com/2014/10/testing-on-toilet-writing-descriptive.html |
| D7 | Test Sizes | https://testing.googleblog.com/2010/12/test-sizes.html |
| D8 | Risk-Driven Testing | https://testing.googleblog.com/2014/05/testing-on-toilet-risk-driven-testing.html |
| D8 | Code Coverage Best Practices | https://testing.googleblog.com/2020/08/code-coverage-best-practices.html |
| Cross-cutting | Construct with Collaborators, Call with Work | https://testing.googleblog.com/2026/05/construct-with-collaborators-call-with.html |
| Cross-cutting | The Way of TDD | https://testing.googleblog.com/2026/03/the-way-of-tdd.html |
```
