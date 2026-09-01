---
name: sdlc-adversarial-loop
description: Second independent test pass that runs concurrently with the primary tester - derives its own cases from the requirements, executes them, and reports coverage per requirement so the orchestrator can reconcile both lanes. Use when a change is high-risk, when a green suite needs an adversarial second opinion, or when an SDLC finding targets the adversarial phase. Do not use standalone - it is invoked by the parallel-tester agent, and it never edits production code or the primary suite.
---

# Adversarial Loop

## Important

- **You run beside the primary tester, not after it.** `test-report.md` is being
  written while you work. Never read it; there is nothing in it yet from this
  iteration, and an older copy would anchor you to a stale verdict.
- **Never edit production code or the primary suite.** Your tests live in the
  file the orchestrator assigned you, nowhere else.
- **Pass 3 is not optional.** A claimed pass in a second opinion manufactures
  false confidence — worse than running no second pass at all.

Five passes. Passes 2-4 repeat only to fix **your own** test bugs, max **3
revisions**. Defects in production code are never fixed here — they are routed.

## Pass 1 — Independent derivation

Read `.sdlc/requirements.md`, `.sdlc/design.md`, and the files named in
`state.json.changed_files`. Nothing from the other lane. List your cases before
writing any code:

| case | R | input | expected | why a first pass misses it |
|---|---|---|---|---|
| range_single_day | R4 | start == end | that day's total | boundary collapse |

Bias the list toward where a first pass goes thin:

- boundary and off-by-one on **either** side of every limit
- malformed, partial, truncated, and wrong-type input
- combinations of filters and flags, not one flag at a time
- empty result sets, and the difference between empty and null
- aggregation consistency: does the total equal the sum of its slices
- ordering, duplicates, and repeated invocation

## Pass 2 — Write

- Write only in the file the orchestrator assigned you. Never in the primary
  suite, never in production code. The other lane is writing its own file at this
  moment; two agents in one file lose each other's work without an error.
- Use the project's existing framework and layout. Do not introduce a new one.
- Your own fixtures inside your own file. If a shared fixture genuinely needs
  changing, that is a `minor` finding targeting `testing`, not an edit.
- No logic in tests. Hardcode the expected value, or the test reproduces the bug
  it exists to catch.

## Pass 3 — Execute (non-negotiable)

Run your cases. Capture the real output verbatim, and the exact command.

Never write "should pass". A second opinion nobody executed is not evidence, it
is a second guess wearing the costume of one.

## Pass 4 — Classify and route

Route every failure to its **root-cause phase** per the routing table in
`sdlc-protocol` — it is the canon; do not keep a second copy here. Two cases are
yours:

- **Your own test is wrong** — fix it here, no finding, note it in the report.
- **A case the suite should have had** — two findings: the defect at its
  root-cause phase, and a `testing` finding for the missing case.

Never suppress a finding on the theory that the other lane is probably filing it
too. You cannot see their findings. The merge step deduplicates on `code`,
`target_phase` and location, and marks the survivor `confirmed_by` — two
independent reports of one defect is the strongest signal this pipeline
produces, and guessing it away destroys it.

Findings go to `.sdlc/inbox/parallel-tester.jsonl`, numbered from the id the
orchestrator gave you. Resolutions and your work-log entry are **returned as
text**, not written — `findings.jsonl` and `work-log.md` are off limits while the
other lane runs. Each finding carries the real error text as `evidence`.

## Pass 5 — Report and check exit

Write `.sdlc/adversarial-report.md`:

```markdown
# Adversarial report — it<N>
Command: `<exact command>`
Result: 9 passed, 2 failed

## Coverage
| case | R | result |
|---|---|---|
| range_single_day | R4 | FAIL |
| empty_range | R4 | pass |

Requirements with no case from this lane: R7 (untestable - see F-030).

## Failures
### range_single_day (R4)
```
<verbatim runner output>
```
-> F-028 target_phase=implementation, F-029 target_phase=testing
```

The coverage table is the deliverable, not a formality: the merge step computes
the agreement count from it, and a lane that reports only its failures is
indistinguishable from a lane that ran nothing.

Exit when **all** hold:

- [ ] `test-report.md` was never opened.
- [ ] Every case was executed and the output is pasted, not summarized.
- [ ] The coverage table lists every case with its `R<n>` and result, passes
      included.
- [ ] Every failure classified and routed with evidence, none suppressed as a
      presumed duplicate.
- [ ] Findings are in your inbox; resolutions and work-log entry are returned as
      text, not written.
- [ ] No production file and no primary-suite file was modified.

## Examples

**A case the other lane may not have.**
In: `R4` covers a date-range aggregation. Your derivation includes start == end;
ran it -> 0 rows where `R4` requires that day's total.
Out: `F-028 major | implementation` (inclusive-end off by one) and
`F-029 minor | testing` (no single-day case). Whether the tester found it too is
not your call to make — the merge step settles it.

**Everything passes.**
In: nine derived cases, all green.
Out: no findings, and a nine-row coverage table naming the `R<n>` behind each.
The table is what turns into the agreement count.

**Stale artifact in the directory.**
`test-report.md` exists from iteration 1; you are in iteration 2.
Out: leave it closed and say so in the report. It describes a run that no longer
matches the code you are testing.

## Finally

Append the work-log entry per `sdlc-protocol` and return pass/fail counts, the
disagreement list, and every finding with its `target_phase`.
