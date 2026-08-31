---
name: sdlc-analyst-loop
description: Turns a task into numbered, testable requirements and amends them when a defect traces back to a missing requirement. Use when writing acceptance criteria, when a spec is too vague to test, or when an SDLC finding targets the analysis phase. Do not use standalone - it is invoked by the sdlc-analyst agent, and it does not design or implement.
---

# Analyst Loop

## Important

- **Never renumber or delete an existing `R<n>`.** Every downstream artifact
  traces to those numbers; renumbering breaks all of them at once.
- **A requirement a tester cannot assert is not a requirement.** Name the
  observable outcome, or file it as `Q<n>`.
- Findings routed here mean a requirement was *missing*, never that code was
  wrong.

Four passes. Repeat passes 2-4 until the exit criteria hold or you hit **3
revisions**, then report what still fails and hand back to the orchestrator.

## Pass 1 — Harvest (once per run)

- Read `.sdlc/state.json` for the task and iteration.
- Read every unresolved finding with `"target_phase": "analysis"`.
- Read `.sdlc/requirements.md` if it exists — you amend, never restart.
- Grep the codebase for the nouns in the task. Existing behaviour constrains
  what you may require.

Write down what the task does **not** say. That list drives pass 3.

## Pass 2 — Draft

One line per requirement:

```markdown
- **R4**: Malformed input is rejected with a `ValueError` naming the bad token.
```

Shape: `<subject> <verb> <observable outcome> [under <condition>]`.
Split any requirement containing "and" that hides two separate outcomes.

For each finding you are resolving, add or amend exactly the requirement that
was missing, and note the finding id beside it.

## Pass 3 — Testability challenge

Self-critique. For every requirement ask:

1. **Can a tester write an assertion from this alone?** If it needs a design
   decision first, it is underspecified — fix the wording.
2. **What is the observable outcome?** "Handles errors gracefully" has none.
   Name the exception type, the message, the return value.
3. **What is the boundary?** Empty, zero, one, max, wrong type. If the
   requirement is silent, either state the behaviour or file it as `Q<n>`.
4. **Is this a requirement or a design decision?** "Uses a stack" is design.
   Move it out.

Every requirement that fails a check gets rewritten in pass 4.

## Pass 4 — Revise and check exit

Rewrite the failures. Then exit only when **all** hold:

- [ ] Every requirement passes all four pass-3 checks.
- [ ] Every unresolved analysis finding maps to a requirement, and its
      `findings.jsonl` line is rewritten with `"resolved": true` plus a
      `"resolution"` naming the requirement.
- [ ] No requirement number was removed or renumbered.
- [ ] Open questions are listed as `Q<n>`, not silently answered.

Otherwise loop back to pass 2. After 3 revisions, stop and report the
unresolvable items — grinding further wastes the run's budget.

## Artifact template

```markdown
# Requirements — <task>
_iteration <N>, version <V>_

## Functional
- **R1**: ...

## Error behaviour
- **R4**: ...

## Out of scope
- ...

## Open questions
- **Q1**: ... (blocks R6)

## Change log
- it2: +R4, +R5 (resolves F-002, F-003)
```

## Examples

**Vague in, testable out.**
In: `"Build an RPN calculator."`
Out: `R1` "evaluate returns a float for a whitespace-separated expression",
`R2` "operators `+ - * /`", `R3` "int and float literals", plus
`Q1` "is scientific notation in scope?" — unanswered in the task, so asked, not
guessed.

**Pass 3 rewrite.**
Draft: `R4` "handles errors gracefully".
Pass-3 check 2 fails — no observable outcome.
Rewritten: `R4` "Malformed input is rejected with a `ValueError` naming the bad
token."

**Finding in.**
In: `F-003 target_phase=analysis` — "1 0 / raises ZeroDivisionError; no
requirement covers it".
Out: `R5` added, `F-003` rewritten `"resolved": true,
"resolution": "added R5 (division-by-zero contract)"`, `R1`-`R4` untouched.

## Finally

Append your work-log entry per `sdlc-protocol`, then return the summary the
`sdlc-analyst` agent definition asks for.
