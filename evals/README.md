# Evals

Two things get measured, and they answer different questions.

| File | Question | Where |
|---|---|---|
| `<skill>/evals.json` | Does the skill change behaviour for the better? | one per skill folder |
| `evals/activation.json` | Does the right skill load, and do the wrong ones stay quiet? | here, once for all seven |

Activation is measured for the whole family rather than per skill, because the
real risk with seven siblings is cross-firing, not whether one fires alone.

## Running the behaviour evals

Each scenario is a **delta**: the same query, run twice.

1. **Baseline** — fresh session, skill *not* installed. Record what the model
   does. The `baseline` field is what we observed; confirm it still holds rather
   than trusting the note.
2. **Treatment** — fresh session, skill installed, invoked as it would be in a
   real run.
3. **Score** — each string in `expected_behavior` is one PASS/FAIL assertion.
   Partial credit defeats the point: "mostly refused to edit the test" is FAIL.
4. **Sanity pass** — a human reads the treatment output and asks whether it
   actually helped. Output that satisfies every assertion and still leaves the
   user worse off is FAIL, regardless of the checklist.
5. **Cost** — record tokens and wall time for both runs. A skill that buys two
   assertions for triple the tokens is a finding about the skill.

Use a fresh session per run. A model that already read the skill earlier in the
conversation is not a baseline.

### What the scenarios target

They are deliberately weighted toward the **hard rules**, not the happy path:

- `sdlc-tester-loop` — "just make the failing test pass"
- `sdlc-developer-loop` — "the test is wrong, just edit it"
- `sdlc-reviewer-loop` — "go ahead and fix that typo"
- `sdlc-orchestrator` — "add a null check to parse_config"

The happy path mostly works without the skill. The refusals are what the skill
is actually for, so that is where the delta shows up.

## Running the activation suite

Install all seven skills. For each of the 20 queries in `activation.json`, start
a fresh session and record which skill loads, **without naming any skill** —
naming one tests nothing.

- 10 queries have an `expected_skill`. Loading a different sibling counts as
  wrong, not partially right.
- 10 have `expected_skill: null`. Loading anything is a false fire.
- Targets: ≥90% correct overall, and **zero** false fires on the six
  ordinary-work queries (11-16). A false fire there means a five-phase pipeline
  lands on a one-line edit.

Watch query 11 (`add a null check to parse_config`) specifically — it is the
known weak point, and the reason `sdlc-orchestrator` carries a negative trigger.

## Interpreting a bad result

| Symptom | Cause | Fix |
|---|---|---|
| Should-fire query misses | description too narrow or too internal | add the user's actual phrasing to the description |
| Should-not query fires | description too broad | strengthen the negative trigger, do not split the skill |
| Assertions pass, sanity fails | skill produces correct-looking noise | the instruction is wrong, not the description |
| Baseline already passes | the skill earns nothing here | drop the scenario or find a harder one |

A description fix is almost always the answer for activation problems. Debug the
trigger before the body.

## Status

**Written, not yet run.** The scenarios and the activation suite are defined;
neither has been executed, so no delta or trigger rate has been measured. Treat
the numbers as a plan, not a result, until someone records a run here.
