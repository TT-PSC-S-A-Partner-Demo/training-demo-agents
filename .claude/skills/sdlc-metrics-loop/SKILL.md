---
name: sdlc-metrics-loop
description: Turns metrics-shaped requirements into numbered KPI definitions with event semantics, aggregation rules, dimensions, and a validation example per metric. Use when a task asks for usage analytics, telemetry analysis, adoption reporting, or when an SDLC finding targets the metrics phase. Do not use standalone - it is invoked by the metrics-analyst agent, and it writes no queries or dashboards.
---

# Metrics Loop

## Important

- **A metric two people can compute differently is not defined.** Event, filter,
  aggregation, window — all four, every time.
- **Never assume a telemetry field exists.** Cite the emitter, or file `Q<n>`.
- Every `M<n>` traces to an `R<n>`. Untraced metric is unrequested scope.

Four passes. Repeat 2-4 until the exit criteria hold or you hit **3 revisions**,
then report what still fails and hand back to the orchestrator.

## Pass 1 — Schema harvest (once per run)

Before defining anything, find out what the telemetry actually emits:

- Read `.sdlc/requirements.md`. List the `R<n>` that ask for a number.
- Read every unresolved finding with `"target_phase": "metrics"`.
- Read `.sdlc/metrics.md` if it exists — you amend, never restart.
- Grep the codebase for event emitters, schema definitions, fixture files, and
  existing aggregation code. Record each field you find with the file that
  proves it.

Write down which fields the requirements need that the harvest did **not** find.
That list drives pass 3.

## Pass 2 — Draft

One block per metric:

```markdown
- **M1** (serves R3): Daily active user
  - **Event**: `command.completed`
  - **Filter**: `actor_type == 'human'`
  - **Aggregation**: distinct `actor_id`
  - **Window**: calendar day, UTC
  - **Dimensions**: `team_id`, `command_name`
  - **Empty/partial**: no events in window -> `0`, not null
  - **Validates**: 4 events / 2 actors / 1 bot in one day -> `1`
  - **Risk**: late-arriving events shift the count for up to 24h
```

Every field referenced gets a source citation. Uncited -> `Q<n>` instead.

## Pass 3 — Ambiguity challenge

Self-critique. For every metric ask:

1. **Could two analysts compute a different number from this?** If yes, name what
   they would disagree about and pin it down.
2. **Is every field proven to exist?** Cite the emitter file, or move the metric
   to `Q<n>`.
3. **What does it do on empty, partial, duplicate, and late data?** Silence here
   ships a dashboard that reads null as zero.
4. **Which `R<n>` dies if I delete this metric?** None -> delete it, and raise a
   finding targeting `analysis` if you believe it is genuinely needed.
5. **Does the validation example actually produce the stated number?** Compute it
   by hand. An example that does not check out is worse than none.

## Pass 4 — Revise and check exit

Rewrite the failures. Exit only when **all** hold:

- [ ] Every metric passes all five pass-3 checks.
- [ ] Every metric has a validation example that computes.
- [ ] Every field is cited to an emitter or schema file.
- [ ] Every `M<n>` traces to an `R<n>`, and every number-shaped `R<n>` has an `M<n>`.
- [ ] Every metrics finding resolved and its `findings.jsonl` line updated with a
      `"resolution"`.
- [ ] No metric number removed or renumbered.
- [ ] Open questions listed as `Q<n>`, not silently answered.

After 3 revisions, stop and report the unresolvable items.

## Artifact template

```markdown
# Metrics — <task>
_iteration <N>, version <V>_

## Definitions
- **M1** (serves R3): ...

## Data-quality risks
- late events: ...

## Open questions
- **Q1**: ... (blocks M4)

## Trace
| M | serves R | source of truth |
|---|---|---|

## Change log
- it2: +M4 (resolves F-006)
```

## Examples

**Vague in, computable out.**
In: `R3` "report how many developers actively use the tool".
Out: `M1` as in the pass-2 block above, plus `Q1` "does a failed command count as
activity?" — the requirement never said, and both answers are defensible.

**Pass 3 check 2 kills a metric.**
Draft splits paid from trial on `event.seat_type`. Grep finds no emitter writing
it. Out: `Q2` naming the missing field, not a metric. Assuming it exists ships a
column that is silently always null.

**Pass 3 check 4 deletes one.**
Draft adds `M7` median latency. No `R<n>` mentions latency. Out: `M7` cut, finding
targeting `analysis`.

## Finally

Append your work-log entry per `sdlc-protocol`, then return the summary the
`metrics-analyst` agent definition asks for.
