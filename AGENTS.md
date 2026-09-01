# SDLC Agent Team — Codex / portable entry point

This file is the vendor-neutral entry point for
`.claude/skills/sdlc-orchestrator/`. Codex reads `AGENTS.md` automatically from
the repo root and uses the project-scoped custom agents in `.codex/agents/`.
Copy `AGENTS.md`, `.claude/`, and `.codex/` to a target project.
The canonical role and loop definitions remain under `.claude/`; the Codex
profiles and skills are lightweight adapters that point back to them.

## What this is

Five core role agents, plus two optional ones, driven by an orchestrator through
a feedback loop that rewinds to the phase owning each defect's **root cause**.

```
                                                  +-- sdlc-tester ------+
analysis -> [metrics] -> design -> implementation -+                     +- merge -> review -> done
    ^           ^           ^            ^         +-- [parallel-tester] +     |        |
    |           |           |            |                   (concurrent)      |        |
    +-----------+-----------+------------+---------- feedback ------------------+--------+
```

Every arrow back is a rewind. They do not all leave from review: either tester
rewinds to analysis, metrics, design or implementation, and the reviewer rewinds
to any of the five phases before it. The target is always the phase owning the
defect's root cause, never the phase that noticed it.

| # | Phase | Role file | Loop file |
|---|---|---|---|
| 0 | analysis | `.claude/agents/sdlc-analyst.md` | `.claude/skills/sdlc-analyst-loop/SKILL.md` |
| 1 | metrics *(opt)* | `.claude/agents/metrics-analyst.md` | `.claude/skills/sdlc-metrics-loop/SKILL.md` |
| 2 | design | `.claude/agents/sdlc-architect.md` | `.claude/skills/sdlc-architect-loop/SKILL.md` |
| 3 | implementation | `.claude/agents/sdlc-developer.md` | `.claude/skills/sdlc-developer-loop/SKILL.md` |
| 4 | testing | `.claude/agents/sdlc-tester.md` | `.claude/skills/sdlc-tester-loop/SKILL.md` |
| 4 | testing, 2nd lane *(opt)* | `.claude/agents/parallel-tester.md` | `.claude/skills/sdlc-adversarial-loop/SKILL.md` |
| 5 | review | `.claude/agents/sdlc-reviewer.md` | `.claude/skills/sdlc-reviewer-loop/SKILL.md` |

## The two optional phases

Neither runs by default. Each costs an iteration's worth of budget, and the
orchestrator decides once, at entry, from the shape of the task.

**metrics** — enable when the task asks for numbers: usage analytics, telemetry,
KPIs, adoption reporting. It turns "count the active users" into a definition two
analysts compute the same way, with a validation example the tester can assert.
Skipped, that argument still happens — just later, at review, with a dashboard
already built on the losing definition.

**adversarial** — enable for high-risk changes, or when the user asks for an
independent second opinion. It adds a **second lane to phase 4**: both testers
launch at the same time, from the same requirements and design, and neither can
read the other's report because neither report exists yet.

That concurrency is the point. Run the second pass afterwards and it can read
`test-report.md`, and a second opinion that has seen the first one is a review of
one. Run them together and independence stops being a rule somebody has to keep.
The two lanes also cost one iteration between them, not two.

Collisions are prevented structurally, not by discipline: each lane writes its
own artifact, its own assigned test file, and its own finding drop-box under
`.sdlc/inbox/`, numbering from a disjoint id block. Neither touches
`findings.jsonl` or `work-log.md`. The orchestrator merges afterwards — appending
in fixed order, deduplicating defects both lanes found and marking the survivor
`confirmed_by`, and counting agreement.

Two things come out of that merge worth naming. A defect **both** lanes reached
independently is the strongest signal the pipeline produces. And the same case
name with two different results is not a defect to route at all — it means the
runs saw different state, so the orchestrator stops at `blocked` rather than
picking a winner.

Shared contract: `.claude/skills/sdlc-protocol/SKILL.md` — state layout, finding
format, root-cause routing table. Read it before anything else.

The two files per phase do different jobs. The **role file** is the mandate: what
this agent owns, what it must never touch, what it returns. The **loop file** is
the procedure: four to six passes of draft, self-critique and revision, capped at
three revisions, with an explicit exit checklist. Read both — acting on the role
file alone gets you the right scope with none of the rigor, which is how a
review turns into a list of hunches.

## Codex execution

Codex uses the seven project-scoped custom agents in `.codex/agents/`. When the
SDLC orchestrator is invoked, delegate each phase to its matching custom agent;
the orchestrator must not perform a phase itself. Run phases sequentially except
for phase 4: when `adversarial` is enabled, spawn `sdlc-tester` and
`parallel-tester` concurrently and wait for both before the protocol merge.

Use the canonical procedure in
`.claude/skills/sdlc-orchestrator/SKILL.md`. Invoke it explicitly as
`$sdlc-orchestrator <task>`, or ask Codex to run the SDLC team. Current Codex
clients may also select custom agents by name for a single phase.

If a host genuinely has no subagent support, use the orchestrator skill's
`## Running without subagents` fallback. That is a compatibility path, not the
default Codex path. In fallback mode the tester and reviewer boundaries must be
re-established exactly as that section describes; the adversarial lane cannot
be simulated in one context.

## Do not

- Skip the tester's execution pass and report tests you did not run.
- Let the developer edit test files.
- Let the reviewer edit source or tests — it writes only its own `.sdlc/` artifacts.
- Route every failure to the developer — root-cause routing is the whole point.
- Let either tester read the other lane's report, or run the two lanes
  sequentially "to be safe". Sequencing them is what hands the second one the
  first one's answers.
- Let a lane suppress a finding because the other lane probably filed it too.
  Neither can see the other; the merge step deduplicates, and a defect found
  twice independently is worth more than a defect found once.
- Run the metrics phase on a task with no numbers in it, or skip it on a task
  built entirely out of them.
