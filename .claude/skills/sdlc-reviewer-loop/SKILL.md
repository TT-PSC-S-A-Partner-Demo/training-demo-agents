---
name: sdlc-reviewer-loop
description: Reviews a change for correctness, over-engineering, security, and codebase fit, verifies each finding with a concrete failure scenario, and issues GO or NO-GO. Use when reviewing a diff before merge or deciding whether work is shippable. Do not use standalone - it is invoked by the sdlc-reviewer agent, and it never fixes what it finds.
---

# Reviewer Loop

## Important

- **Pass 4 is what makes this a review and not a list of hunches.** A candidate
  you cannot turn into a concrete failing input gets dropped, not downgraded.
- **Never edit source or tests.** You write `.sdlc/review.md`, `findings.jsonl`,
  and `work-log.md` — nothing else.
- **Say GO when the work is good.** A reviewer who always finds something trains
  the team to ignore the verdict, and the one real blocker goes with it.

Five passes, read-only on code. Passes 2-4 repeat at most **3 times**; after
that, report what is still unverified rather than looping. The verify pass (4)
is never the one you skip to save a round.

## Pass 1 — Scope

- Read `.sdlc/requirements.md`, `.sdlc/design.md`, `.sdlc/test-report.md`.
- List the files changed this run. Read each one in full, plus enough of its
  callers to know how it is used.
- Note what the tester already covered. Do not re-report a known failure.

## Pass 2 — Six-axis sweep

Collect **candidates**, do not judge yet:

1. **Correctness** — off-by-one, wrong operator, inverted condition, swallowed
   exception, missing `await`, resource never closed, mutable default.
2. **Root cause** — a fix that special-cases the failing input instead of
   repairing the logic.
3. **Over-engineering** — abstraction, config, or generality no `R<n>` demands.
4. **Codebase fit** — naming, error handling, and idiom unlike the neighbours.
5. **Security** — unvalidated input, injection, path traversal, secrets in code
   or logs, unsafe deserialization, missing authz check.
6. **Trace** — each `R<n>` implemented **and** tested. Untested requirement is a
   finding against `testing`.

## Pass 3 — Contract check

Compare each public signature against `.sdlc/design.md`. A silent drift between
design and code is a finding, even when the tests pass — the next developer
reads the design and gets it wrong.

## Pass 4 — Verify every candidate

For each candidate, construct a **concrete failure scenario**: exact input or
state, and the wrong output or crash it produces. Trace the code path by hand;
use `Bash` to run a read-only reproduction where you can.

- Cannot make it concrete -> **drop it**. Unverified findings burn the team's
  trust and their budget.
- Verified and would ship broken -> `blocker`.
- Verified but degrades quality only -> `major`.
- Style or preference -> `minor`, log only, no rewind.

Route each to its root-cause phase per the routing table in `sdlc-protocol`.
One addition that is yours: a defect the suite should have caught is *also* a
`testing` finding — green tests are evidence, not proof.

## Pass 5 — Verdict

Write `.sdlc/review.md`:

````markdown
# Review — it<N>
## Verdict: GO | NO-GO

## Checked clean
- error model matches design (all 4 rows)
- R1-R5 implemented and tested

## Findings
### F-007 blocker | implementation | src/calc.py:31
Guard uses `>` where `>=` is required.
Failure: `evaluate("1 +")` pops an empty stack -> IndexError instead of ValueError.
````

**NO-GO** if any `blocker` survives pass 4. `major` findings rewind the pipeline
but do not by themselves block. `minor` never rewinds.

Say **GO** when the work is good. A reviewer who always finds something trains
the team to ignore the verdict, and the one real blocker gets ignored with it.

Exit criteria:

- [ ] Every finding has `file:line` and a concrete failure scenario.
- [ ] Every finding routed to its root-cause phase.
- [ ] No file was edited.
- [ ] Verdict stated explicitly.

## Examples

**Candidate survives pass 4.**
Spotted: `rpn_calc.py:14` guards `len(stack) < 2` before every pop, but the unary
branch added for `R8` pops once.
Verified: `evaluate("neg")` → IndexError, not the `ValueError` the error model
promises.
Out: `F-009 blocker | implementation | rpn_calc.py:14`, verdict **NO-GO**.

**Candidate dropped in pass 4.**
Spotted: "the float comparison looks fragile."
Verification: no input produces a wrong answer under the documented contract.
Out: nothing filed. An unverified hunch costs the team a hunt and buys nothing.

**Contract drift, pass 3.**
Design says `evaluate(expression: str) -> float`; code returns
`tuple[float, list]`. Tests pass because they only read `[0]`.
Out: `F-010 major | design | rpn_calc.py:9` — the design now lies about the API,
and the next developer will trust it.

**Clean pass.**
`R1`-`R5` traced to code and tests, error model matches, no security surface.
Out: **GO**, with the "checked clean" list.

## Finally

Append the work-log entry per `sdlc-protocol` and return the verdict plus every
finding with severity, `target_phase`, location, and failure scenario.
