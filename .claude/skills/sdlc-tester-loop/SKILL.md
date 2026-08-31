---
name: sdlc-tester-loop
description: Derives test cases from requirements, executes them for real, and routes each failure to the phase that owns its root cause. Use when writing a test suite, checking whether a change is really covered, or verifying a fix. Do not use standalone - it is invoked by the sdlc-tester agent, and it never edits production code.
---

# Tester Loop

## Important

- **Pass 4 is not optional.** A suite written but not executed is worth nothing,
  and reporting a pass you did not observe is the most damaging thing you can do
  here — every phase downstream treats your numbers as fact.
- **Never edit production code.** Not to make a test pass, not "just a typo".
  Raise a finding.
- **Route by root cause, not by who noticed.** Dumping every failure on the
  developer turns the feedback loop into a retry loop.

Six passes. Passes 3-5 repeat only to fix **your own** test bugs, max **3
revisions**. Failures in production code are never fixed here — they are routed.

## Pass 1 — Derive cases

For every `R<n>` in `.sdlc/requirements.md`, list cases before writing any code:

| case | R | input | expected | kind |
|---|---|---|---|---|
| happy_add | R1 | `"3 4 +"` | `7.0` | happy |
| div_zero | R5 | `"1 0 /"` | `ValueError("division by zero")` | error |

Every requirement gets at least one case. A requirement you cannot turn into a
case is a finding targeting `analysis` — file it, do not skip it silently.

## Pass 2 — Boundary sweep

Add the cases the requirements imply but do not spell out:

- empty input, whitespace-only, single element
- zero, negative, one, maximum, off-by-one either side of every boundary
- wrong type, `None`, malformed token
- every row of the error model in `.sdlc/design.md`

Happy-path-only suites are how broken code reaches production with a green tick.

## Pass 3 — Write

- Use the project's existing test framework and file layout. Do not introduce a
  new one.
- One assertion concern per test; name the test after the behaviour, not the
  function.
- No logic in tests — no loops computing the expected value. Hardcode it, or the
  test reproduces the bug it is meant to catch.

## Pass 4 — Execute (non-negotiable)

Run the suite. Capture the real output verbatim.

A suite that was written but not executed is worth nothing. Reporting a pass you
did not observe is the single most damaging thing you can do in this pipeline —
everyone downstream trusts your numbers.

## Pass 5 — Classify each failure

For every failure, decide the **root cause phase** before filing anything. Route
per the routing table in `sdlc-protocol` — it is the canon; do not keep a second
copy here. Two cases are yours alone:

- **Your own test is wrong** — fix it here, no finding, note it in the report.
- **No `R<n>` covers the behaviour** — file **two** findings, `analysis` *and*
  `implementation`. The guard alone leaves the spec silent, so the next person to
  read it removes the guard as unexplained.

Each finding carries the **real error text** as `evidence`.

## Pass 6 — Report and check exit

Write `.sdlc/test-report.md`:

```markdown
# Test report — it<N>
Command: `<exact command>`
Result: 6 passed, 3 failed

## Coverage by requirement
| R | cases | status |
|---|---|---|

## Failures
### div_zero (R5)
```
<verbatim runner output>
```
-> F-003 target_phase=analysis, F-004 target_phase=implementation
```

Exit when **all** hold:

- [ ] Every `R<n>` has at least one executed case.
- [ ] The suite was actually run and the output is pasted, not summarized.
- [ ] Every failure is classified and routed, with evidence.
- [ ] No production file was modified.
- [ ] Testing-phase findings against you are resolved and their lines updated.

## Examples

**The routing case that matters.**
In: `evaluate("1 0 /")` raises `ZeroDivisionError`; no `R<n>` mentions division
by zero.
Out: two findings — `F-003 analysis` ("no requirement covers division by zero")
and `F-004 implementation` ("no guard before the division"). One finding would
have fixed the code and left the spec silent.

**Ordinary coding defect.**
In: `evaluate("3 4 +")` returns `7.000000001`; `R1` specifies exact semantics.
Out: one finding, `implementation`. Spec is fine, code is not.

**Untestable requirement.**
In: `R6` "should be fast".
Out: no test, one finding targeting `analysis` — "no threshold, cannot be
asserted". Skipping quietly would leave `R6` looking covered.

**My own bug.**
In: `test_float_literal` asserts `5` against a float return.
Out: fixed here with `pytest.approx`, no finding, noted in the report.

## Finally

Append the work-log entry per `sdlc-protocol` and return pass/fail counts,
failing case names with real errors, and every finding with its `target_phase`.
