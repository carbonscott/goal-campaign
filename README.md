# goal-campaign

A guide and templates for running goal campaigns with Claude Code.

A goal campaign is a written contract that a main agent works through in
iterations. Each iteration it delegates to a few subagents, and it stops when
every checkable end state is met. The contract is written by the
[`/make-goal`](https://github.com/carbonscott/make-goal) skill. You run it
with Claude Code's `/goal` loop, or as a plain prompt.

**Read the guide:** [carbonscott.github.io/goal-campaign](https://carbonscott.github.io/goal-campaign/)
(also available as [`GUIDE.md`](GUIDE.md)).

## Quick start

Install `/make-goal`:

```bash
git clone https://github.com/carbonscott/make-goal
mkdir -p ~/.claude/skills/make-goal
cp make-goal/SKILL.md ~/.claude/skills/make-goal/SKILL.md
```

In Claude Code, switch to plan mode (shift+tab) and state the goal and the
budget:

```
/make-goal <your goal>.  Work in iterations, budget 3-9 iterations, 2-4
subagents per iteration.  Use the Workflow tool over the Agent tool.
```

Approve the draft, then start the loop with the command it prints:

```
/clear
/goal work until all done_when conditions are met in @.goal/<slug>.json
```

For bigger campaigns, add the templates below to the `/make-goal` prompt. The
guide explains when each one helps.

## Contents

| Path | What it is |
|---|---|
| [`GUIDE.md`](GUIDE.md) | the guide, from a minimal prompt to a fully structured one |
| [`index.html`](index.html) | the same guide as a web page (served by GitHub Pages) |
| [`templates/goal-cast/advisor-worker.goal_cast.md`](templates/goal-cast/advisor-worker.goal_cast.md) | general default cast: plan, do, then attack the result |
| [`templates/goal-cast/implementer-runner.goal_cast.md`](templates/goal-cast/implementer-runner.goal_cast.md) | cast for beating a measured record |
| [`templates/goal-cast/reviewer-fixer.goal_cast.md`](templates/goal-cast/reviewer-fixer.goal_cast.md) | cast for driving a PR to a clean review |
| [`templates/delegation-guard.md`](templates/delegation-guard.md) | watchdog for runaway subagents |
| [`images/`](images/) | figures used by the guide |

## License

[MIT](LICENSE)
