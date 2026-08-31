---
name: sdlc-orchestrator
description: Runs a task through a five-agent SDLC pipeline - analyst, architect, developer, tester, reviewer - rewinding to the phase that owns each defect's root cause until the pipeline is clean or the iteration budget runs out. Use when implementing a feature end to end with full rigor, when a change must be specified, built, tested and reviewed independently, or when asked to run the SDLC team. Do not use for single-file edits, quick bug fixes, or questions about existing code - the five-phase loop costs far more than those tasks are worth.
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
analysis -> design -> implementation -> testing -> review -> done
    ^          ^            ^             |         |
    +----------+---- feedback ------------+---------+
```

## Step 0 — Initialize

1. `Skill(sdlc-protocol)` — load the shared contract. Everything below assumes it.
2. Create `.sdlc/` and write `state.json`:

```json
{"task":"<task text>","iteration":1,"max_iterations":6,"entry_phase":"analysis","status":"running"}
```

3. Create empty `findings.jsonl` and `work-log.md`.
4. If the task text is one line of vague intent, ask the user **once** for the
   missing acceptance criteria before spending an iteration on a guess.

## Step 1 — Run the pipeline from `entry_phase`

For each phase from `entry_phase` to `review`, in order:

| Phase | Subagent |
|---|---|
| analysis | `sdlc-analyst` |
| design | `sdlc-architect` |
| implementation | `sdlc-developer` |
| testing | `sdlc-tester` |
| review | `sdlc-reviewer` |

**On Claude Code**, launch with the `Agent` tool, `subagent_type` set to the name
above. **On any other agent**, follow `## Running without subagents` below — the
tool call is the Claude Code path, not a requirement of this skill.

Give the role, in its prompt: the task, the current iteration, the artifact paths
it needs, and the **ids of the unresolved findings targeting its phase**. It
reads the rest itself.

Run phases **sequentially** — each consumes the previous one's artifact.

After each agent returns:

- Read the findings it appended to `.sdlc/findings.jsonl`.
- If it produced findings with severity `blocker` or `major` -> **stop the
  pipeline here** and go to step 2. Later phases would review stale work.
- `minor` findings are logged and do not stop anything.
- Otherwise continue to the next phase.

Pipeline reaches the end of `review` with no `blocker`/`major` and a **GO**
verdict -> set `status: "success"` and go to step 4.

## Step 2 — Route the feedback

From the findings that stopped the pipeline, pick the **earliest** phase in
pipeline order among their `target_phase` values. That becomes the new
`entry_phase`.

Earliest, not the one that raised them: a missing requirement found by the
tester must go back to the analyst, or the same defect returns next iteration
wearing a different mask.

Increment `iteration`, write `state.json`, and report one line to the user:

```
it2 | testing FAIL 3/6 | F-002,F-003 -> rewind to analysis
```

## Step 3 — Budget check

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

Role files live at `.claude/agents/sdlc-<role>.md`, loops at
`.claude/skills/sdlc-<role>-loop/SKILL.md`.

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

- One agent runs at a time. No parallel phases; they share `.sdlc/` state.
- Never edit another agent's artifact yourself.
- Never silently drop a finding. Resolved or reported — nothing else.
- Never invent an agent's result. If a subagent fails to return, re-run it once,
  then report the failure.
