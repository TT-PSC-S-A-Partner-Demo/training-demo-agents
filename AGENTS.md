# SDLC Agent Team — Codex / portable entry point

This file is the vendor-neutral version of `.claude/skills/sdlc-orchestrator/`.
Codex reads `AGENTS.md` automatically from the repo root. Copy this file to the
root of the target project, together with the `.claude/` directory (the role and
loop definitions are plain markdown — Codex can read them as instruction files
even though it does not spawn subagents the way Claude Code does).

## What this is

Five role agents driven by an orchestrator through a feedback loop that rewinds
to the phase owning each defect's **root cause**.

```
analysis -> design -> implementation -> testing -> review -> done
    ^          ^            ^             |         |
    +----------+---- feedback ------------+---------+
```

| Phase | Role file | Loop file |
|---|---|---|
| analysis | `.claude/agents/sdlc-analyst.md` | `.claude/skills/sdlc-analyst-loop/SKILL.md` |
| design | `.claude/agents/sdlc-architect.md` | `.claude/skills/sdlc-architect-loop/SKILL.md` |
| implementation | `.claude/agents/sdlc-developer.md` | `.claude/skills/sdlc-developer-loop/SKILL.md` |
| testing | `.claude/agents/sdlc-tester.md` | `.claude/skills/sdlc-tester-loop/SKILL.md` |
| review | `.claude/agents/sdlc-reviewer.md` | `.claude/skills/sdlc-reviewer-loop/SKILL.md` |

Shared contract: `.claude/skills/sdlc-protocol/SKILL.md` — state layout, finding
format, root-cause routing table. Read it before anything else.

The two files per phase do different jobs. The **role file** is the mandate: what
this agent owns, what it must never touch, what it returns. The **loop file** is
the procedure: four to six passes of draft, self-critique and revision, capped at
three revisions, with an explicit exit checklist. Read both — acting on the role
file alone gets you the right scope with none of the rigor, which is how a
review turns into a list of hunches.

## Running without subagent support

The procedure lives in `.claude/skills/sdlc-orchestrator/SKILL.md`, section
`## Running without subagents` — four steps, kept there so installing the skill
folder alone is enough. Follow it; this file does not keep a second copy.

What that section does not explain is **why** the isolation matters, which is
the part people skip:

Claude Code spawns each role as a separate subagent. That separation is not
ceremony — it is the only thing stopping one context from writing the code,
judging the code, and approving the code in a single breath. When you simulate
the roles yourself, nothing enforces the boundary but you.

So the isolation matters most at the tester and reviewer phases. Re-read the
role file before each of those two even when you are sure you remember it. The
failure mode is not forgetting the rules; it is remembering them while quietly
grading your own work against them.

## Codex custom prompt

To get a slash command in Codex, copy the orchestrator skill body into
`~/.codex/prompts/sdlc.md`, then run `/sdlc <task>`.

## Do not

- Skip the tester's execution pass and report tests you did not run.
- Let the developer edit test files.
- Let the reviewer edit source or tests — it writes only its own `.sdlc/` artifacts.
- Route every failure to the developer — root-cause routing is the whole point.
