# goal-campaign

How to run a goal campaign with Claude Code's `/goal` and
[`/make-goal`](https://github.com/carbonscott/make-goal), from a minimal prompt
to a fully structured one. The same guide is in [`guide.html`](guide.html) as a
single web page.

```
templates/
  goal-cast/
    advisor-worker.goal_cast.md      general default: plan, do, attack the result
    implementer-runner.goal_cast.md  beat a measured record
    reviewer-fixer.goal_cast.md      drive a PR to a clean review
  delegation-guard.md                watchdog for runaway subagents
guide.html                           this README as a web page
```

---

## Running a goal campaign

A goal campaign is Claude Code's `/goal` loop running against a written
contract. The main agent works in iterations. Each iteration it delegates to a
few subagents, and it stops when every checkable end state is met.

What you need:

- Claude Code with `/goal`
- the [`/make-goal`](https://github.com/carbonscott/make-goal) skill, which
  turns your prompt into the contract
- optionally, the templates in this repo:
  [goal-casts](templates/goal-cast/) and the
  [delegation guard](templates/delegation-guard.md)

Install `/make-goal`:

```bash
git clone https://github.com/carbonscott/make-goal
mkdir -p ~/.claude/skills/make-goal
cp make-goal/SKILL.md ~/.claude/skills/make-goal/SKILL.md
```

The guide climbs four levels. Start at level 0 and add a layer only when the
campaign needs it.

| Level | You add | Use it when |
|---|---|---|
| 0 — minimal | a goal and a budget | the work fits on one machine and one page |
| 1 — `<main>` + resources | resource pointers: hosts, data, services, prior work | agents must reach things they cannot discover |
| 2 — goal-cast | named roles per iteration | results need checking, not just producing |
| 3 — delegation guard | a watchdog for subagents | many agents or long batch jobs run at once |

---

## Level 0 — minimal

A minimal campaign has three parts:

1. `/goal`
2. a prompt that states the goal
3. a budget as two ranges: **X agents per iteration** (e.g. 2–4) and
   **Y iterations** (e.g. 3–9)

The budget is given as ranges so the main agent can choose within them as it
learns more during the campaign.

### Stating the goal

`/make-goal` turns what you write into `done_when`: a list of end states, each
with a proof the agent must print. You can state the goal in one of three ways.

**One real goal.** Say what you want to know or have at the end.

```
/make-goal compared with the main worktree or branch, this dev-min version is
minimal, but is it still as performant for write, read, and search, and what
else?  We didn't really reinvent sqlite, so I would assume it's still good
(our software only adds some structure).  Having said, I want to see real data
from real and stressful experiments.  Please work on it until the result is
ready in @.artifacts.
```
*lnb-min-perf-bench: a benchmark across four scales, met in 3 iterations.*

```
/make-goal i have a loose goal of knowing the areas of interests in quantum
material research.  you have access to @.externals/preprint_servers.json.
Please work on reseaching these preprints [...].  You should work in
iterations, and the budget is 3-9 iterations.  You can use 4-8 subagents
(opus 5.5 at medium) per iteration, and launch them using Workflow tool over
Agent tool.  The whole exploratory happens inside @quantum-materials/
```
*qm-preprint-landscape: a literature survey, met in 3 iterations.*

**Several goals.** List them. Each one becomes its own `done_when` item or
deliverable. Conditions work too ("only if it helps, make a PR").

```
/make-goal read /tmp/test-lnb2/.artifacts/handoffs/lnb-log-perf-feedback.md.
(1) could we create a branch and test the performance in that directory?
will it actually improve things?  (2) conditionally, if it indeed will
improve the performance, make a PR. (3) PR reivew and fix loop until it's
merged.  PR review should be done independently with /review-pr skill
(4) be honest about the trade-offs too.
```
*lnb-log-merge-perf: measure, then conditionally ship, met in 7 iterations.*

```
/make-goal you actually have made it work already.  Please work on this new
cli design, fully test it on a compute node, make a PR, and independently
review and fix the PR until the PR is merged.  Workflow Tool over Agent Tool.
```
*srsvd-cli-layout: build, test on the cluster, and merge.*

**An iteration budget, when "done" is hard to define.** An open-ended goal
like "make it as fast as you can" has no natural finish line. Give it a fixed
number of iterations instead, and ask for the best result at the end. This is
still checkable: the ledger digest prints `iterations N/20` every turn, so the
evaluator can see when the budget is spent.

```
[...] the goal should really be beating the previous best record across.
[...] Let's give it a finite budget of 5 minute steady state, and meausre how
much data have been processed through the end-to-end pipeline.

Please run it for 100 iterations.
```
*srsvd-ooc-throughput: out-of-core throughput on one GPU.*

The contract then pairs the iteration count with a result that has to be
reported whatever happens. From `srsvd-dist-throughput` (20 iterations):

> THE RECORD IS REPORTED EITHER WAY. Name the final champion at each N [...]
> the factor of DIST5_GB(N_max) over the single-node champion [...]

and from `srsvd-ooc-throughput`:

> If no candidate ever beat the origin, THAT IS THE FINDING [...]

This way a campaign that finds no speedup still ends with a clear answer.

### Step 1 — write the contract

Run `/make-goal` in **plan mode** (shift+tab). Plan mode explores the project
before drafting, so the contract uses what is already there.

State the budget and the delegation tool yourself. If you leave them out,
`/make-goal` falls back to its defaults: 2–5 agents per iteration, 3–9
iterations, and the Agent tool. Say "Workflow tool over Agent tool" if you
want to set reasoning effort ("opus at medium", "xhigh"). The Agent tool can
set only the model.

You don't have to write the goal in one go. You can discuss the problem first
in the same session, then hand the conversation over:

```
[a long discussion about scaling srsvd-jax across 2–4 nodes, node-local NVMe
shards, and all-reduce, ending with:]  Let's think about how to properly do this!

/make-goal <main>
could we turn it into a goal campaign?
</main>
```
*srsvd-dist-throughput: 2 nodes at 99% scaling efficiency, 2.08× the
single-node record, 20 of 20 iterations.*

`/make-goal` interviews you for anything missing, then shows a draft. Nothing
is written until you approve. It writes three files:

- `.goal/<slug>.json` is the contract. Each `done_when` item carries its own
  proof, for example:

  > Every cited preprint ID is real, proven by printing the final-turn output
  > of `python quantum-materials/verify_citations.py`, showing [...]
  > `cited IDs: N | found in corpus: N | missing: 0` with N ≥ 18.

  > THE PR EXISTS WITH THE DECLARED SHAPE: the output of `gh pr view <n>
  > --repo carbonscott/srsvd-jax-bench --json number,title,headRefName,
  > baseRefName,state,url` is printed showing headRefName dist and
  > baseRefName main [...]

- `.goal/<slug>.ledger.json` gets one entry per iteration: what was
  delegated, and what is new.
- `.goal/<slug>.claims.json` holds the campaign's key claims, each tagged
  `verified`, `inherited` or `inferred`.

The `/goal` evaluator reads only conversation text. It cannot open files. So
every end state must be something the agent prints.

### Step 2 — run it

Start from a clean context and paste the command `/make-goal` printed:

```
/clear
/goal work until all done_when conditions are met in @.goal/<slug>.json
```

### Step 3 — read the result

Open the claims file and read the `inferred` and `inherited` claims first.
Those are the ones nobody re-checked.

Level 0 is enough for most work. In the campaign archive, 17 of 18 early
campaigns used nothing more and met their goal.

---

## Level 1 — `<main>` and resource pointers

### What a resource pointer is

A resource pointer is a file of checked facts about where things are and how
to reach them. It covers things like remote hosts, cluster accounts,
data locations, service APIs, logs, and the commands that read them. You
reference it with `@file` in the prompt, and every agent can read it when it
needs to.

### Why it helps an autonomous run

- **No one is there to ask.** During a campaign you are not in the loop. Any
  fact the agent lacks becomes a guess, or an iteration spent finding it out.
- **Subagents start cold.** Each subagent begins with no context. A pointer
  file gives every one of them the same map in a single read.
- **It carries over.** Cluster and data facts rarely change between
  campaigns. Write them once, check them once, and reuse the file.
- **It keeps the prompt about the goal.** Facts that don't change live in the
  file. What is specific to this campaign stays in the prompt.

### Examples

| File | Holds | Used by |
|---|---|---|
| `.externals/preprint_servers.json` | the preprint servers the agents may query | qm-preprint-landscape |
| `.ai/s3df.setup.json` | slurm account, partitions, GPU types, how to hold an allocation | srsvd-dist-throughput, zenodo-osti-bench |
| `.ai/bridge.setup.json`, `<slug>.bridge.json` | how to reach the remote host through a bridge session | zenodo-osti-bench, elog-route-expansion |
| `xpp-resources.json` | where DAQ logs, code, and data live, with commands that read them | xpp-epix100-incident-rev2 |

An excerpt from `s3df.setup.json`. Note that it tells agents what *not* to do,
as well as what to do:

```json
"submission_pattern": "Hold the allocation once with `salloc --no-shell`, then
  drive steps into it with `srun --jobid=<id>` as many times as needed -- a
  failed step is another srun, never a re-queue. [...] Do NOT use sbatch, and
  do not drop and re-request the allocation between retries."
```

Write facts you have checked, and say when you checked them. Agents act on
these files without questioning them. So a stale path costs a whole
iteration.

### Using `<main>`

Once you start adding more blocks, wrap the goal itself in `<main>` so the
parts stay separate.

```
/make-goal <main>

We did diagnosis based on the resource pointers in @xpp-resources.json and
produced a report in @xpp-epix100-incident-2026-09-21.html.

Please check out the recent discussion threads in xpp-he-data channel [...]

The goal is to produce an artifact like @xpp-epix100-incident-2026-09-21.html.

The orchestrator agent should have access to @xpp-resources.json, these are
the resource pointers.

</main>
```
*xpp-epix100-incident-rev2: an incident report, met.*

```
You need information about the remote login nodes (which can connect you to
compute nodes through slurm), and it's in @.ai/s3df.setup.json.
```
*From srsvd-dist-throughput.*

You can also ask the campaign to build its own pointer file as it goes. The
single-node throughput campaign asked for one, and the agents kept
`<slug>.setup.json` up to date. They moved each fact from `inherited` to
`verified` once they had checked it on the node.

```
You need to collect setup information you need to run the campaign - remote,
remote storage (wekafs, nvme, etc), bridge, slurm, data.  Data can be
artificially generated on disk.
```
*From srsvd-ooc-throughput.*

Two more inputs are worth passing in `<main>`:

- A **handoff doc** from an earlier session. For example, srsvd-dist-merge
  started with "read .ai/handoffs/merge-dist-harness-to-main.md".
- An **earlier contract**, when this campaign continues one. For example,
  "We have done a good campaign associated with
  @../runtime-b/.goal/srsvd-ooc-throughput.json."

---

## Level 2 — goal-cast

A goal-cast gives each iteration a fixed shape: named roles, each with a
model, an effort level, and a cadence. Paste it after `</main>`. Fill in the
`[slots]` and delete any role you don't need.

| Cast | Use when | Campaigns that used it |
|---|---|---|
| [advisor-worker](templates/goal-cast/advisor-worker.goal_cast.md) | general default: plan, do, then attack the result | zenodo-osti-bench, fix-issues-onboard, elog-search-skill |
| [implementer-runner](templates/goal-cast/implementer-runner.goal_cast.md) | a number is the goal, measured against a champion | srsvd-ooc-throughput (an earlier JSON version) |
| [reviewer-fixer](templates/goal-cast/reviewer-fixer.goal_cast.md) | a PR must reach a clean review | srsvd-dist-merge, pr1-review-merge |

### Advisor-worker

The shape, trimmed (see the file for the full text):

```
<goal-cast>
<orchestrator>
You are the orchestrator. Each iteration: think, decompose, delegate,
integrate, ledger. Every decision is yours — advisors advise [...]
Prefer the Workflow tool over the Agent tool whenever Workflow is available.
</orchestrator>

<advisor name="planner" model="[fable]" effort="[xhigh]"
         cadence="iteration 1; any iteration that changes direction [...]">
Consulted BEFORE a decomposition is committed [...] Run a pre-mortem [...]
</advisor>

<advisor name="skeptic" [...] cadence="every iteration that produces a
         measurement or claim">
Consulted AFTER results, BEFORE belief [...] Verdict per claim: BELIEVE, or
QUARANTINE with the reason.
</advisor>

<workers model="[opus]" count="[2-4] per iteration [...]" effort="[xhigh]">
Return evidence, not conclusions [...]
</workers>

<worker name="cold-reader" [...]>
Given the deliverable and NOTHING else [...] attempt to use it end to end.
</worker>

<discipline>
- Predict before you measure [...]
- Quarantine has teeth [...]
</discipline>
</goal-cast>
```

Advisors count toward the per-iteration budget. So an iteration that brings
in the planner and the skeptic has fewer slots left for workers.

### Implementer-runner: beating a performance record

Implementers write candidate changes. One runner measures every candidate next
to the current champion, re-measured in the same iteration. A candidate
becomes the new champion only if it wins by more than the noise floor.

The single-node srsvd-jax campaign used this cast. You can tune a cast for one
campaign without touching the template. Here the user changed the implementer
count and added a setup phase after seeing the draft:

```
- The orchestrator can spend 1-3 iterations to set up the campaign like
  generating data for the campaign on nvme.
- Originally, it's 1 implementer per iteration, but you can have 1-3
  implementers (opus, xhigh) per iterations in case the change is large (do
  not change the base goal cast json, but in the campaign specific goal cast
  json)
- Use 4 bridge session max to mitigate contention from subagents and main
  agent.
```
*srsvd-ooc-throughput. The champion reached 2.54× the starting throughput,
at 94% of the one-GPU read limit.*

### Reviewer-fixer: landing a PR

Each iteration a fresh reviewer reads the PR, fixers close the findings in
parallel, and a verifier checks that nothing the tests miss has broken. The
prompt can be as short as the goal:

```
/make-goal <main>

you decide how many PRs, but I want to merge them, since we have identified a
champion now.

</main>

<goal-cast>
[reviewer-fixer, slots filled]
</goal-cast>
```
*srsvd-dist-merge: 3 iterations took the multi-node benchmark harness to an
APPROVE verdict from a fresh reviewer. GitHub won't let an author approve
their own PR, so the verdict was posted as a comment. The user then
authorized the merge.*

---

## Level 3 — delegation guard

When many agents run at once, or a worker processes a long list, one looping
agent can cost many times what the rest cost together. In one real case, one
of three sibling agents looped for 1,160 turns, wrote nothing, and cost
~60× what the other two did.

Paste
[`delegation-guard.md`](templates/delegation-guard.md)
after the goal-cast. It tells the orchestrator to:

- **Poll** transcript sizes (not wall-clock time) on a fixed schedule.
- **Question** any agent that is past a size cap or 3× the size of its
  siblings.
- **Stop** agents that are going in circles.
- **Give** every worker a batch size, so an agent that writes no output
  shows up as a failure early.

The part that prevents most problems is the batch rule. Every worker gets it:

```
"Work in batches of about [12] items. For each batch: read the [12], write
 results for every id, and append to [output file] before starting the next
 batch. Never read your whole list before writing anything. [...]"
```

The full level-3 prompt is laid out like this:

```
/make-goal <main>
...goal, budget, "Workflow over Agent", @resource pointers...
</main>

<goal-cast>
...a cast from goal-cast/, slots filled...
</goal-cast>

<delegation-guard>
...delegation-guard.md, slots filled...
</delegation-guard>
```

Campaigns that used all three blocks:

- **srsvd-ooc-throughput:** long GPU runs on one node.
- **zenodo-osti-bench:** 32 datasets registered in parallel, met in 6
  iterations.
- **elog-route-expansion:** many API routes implemented by 2–4 agents per
  iteration.

---

## Short reference

- Install [`/make-goal`](https://github.com/carbonscott/make-goal). Run it in
  plan mode.
- State one goal, several goals, or an iteration budget with a report that is
  due whatever happens.
- State the agent budget and "Workflow over Agent" in the prompt.
- Approve the draft, then `/clear` and paste the `/goal` command.
- Read `inferred` and `inherited` claims first.
- Add levels only as needed: resource pointers when agents can't reach
  something, a cast when results need checking, a guard when agents run at
  scale.
- Split big work into chained campaigns. For example, the single-node
  throughput campaign, then the multi-node campaign that built on it, then a
  merge campaign that landed the result.
