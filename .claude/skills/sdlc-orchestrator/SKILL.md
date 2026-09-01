---
name: sdlc-orchestrator
description: Runs a task through a five-agent SDLC pipeline - analyst, architect, developer, tester, reviewer, plus an optional metrics phase and an optional second test lane that runs concurrently with the primary tester - rewinding to the phase that owns each defect's root cause until the pipeline is clean or the iteration budget runs out. Use when implementing a feature end to end with full rigor, when a change must be specified, built, tested and reviewed independently, or when asked to run the SDLC team. Do not use for single-file edits, quick bug fixes, or questions about existing code - the five-phase loop costs far more than those tasks are worth.
---

# SDLC Orchestrator

## Important

- **You do no analysis, design, coding, testing, or review yourself.** You route
  work and own the loop. Doing a phase's work destroys the independence the
  feedback loop exists to provide.
- **Rewind to the *earliest* phase among the findings' `target_phase` values**,
  not to the phase that raised them.
- **Never report success while an unresolved `blocker` sits in
  `findings.jsonl`.**

```
                                                  +-- sdlc-tester ------+
analysis -> [metrics] -> design -> implementation -+                     +- merge -> review -> done
    ^           ^           ^            ^         +-- [parallel-tester] +     |        |
    |           |           |            |                   (concurrent)      |        |
    +-----------+-----------+------------+---------- feedback ------------------+--------+
```

Phase 4 runs its two agents **at the same time**. `[metrics]` and
`[parallel-tester]` are optional; they run only when you put them in
`optional_phases` at step 0.

Every arrow back is a rewind, and they do not all come from review: the testers
rewind to analysis, metrics, design or implementation, and the reviewer rewinds
to any of the five before it. The target is always the phase that owns the
defect's root cause.

## Step 0 — Initialize

1. `Skill(sdlc-protocol)` — load the shared contract. Everything below assumes it.
2. Create `.sdlc/` and write `state.json`:

```json
{"task":"<task text>","iteration":1,"max_iterations":6,"entry_phase":"analysis","optional_phases":[],"findings_seq":0,"changed_files":[],"status":"running"}
```

3. Create empty `findings.jsonl` and `work-log.md`.
4. **Decide the optional phases now, once.** Both cost budget, so neither is a
   default:
   - add `"metrics"` when the task asks for numbers — usage analytics, telemetry,
     KPIs, adoption or activity reporting. Without it, "count of active users"
     reaches the architect undefined and the disagreement surfaces at review.
   - add `"adversarial"` when the change is high-risk, or when the user asked for
     an independent second test pass.

   Say in your first line to the user which you enabled and why. Do not
   re-litigate this per iteration; a run that changes shape mid-flight cannot be
   compared against its own earlier iterations.
5. If the task text is one line of vague intent, ask the user **once** for the
   missing acceptance criteria before spending an iteration on a guess.

## Step 1 — Run the pipeline from `entry_phase`

For each phase from `entry_phase` to `review`, in order:

| # | Phase | Subagent | |
|---|---|---|---|
| 0 | analysis | `sdlc-analyst` | |
| 1 | metrics | `metrics-analyst` | skip unless in `optional_phases` |
| 2 | design | `sdlc-architect` | |
| 3 | implementation | `sdlc-developer` | |
| 4 | testing | `sdlc-tester` **‖** `parallel-tester` | second agent skipped unless in `optional_phases` |
| 5 | review | `sdlc-reviewer` | |

### Phase 4 — the concurrent one

When `adversarial` is enabled, launch **both testers in a single message**, as
two `Agent` calls with no dependency between them, so they actually run at once.
Then follow the merge step in `sdlc-protocol`.

Do not sequence them to be safe. Running the parallel tester afterwards hands it
`test-report.md` to read, and a second opinion that has seen the first one is not
a second opinion — it is a review of one. Concurrency is what makes its
independence structural rather than a rule it has to keep.

Before the fan-out:

- reserve **two disjoint finding-id blocks** and name each agent's starting id in
  its own prompt,
- create `.sdlc/inbox/`,
- tell both agents they are in a concurrent phase: findings go to
  `.sdlc/inbox/<agent-name>.jsonl`, resolutions and the work-log entry come back
  in the return value, and neither touches `findings.jsonl` or `work-log.md`,
- name the exact test file each may write. They must not be the same file.

After both return, run the merge step before deciding anything. Until the merge
runs, `findings.jsonl` does not yet contain this phase's findings, so any routing
decision made before it is made on stale data.

If only one of the two returns, re-run the missing one once, then report the
failure. Never merge a half-finished concurrent phase — a missing second opinion
reported as agreement is the worst output this pipeline can produce.

**On Claude Code**, launch with the `Agent` tool, `subagent_type` set to the name
above. **On any other agent**, follow `## Running without subagents` below — the
tool call is the Claude Code path, not a requirement of this skill.

Give the role, in its prompt: the task, the current iteration, the artifact paths
it needs, the **ids of the unresolved findings targeting its phase**, and the
**finding id to start numbering from** (`findings_seq` + 1, per `sdlc-protocol`
§ Allocating finding ids). It reads the rest itself.

Both testers and the reviewer also get `changed_files` from `state.json` in their
prompt. None of them may assume a git repo exists to diff.

Run phases **sequentially, except phase 4** — each consumes the previous one's
artifact. Phase 4 is the exception because its two agents consume the *same*
inputs and produce disjoint outputs; see below.

After each agent returns (after the merge step, for phase 4):

- Read the findings now in `.sdlc/findings.jsonl`, and update `findings_seq` to
  the highest id allocated.
- After `implementation`, record the developer's reported file list in
  `state.json.changed_files`.
- If it produced findings with severity `blocker` or `major` -> **stop the
  pipeline here** and go to step 2. Later phases would review stale work.
- `minor` findings are logged and do not stop anything.
- Otherwise continue to the next phase.

Pipeline reaches the end of `review` with no `blocker`/`major` and a **GO**
verdict -> set `status: "success"` and go to step 4.

## Step 2 — Route the feedback

From the findings that stopped the pipeline, pick the **earliest** phase in
pipeline order among their `target_phase` values — lowest `#` in the phase table,
optional phases included. `testing` and `adversarial` both sort at `#4`. That
becomes the new `entry_phase`.

Rewinding **to** phase 4 re-runs both testers concurrently, as normal. The one
exception: when every unresolved finding at `#4` targets `adversarial` and none
targets `testing`, re-run the parallel tester alone — nothing changed that the
primary suite has not already covered.

A finding targeting a phase not in `optional_phases` cannot be routed: no agent
owns it. Re-route it per the fallback in `sdlc-protocol`'s routing table
(`metrics` -> `analysis`, `adversarial` -> `testing`) rather than enabling the
phase mid-run.

Earliest, not the one that raised them: a missing requirement found by the
tester must go back to the analyst, or the same defect returns next iteration
wearing a different mask.

Increment `iteration`, write `state.json`, and report one line to the user:

```
it2 | testing FAIL 3/6 | F-002,F-003 -> rewind to analysis
```

## Step 3 — Budget check

Each loop skill caps itself at **3 internal revisions**; that budget is the
agent's own and is unrelated to `max_iterations`, which counts pipeline passes.
An agent that exhausts its revisions returns with unresolved items rather than
failing: log them, treat the phase as FAIL, and let the normal routing decide the
rewind. It costs one iteration, the same as any other FAIL. An agent hitting its
revision cap twice on the same finding code counts toward the stuck-loop rule
below.

- `iteration <= max_iterations` -> back to step 1.
- Budget exhausted -> `status: "budget_exhausted"`, go to step 4.
- **Same finding code raised in three consecutive iterations** -> the loop is
  stuck, not converging. Stop, set `status: "blocked"`, and surface it: repeated
  identical failures mean the root cause is mis-routed, and more iterations only
  burn tokens.

## Step 4 — Report

Print, and write to `.sdlc/work-log.md`:

- verdict (`SUCCESS` / `BUDGET EXHAUSTED` / `BLOCKED`) and iteration count,
- what each rewind was for, one line each,
- final test numbers and the reviewer's verdict,
- files changed,
- any finding still unresolved, and any `Q<n>` still open.

Never report success while an unresolved `blocker` sits in `findings.jsonl`.

## Running without subagents

Where subagents are unavailable (Codex, Cursor, Gemini CLI, plain chat), the
pipeline still runs — simulate the isolation instead of spawning it:

1. For each phase, read **only** that role's definition plus its loop skill, then
   act strictly within that role's mandate and hard rules for the whole phase.
2. Write the phase artifact to `.sdlc/` **before** switching roles. Switching
   with unwritten state is how the loop degrades into one long improvisation.
3. Re-read the role file before the testing and review phases even if you think
   you remember it. Judging code you just wrote, in the same breath, is exactly
   the failure this pipeline exists to prevent.
4. Enforce the iteration budget in `state.json` yourself.

Role files live at `.claude/agents/<agent-name>.md`, loops at
`.claude/skills/<loop-name>/SKILL.md`, both named in the phase table above —
note that the two optional roles are `metrics-analyst` / `sdlc-metrics-loop` and
`parallel-tester` / `sdlc-adversarial-loop`, which do not follow the
`sdlc-<role>` pattern the five core phases use.

The concurrent phase is the one that cannot be simulated. Its independence comes
from `test-report.md` not existing yet when the parallel tester derives its
cases — and a single context cannot unsee a file it wrote itself an hour ago.
Where subagents are unavailable: either skip the adversarial agent, or run it in
a genuinely fresh session with only `requirements.md` and `design.md` in front of
it. Running it in the same context and calling the result a second opinion is the
one shortcut that produces a worse outcome than not running it at all, because it
converts "nobody checked" into "two passes agreed".

## Examples

**Reference run (3 iterations, 2 rewinds).**

```
it1 | analysis OK, design OK, implementation OK
    | testing FAIL 3/6 -> F-002..F-006     -> rewind to analysis
it2 | analyst adds R4,R5 | testing OK 6/6
    | review FAIL (undocumented API)       -> rewind to implementation
it3 | developer documents | testing 6/6 | review GO   -> SUCCESS
```

Iteration 1 rewinds to `analysis`, not `implementation`, because two of the five
findings target the analyst — earliest wins.

**Run with both optional agents (metrics-shaped task).**

```
it1 | analysis OK | metrics OK (M1-M4) | design OK | implementation OK
    | testing  || tester 8/8  ,  parallel-tester 7/9
    | merge: agreement 7/7, 1 gap, F-011 impl (confirmed_by: -), F-012 testing
    |                                                    -> rewind to implementation
it2 | developer fixes inclusive-end boundary
    | testing  || tester 9/9  ,  parallel-tester 9/9
    | merge: agreement 9/9, no gap
    | review GO                                          -> SUCCESS
```

The second tester earned its budget in `it1`: the primary suite was green at 8/8
and the boundary defect went straight past it. Both runs cost one iteration
between them, not two, because they ran at once.

Note `it2` reports the agreement count. Without it, "no findings" is ambiguous
between a second pass that checked and a second pass that did nothing.

**Merge catches a contradiction.**
`div_zero` passes in `test-report.md` and raises `ZeroDivisionError` in
`adversarial-report.md` — same case name, same inputs, two results.
No rewind can fix that: the two runs saw different state. Stop at
`status: "blocked"` and surface both commands with both outputs.

**Stuck loop.**
`TEST_FAILED_DIV_ZERO` raised in it2, it3, it4. Stop at it4:
`status: "blocked"`. The same code three times running means the root cause is
mis-routed, and more iterations only burn budget.

**Task too small for this skill.**
"Add a null check to `parse_config`." Do not start a run. Say so and hand it to
an ordinary edit — five phases and a findings ledger cost more than the change.

## Optional user gates

If the user asked for gated mode, call `AskUserQuestion` after **analysis** and
after **review** — approve / revise / abort. Decide gated vs autonomous at
entry, from the user's request, and do not re-litigate it at each gate.

## Rules

- One agent runs at a time, **except phase 4**, whose two agents are launched
  together and kept from colliding by the inbox and the disjoint id blocks in
  `sdlc-protocol`. Never run any other pair concurrently: every other phase
  consumes the previous one's artifact, so there is nothing to overlap.
- Never edit another agent's artifact yourself.
- Never silently drop a finding. Resolved or reported — nothing else.
- Never invent an agent's result. If a subagent fails to return, re-run it once,
  then report the failure.
