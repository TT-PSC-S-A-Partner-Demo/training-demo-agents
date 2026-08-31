---
name: sdlc-architect-loop
description: Produces a technical design - reuse survey, module map, public contracts, error model, requirement trace - before any code is written. Use when deciding how to structure a change, whether to extend an existing module or add a new one, or when an SDLC finding targets the design phase. Do not use standalone - it is invoked by the sdlc-architect agent, and it writes no production code.
---

# Architect Loop

## Important

- **Reuse survey first, always.** A design that reinvents an existing module is
  a defect, and it is cheapest to catch here.
- **Every component traces to an `R<n>`, every `R<n>` traces to a component.**
  A gap in either direction is a finding, not something to paper over.
- Want something the requirements do not cover? Raise a finding targeting
  `analysis` instead of designing it in quietly.

Five passes. Repeat 3-5 until exit criteria hold or **3 revisions**, then report.

## Pass 1 — Reuse survey (do this first, always)

Before proposing anything, find what exists:

- Glob for modules whose names match the task's nouns.
- Grep for the operations the requirements describe.
- Read the two or three closest existing modules end to end. Their shape is your
  default shape.

Record findings as: `<existing thing> — covers <R<n>> fully | partially | not`.
A design that adds a new module where an existing one could be extended is a
finding against yourself. Catch it here, not in review.

## Pass 2 — Draft

Write `.sdlc/design.md` with these sections, in order:

1. **Reuse survey** — the pass-1 table.
2. **Module map** — `path — responsibility`, one line each, marked `new` or `change`.
3. **Public contracts** — exact signatures with types.
4. **Data flow** — how a call travels through the components, in prose or ASCII.
5. **Error model** — `<condition> -> <exception type> -> <message shape>`.
6. **Requirement trace** — `R<n> -> component`, every requirement, no gaps.

## Pass 3 — Simplification challenge

Self-critique. For each component ask:

1. **Which `R<n>` dies if I delete this?** None -> delete it. This catches
   speculative interfaces, config layers, and "future-proof" indirection, which
   are the most expensive defects to remove later.
2. **Could an existing module do this?** If yes and you still added a new one,
   justify in one line or revert to reuse.
3. **Is the abstraction level right for this project's scale?** A three-function
   script does not need a plugin registry.
4. **How many files must change to satisfy one requirement?** More than two is a
   smell — usually the responsibility split is wrong.

## Pass 4 — Trace challenge

1. Every `R<n>` appears in the trace table. A gap means either an unmapped
   requirement (fix the design) or a requirement nobody can build (raise a
   finding targeting `analysis`).
2. Every component traces back to at least one `R<n>`. Untraced component ->
   delete, or the requirement is missing.
3. Every error-model row traces to a requirement about error behaviour.

## Pass 5 — Revise and check exit

Exit only when **all** hold:

- [ ] Reuse survey present and acted on.
- [ ] Every contract has concrete types, no `Any`-shaped hand-waving.
- [ ] Trace table complete in both directions.
- [ ] Every design finding resolved and its `findings.jsonl` line updated.
- [ ] No component survives the pass-3 delete test.

## Examples

**Pass 1 changes the answer.**
In: `R7` "export the report as CSV".
Survey finds `reporting/exporters.py` with `export_json()` beside it.
Out: trace says `R7 -> reporting/exporters.py (extend)`. A new module would have
shipped a second, divergent CSV quoting rule.

**Pass 3 deletes a component.**
Draft has `ExporterRegistry` for plugin lookup.
Delete test: remove it, and no `R<n>` fails — nothing asks for third-party
exporters.
Out: registry cut, `export_csv()` called directly.

**Pass 4 finds a hole.**
`R6` "should be fast" maps to no component and cannot.
Out: finding targeting `analysis` — "R6 has no measurable threshold; needs a
number and a workload before it can be designed or tested."

## Finally

Append the work-log entry per `sdlc-protocol` and return the summary the
`sdlc-architect` agent definition asks for.
