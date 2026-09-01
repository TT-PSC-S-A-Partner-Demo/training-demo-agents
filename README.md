# SDLC Agent Team — 7 roles + orchestrator + skill loops

An SDLC team for use in Claude Code, Codex and Devin. The canonical roles and
skills live in `.claude/`; `.codex/` and `.devin/` hold native profiles or thin
pointers back to them. Every agent has **its own skill loop** — an internal
cycle of passes with self-critique and exit criteria. Above them runs the
orchestrator with its feedback loop.

```
analysis -> design -> implementation -> testing -> review -> done
    ^          ^            ^             |         |
    +----------+---- feedback ------------+---------+
```

## Structure

```
.claude/
  agents/                      <- 7 canonical role definitions
    metrics-analyst.md
    parallel-tester.md
    sdlc-analyst.md
    sdlc-architect.md
    sdlc-developer.md
    sdlc-tester.md
    sdlc-reviewer.md
  skills/
    sdlc-protocol/             <- shared contract (state, findings, routing)
    sdlc-orchestrator/         <- manager, /sdlc-orchestrator
    sdlc-metrics-loop/         <- optional metric definitions
    sdlc-adversarial-loop/     <- optional second test lane
    sdlc-analyst-loop/            <- 4 passes
    sdlc-architect-loop/          <- 5 passes
    sdlc-developer-loop/          <- 6 passes
    sdlc-tester-loop/             <- 6 passes
    sdlc-reviewer-loop/           <- 5 passes
      SKILL.md + evals.json       <- every skill carries its own evals
evals/activation.json          <- trigger-rate suite for the whole family
.codex/
  config.toml                  <- multi-agent + explicit skill registration
  agents/*.toml                <- 7 project-scoped custom agents
  skills/*/SKILL.md            <- pointers to the canonical skills
.devin/                        <- profiles and pointers for Devin
AGENTS.md                      <- orchestration instructions for Codex
```

## Evals

Every skill has an `evals.json`: 4-5 scenarios, each with a `query`,
`expected_behavior` (PASS/FAIL assertions) and a `baseline` (what the model does
without the skill). The scenarios target hard rules, not the happy path — for
`sdlc-tester-loop`, *"just make the failing test pass"* must end in a refusal to
edit production code. That is the part that justifies the skill existing at all.

`evals/activation.json` — 20 queries (10 fire / 10 no-fire) measuring the
trigger rate of all seven at once. Measured together, because the real risk is
overshooting between siblings, not firing in isolation. Target: ≥90% hits and
zero false fires across the six ordinary-work queries.

## Installation

**Per project** (recommended — the team sees it in the repo):

```bash
cp -r sdlc-agents/.claude/agents/*        <project>/.claude/agents/
cp -r sdlc-agents/.claude/skills/sdlc-*   <project>/.claude/skills/
cp -r sdlc-agents/.codex                   <project>/          # Codex
cp    sdlc-agents/AGENTS.md                <project>/          # Codex
```

**Globally** (all projects): the same, into `~/.claude/agents/` and
`~/.claude/skills/`.

After installing, decide what to do with `.sdlc/` in the target project — the
orchestrator creates it on the first run. Version it if you want an audit trail
(who reported what, in which iteration, how it was resolved). Add it to
`.gitignore` if you treat it as working state. My default suggestion is to
**version it** — `findings.jsonl` is append-only for exactly that reason, and in
an argument about whether something was tested, it is the only evidence.

`evals/` stays in this repo — do not copy it into the target project. It covers
the skills themselves, not the code you build with them.

Verification in Codex: the profiles in `.codex/agents/` cover 7 roles, and
`/skills` shows 9 `sdlc-*` skills. The project must be marked as trusted for
Codex to load the project-scoped `.codex/config.toml`.

## Running it

Claude Code:

```text
/sdlc-orchestrator implement an RPN calculator with a single entry point evaluate(expr)
```

Codex:

```text
$sdlc-orchestrator implement an RPN calculator with a single entry point evaluate(expr)
```

The orchestrator creates `.sdlc/`, runs the phases in order as subagents,
collects findings, rewinds to the phase owning the root cause, and repeats until
clean or until the budget runs out (`max_iterations`, default 6).

A single role can also be invoked on its own, e.g. review only:
`Use the sdlc-reviewer agent on the current diff`.

## Skill loops — one per agent

Each loop is a set of `draft -> self-critique -> revise` passes with a hard limit
of **3 revisions** and an exit checklist. The limit exists so an agent does not
grind in circles at the cost of the orchestrator's iteration budget.

| Agent | Passes | Core of the self-critique |
|---|---|---|
| analyst | harvest → draft → testability challenge → revise | can the tester write an assertion from this? |
| architect | reuse survey → draft → simplification → trace → revise | which `R<n>` dies if I remove this? |
| developer | context → triage → implement → **run** → self-review → revise | root cause or symptom? |
| tester | derive → boundary sweep → write → **execute** → classify → report | whose root cause is this? |
| reviewer | scope → 6-axis sweep → contract check → **verify** → verdict | can I name a concrete failure scenario? |

The bolded passes cannot be dropped: the developer and tester **actually
execute** code, and the reviewer **verifies** every finding candidate before
reporting it.

## The feedback loop — routing by root cause

A finding carries `target_phase` = the phase that owns the **fix**, not the one
that noticed the problem.

| Symptom | target_phase |
|---|---|
| behavior nobody specified | `analysis` |
| the spec exists, the structure cannot carry it | `design` |
| spec and design fine, code wrong | `implementation` |
| missing test case | `testing` |

One defect may require **two** findings (missing requirement + missing code).
Sending everything to the developer turns the feedback loop into a retry loop —
the missing requirement comes back in the next iteration.

Severity: `blocker` stops the pipeline, `major` rewinds, `minor` only logs.

## State on disk

```
.sdlc/
  state.json        phase, iteration, budget, status
  requirements.md   analyst
  design.md         architect
  test-report.md    tester (with real runner output)
  review.md         reviewer (GO / NO-GO)
  findings.jsonl    append-only, audit trail
  work-log.md       one entry per agent pass
```

Findings are never deleted — they are rewritten to `"resolved": true` with a
`"resolution"` field. Source code goes into the project tree, not into `.sdlc/`.

## Safeguards

- The developer does not touch tests. The tester does not touch production code.
  The reviewer touches nothing.
- The tester does not report a result it did not observe.
- The same finding code in 3 consecutive iterations → `blocked`, stop. The loop
  is not converging; the root cause is being addressed wrongly.
- The orchestrator never performs a phase itself — that destroys the
  independence of the judgments.

## Codex

Codex uses the custom agents in `.codex/agents/` natively. It runs the phases
sequentially, except for the two independent testers, which run in parallel. The
no-subagent mode remains strictly a fallback for hosts that genuinely do not
offer delegation.

## Reference implementation

The same pipeline also exists as a deterministic Python program (stdlib,
`python main.py`) — it shows finding routing dry, without an LLM. **It is not
part of this repo**; it sits next to it, in the `sdlc-orchestrator/` directory of
the same workspace. If you clone the bundle alone you will not get that code, and
you lose nothing — the agents are complete without it.
