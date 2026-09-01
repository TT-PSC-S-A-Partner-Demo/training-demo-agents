---
name: sdlc-protocol
description: Shared contract for the SDLC agent team - where state lives, how artifacts are named, the finding format, and which phase owns each class of defect. Use before reading or writing any .sdlc/ artifact, filing a finding, or deciding where a defect should be sent back to. Do not use for general code review or commit conventions - this is the SDLC team's internal wire format.
---

# SDLC Protocol

Single source of truth for how the SDLC team exchanges work. Every role agent
and the orchestrator obey this file.

## Important

- **This file is the canon for the routing table below.** Other skills point
  here; they do not keep their own copy. Five copies of a table drift.
- **`target_phase` is where the *fix* belongs, not where the defect was
  noticed.** Getting this wrong turns the feedback loop into a retry loop.
- **Findings are never deleted**, only rewritten as resolved. The file is the
  audit trail.

## State layout

All shared state lives under `.sdlc/` in the project root:

```
.sdlc/
  state.json             # phase pointer, iteration, budget, finding-id counter
  requirements.md        # analyst artifact
  metrics.md             # metrics-analyst artifact (optional phase)
  design.md              # architect artifact
  work-log.md            # append-only, every agent appends one entry per run
  findings.jsonl         # one JSON object per line, rewritten in place on resolve
  test-report.md         # tester artifact
  adversarial-report.md  # parallel-tester artifact (optional)
  review.md              # reviewer artifact
  inbox/                 # per-agent finding drop-box, used by concurrent phases only
    sdlc-tester.jsonl
    parallel-tester.jsonl
```

Source code goes in the real project tree, never in `.sdlc/`.

### state.json

```json
{
  "task": "<original task text>",
  "iteration": 1,
  "max_iterations": 6,
  "entry_phase": "analysis",
  "optional_phases": [],
  "findings_seq": 0,
  "changed_files": [],
  "status": "running"
}
```

`status`: `running` | `success` | `budget_exhausted` | `blocked`.

`optional_phases`: which of `metrics` and `adversarial` this run includes. The
orchestrator decides once, at entry, and does not re-decide per iteration.

`findings_seq`: the highest finding number allocated so far. See
**Allocating finding ids** below — this field is what stops two agents in one
iteration from both filing `F-007`.

`changed_files`: paths the developer reported writing this run. The orchestrator
records them; the tester and reviewer read them rather than guessing what the
diff was. There is no assumption that the project is a git repository.

## Phases

`analysis -> design -> implementation -> testing -> review -> done`

| # | Phase | Owner agent | Produces | |
|---|---|---|---|---|
| 0 | analysis | `sdlc-analyst` | `requirements.md` | |
| 1 | metrics | `metrics-analyst` | `metrics.md` | optional |
| 2 | design | `sdlc-architect` | `design.md` | |
| 3 | implementation | `sdlc-developer` | source files | |
| 4 | testing | `sdlc-tester` **‖** `parallel-tester` | `test-report.md` **‖** `adversarial-report.md` | second agent optional |
| 5 | review | `sdlc-reviewer` | `review.md` | |

Phase 4 is the one **concurrent** phase: when `adversarial` is enabled, both
testers run at the same time, from the same inputs, and neither can see the
other's output. That is not an optimization — it is what makes the second opinion
independent. A second pass that runs afterwards can read the first one's report;
a second pass that runs beside it cannot, because the report does not exist yet.

`target_phase: "adversarial"` still names the parallel tester specifically (its
own test bugs, its own missed cases), but it sorts at `#4` alongside `testing`
for every rewind decision.

The `#` column is the canonical pipeline order. "Earliest phase" anywhere in this
family means lowest `#`, including the optional phases — an optional phase that
is not in `optional_phases` is skipped for both execution and rewind.

### Optional phases

Neither optional phase runs by default; each costs an iteration's worth of budget.

- **metrics** — include when the task asks for numbers: usage analytics,
  telemetry, KPIs, adoption or activity reporting. It defines what each number
  means before the architect designs a query for it.
- **adversarial** — include when the change is high-risk, or when a second
  independent test pass is worth the budget. It never replaces the tester; it
  runs after, derives its own cases blind, and reports disagreements.

Both are ordinary phases once enabled: they own an artifact, they file findings,
and findings can target them.

## Finding format

A finding is one defect with the phase that owns its **root cause** — not the
phase that noticed it. Write one JSON object per line in `.sdlc/findings.jsonl`:

```json
{"id":"F-003","code":"TEST_FAILED_DIV_ZERO","severity":"blocker","target_phase":"analysis","message":"1 0 / raises ZeroDivisionError; no requirement covers it","evidence":"pytest test_div_zero: ZeroDivisionError","iteration":1,"resolved":false}
```

- `severity`: `blocker` (pipeline stops) | `major` (one more loop) | `minor` (log only, no rewind).
- `target_phase`: where the fix belongs. Missing requirement -> `analysis`.
  Wrong structure or contract -> `design`. Coding slip -> `implementation`.
- One defect may need **two** findings when both the requirement and the code
  are missing. Emit both, with different `target_phase`.

### Allocating finding ids

Agents do not pick their own ids. Before invoking an agent, the orchestrator
reads `findings_seq`, reserves a block, and tells the agent in its prompt:
"your finding ids start at F-008". The agent numbers upward from there and
reports how many it used; the orchestrator writes the new `findings_seq`.

For the concurrent phase the orchestrator reserves **two disjoint blocks** before
the fan-out — `sdlc-tester` from F-008, `parallel-tester` from F-028 — sized
generously, because unused ids cost nothing and a collision costs the whole
ledger. Gaps in the numbering are expected and mean nothing.

Two agents choosing ids independently collide on `F-007`, and the resolution step
then rewrites the wrong line. That is why the counter lives in `state.json` and
not in anyone's head.

## Concurrent-phase write rules

Exactly one agent may write any given file. During phase 4 that rule is not
advisory — two agents are genuinely running at once, and a lost append is
invisible until someone audits the ledger by hand.

**While a concurrent phase is running, no agent touches `findings.jsonl` or
`work-log.md`.** Instead:

1. Each agent writes its new findings to its own drop-box,
   `.sdlc/inbox/<its-own-agent-name>.jsonl`, in the ordinary finding format.
2. Each agent **returns** its resolutions to the orchestrator rather than
   rewriting lines: `F-012 -> resolved, "resolution": "<what changed>"`.
3. Each agent **returns** its work-log entry as text rather than appending it.
4. The orchestrator merges — see **Merge step** below. Nothing an agent produced
   is in `findings.jsonl` until that merge runs.

Outside a concurrent phase, agents write `findings.jsonl` and `work-log.md`
directly, exactly as before. The inbox exists only to serialize concurrency.

### Merge step

After both concurrent agents return, the orchestrator, in this order:

1. Appends `sdlc-tester.jsonl` then `parallel-tester.jsonl` to `findings.jsonl`,
   in that fixed order, so the ledger is identical on a re-run.
2. **Deduplicates.** Two findings are one defect when they share `code`,
   `target_phase`, and location. Keep the tester's line, drop the duplicate, and
   add `"confirmed_by": "parallel-tester"` to the survivor. Independent
   confirmation is worth recording — it is the strongest evidence the run
   produces, and it is exactly what the concurrent design buys.
3. Applies the returned resolutions to the existing lines.
4. Appends both work-log entries, tester first.
5. Empties `.sdlc/inbox/`.
6. Reconciles the two reports, mechanically and without judgement:
   - **coverage**: which `R<n>` each pass executed a case for. Covered by neither
     is a `testing` finding the orchestrator reports to the user, not one it
     authors.
   - **agreement**: how many case names appear in both reports with the same
     result. Report the count. Without it, a disagreement proves nothing.
   - **contradiction**: the same case name with different results in the two
     reports. Two honest runs from identical inputs must not disagree — that
     means the runs saw different state, and no rewind fixes it. Stop the run,
     set `status: "blocked"`, and surface both commands and both outputs.

The orchestrator compares reported facts only — case names, requirement ids,
pass/fail. It does not judge whether a case is right; that stays the testers' and
the reviewer's job.

### Root-cause routing table

| Symptom | target_phase | Why |
|---|---|---|
| Behaviour nobody ever specified | `analysis` | requirement gap |
| A number is required but its definition is ambiguous or uncited | `metrics` | semantics gap |
| Spec exists, structure cannot satisfy it | `design` | design gap |
| Spec + design fine, code wrong | `implementation` | coding defect |
| Test itself wrong or missing | `testing` | test gap |
| Parallel tester's own cases are wrong or missing | `adversarial` | second-opinion gap |

`metrics` and `adversarial` rows apply only when that agent is in
`optional_phases`. When it is not, a semantics gap is an `analysis` finding and a
coverage gap is a `testing` finding — there is no agent to route it to
otherwise.

## Resolving findings

When an agent re-runs its phase, it MUST:
1. Read every unresolved finding in `findings.jsonl` whose `target_phase`
   matches its own phase.
2. Address each one explicitly in its output.
3. Rewrite those lines with `"resolved": true` and add `"resolution": "<what changed>"`.

Never delete a finding line, and never reuse an id.

**On the file being "append-only":** it is not, quite. New findings are appended;
resolving one **rewrites that line in place**, keeping its id and its original
`message` and `evidence`, adding `"resolved": true` and `"resolution"`. Nothing
is ever removed and no id is ever recycled, so the ledger stays a complete audit
trail — but a resolving agent does rewrite an existing line rather than append a
second one. An agent that appends a duplicate line for the same id instead leaves
two records of one defect, and the next reader cannot tell which is current.

## Work-log entry

Every agent appends to `.sdlc/work-log.md`:

```markdown
## it<N> | <phase> | <agent> | <OK|FAIL>
- consumed: F-001, F-003
- produced: requirements.md v2
- notes: added R5 (division by zero contract)
```

## Handoff rule

An agent never calls another role agent. It returns to the orchestrator, which
owns all routing. Agents are read-write on their own artifacts only.

## Examples

**Routing a defect with two owners.**
Tester finds `evaluate("1 0 /")` raising `ZeroDivisionError`, and no `R<n>`
mentions division by zero. Two lines get appended:

```json
{"id":"F-003","code":"TEST_FAILED_DIV_ZERO","severity":"blocker","target_phase":"analysis","message":"no requirement covers division by zero","evidence":"ZeroDivisionError: float division by zero","iteration":1,"resolved":false}
{"id":"F-004","code":"TEST_FAILED_DIV_ZERO","severity":"blocker","target_phase":"implementation","message":"no guard before the division","evidence":"ZeroDivisionError: float division by zero","iteration":1,"resolved":false}
```

The orchestrator rewinds to `analysis` — the earlier of the two.

**Resolving one.**
The analyst re-runs, adds `R5`, and rewrites its line in place:

```json
{"id":"F-003",...,"resolved":true,"resolution":"added R5 (division-by-zero contract)"}
```

`F-004` stays open until the developer's re-run clears it.

**Troubleshooting.**

- *Two agents want to write the same artifact.* They cannot — each artifact has
  exactly one owner in the phase table. If you are about to write someone else's
  file, you are doing their phase; stop and return to the orchestrator.
- *A finding has no obvious owner.* Ask which phase must change for the defect to
  be impossible next time. That is the `target_phase`. If the answer is "two of
  them", file two findings.
- *`.sdlc/` does not exist.* Only the orchestrator creates it. A role agent that
  finds it missing was invoked outside a run — say so rather than improvising a
  state directory.
- *A finding keeps coming back.* Three iterations of the same `code` means it is
  mis-routed, not unlucky. The orchestrator stops the run at that point.

**Routing that would be wrong.**
Filing only `F-004`. The guard gets added, `requirements.md` still says nothing
about division by zero, and the next person to read the spec removes the guard
as unexplained. That is the retry loop this protocol exists to prevent.
