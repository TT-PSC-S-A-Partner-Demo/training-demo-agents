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
  state.json        # phase pointer, iteration, budget
  requirements.md   # analyst artifact
  design.md         # architect artifact
  work-log.md       # append-only, every agent appends one entry per run
  findings.jsonl    # append-only, one JSON object per finding
  test-report.md    # tester artifact
  review.md         # reviewer artifact
```

Source code goes in the real project tree, never in `.sdlc/`.

### state.json

```json
{
  "task": "<original task text>",
  "iteration": 1,
  "max_iterations": 6,
  "entry_phase": "analysis",
  "status": "running"
}
```

`status`: `running` | `success` | `budget_exhausted` | `blocked`.

## Phases

`analysis -> design -> implementation -> testing -> review -> done`

| Phase | Owner agent | Produces |
|---|---|---|
| analysis | `sdlc-analyst` | `requirements.md` |
| design | `sdlc-architect` | `design.md` |
| implementation | `sdlc-developer` | source files |
| testing | `sdlc-tester` | tests + `test-report.md` |
| review | `sdlc-reviewer` | `review.md` |

## Finding format

A finding is one defect with the phase that owns its **root cause** — not the
phase that noticed it. Append one JSON object per line to `.sdlc/findings.jsonl`:

```json
{"id":"F-003","code":"TEST_FAILED_DIV_ZERO","severity":"blocker","target_phase":"analysis","message":"1 0 / raises ZeroDivisionError; no requirement covers it","evidence":"pytest test_div_zero: ZeroDivisionError","iteration":1,"resolved":false}
```

- `severity`: `blocker` (pipeline stops) | `major` (one more loop) | `minor` (log only, no rewind).
- `target_phase`: where the fix belongs. Missing requirement -> `analysis`.
  Wrong structure or contract -> `design`. Coding slip -> `implementation`.
- One defect may need **two** findings when both the requirement and the code
  are missing. Emit both, with different `target_phase`.

### Root-cause routing table

| Symptom | target_phase | Why |
|---|---|---|
| Behaviour nobody ever specified | `analysis` | requirement gap |
| Spec exists, structure cannot satisfy it | `design` | design gap |
| Spec + design fine, code wrong | `implementation` | coding defect |
| Test itself wrong or missing | `testing` | test gap |

## Resolving findings

When an agent re-runs its phase, it MUST:
1. Read every unresolved finding in `findings.jsonl` whose `target_phase`
   matches its own phase.
2. Address each one explicitly in its output.
3. Rewrite those lines with `"resolved": true` and add `"resolution": "<what changed>"`.

Never delete a finding line. History is the audit trail.

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
