---
name: sdlc-developer-loop
description: Implements an approved design in real source files, runs the code, and self-reviews the diff before handing it on. Use when writing the implementation for a design, or fixing a defect whose root cause is in the code. Do not use standalone - it is invoked by the sdlc-developer agent, and it never writes or edits tests.
---

# Developer Loop

## Important

- **Never touch a test file.** The tester owns them. Editing a test to make it
  pass is the one move that destroys the whole loop's value.
- **Pass 4 is not optional.** Run the code and paste real output. "Should work"
  is not a result.
- **Fix the cause, not the symptom.** Special-casing the failing input passes the
  test and leaves the bug.

Six passes. Repeat 3-6 until exit criteria hold or **3 revisions**, then report
what is still broken with the real error text.

## Pass 1 — Context load

- Read `.sdlc/design.md` (authoritative) and `.sdlc/requirements.md`.
- Read the files you are about to change, in full, plus one neighbour file you
  are not changing. The neighbour teaches you the house style.
- Note the project's test command, formatter, and lint config.

## Pass 2 — Root-cause triage (only when findings target you)

For each unresolved `"target_phase": "implementation"` finding, write one line
before touching code:

```
F-004 | symptom: IndexError on '1 +' | cause: no arity guard before pop | fix: check len(stack) >= 2
```

If the cause turns out to be a missing contract rather than a coding slip, stop:
raise a finding targeting `design` or `analysis` and return. Implementing around
a design gap buries it where nobody will find it.

## Pass 3 — Implement

- Smallest change that fixes the cause. No drive-by refactors, no reformatting
  untouched lines — they hide the real diff from the reviewer.
- Match neighbouring code: naming, error handling, comment density, import order.
- Guard clauses at the top; the happy path unindented.
- Never touch test files. The tester owns them.

## Pass 4 — Run it

Non-negotiable. Execute the code you just wrote:

- the project's test command if one exists,
- otherwise a direct call exercising each changed path, including the failure
  case from every finding you claim to fix.

Paste real output into your report. Never write "should now pass".

## Pass 5 — Self-review

Read your own diff as if someone else wrote it:

1. **Does each finding's failure case now actually behave correctly?** Re-run it
   specifically, not just the suite.
2. **Symptom or cause?** A fix that special-cases the exact test input is a
   symptom patch — redo it.
3. **New failure modes?** What does the change do with empty, `None`, zero,
   concurrent access?
4. **Anything left in?** Debug prints, commented-out code, TODOs, unused imports.
5. **Does it read like the neighbour file?** If not, fix the style now.
6. **Did I implement anything the design does not ask for?** Delete it.

## Pass 6 — Revise and check exit

Exit only when **all** hold:

- [ ] Every design component in scope is implemented.
- [ ] Every implementation finding resolved, its failure case re-run and passing,
      and its `findings.jsonl` line updated with a `"resolution"`.
- [ ] The code ran, and you have the real output.
- [ ] No test file was modified.
- [ ] Self-review pass 5 is clean.

## Examples

**Triage line, then the fix.**
In: `F-005 target_phase=implementation` — "underflow: `1 +` raises IndexError".
Pass 2: `F-005 | symptom: IndexError on '1 +' | cause: no arity guard before pop
| fix: check len(stack) >= 2`.
Pass 3: guard added at `rpn_calc.py:14`.
Pass 4: ran `python -c "from rpn_calc import evaluate; evaluate('1 +')"` →
`ValueError: not enough operands for '+'`, pasted verbatim.

**Cause vs symptom, same finding.**
Symptom patch: `if expression == "1 +": raise ValueError(...)`. Test goes green,
`2 *` still crashes. Pass 5 check 2 catches it; redo as the arity guard.

**Triage says stop.**
In: design contracts `evaluate() -> float`, but `R7` needs the operand stack.
Out: no code. Finding targeting `design`. Returning a tuple anyway would leave
`design.md` lying about the public API.

## Finally

Append the work-log entry per `sdlc-protocol` and return: files changed, the
command you ran with its actual output, findings resolved, findings raised.
