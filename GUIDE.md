# Running a goal campaign

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

### Step 1 — write the contract

Run `/make-goal` in **plan mode** (shift+tab). Plan mode explores the project
before drafting, so the contract uses what is already there.

State the budget and the delegation tool yourself. If you leave them out,
`/make-goal` falls back to its defaults: 2–5 agents per iteration, 3–9
iterations, and the Agent tool.

A real level-0 prompt (`qm-preprint-landscape`, met in 3 iterations):

```
/make-goal i have a loose goal of knowing the areas of interests in quantum
material research.  you have access to @.externals/preprint_servers.json.
Please work on reseaching these preprints [...].  You should work in
iterations, and the budget is 3-9 iterations.  You can use 4-8 subagents
(opus 5.5 at medium) per iteration, and launch them using Workflow tool over
Agent tool.  The whole exploratory happens inside @quantum-materials/
```

Why "Workflow tool over Agent tool": the Agent tool can set only the model.
Reasoning effort ("at medium", "xhigh") can be set only through Workflow.

`/make-goal` interviews you for anything missing, then shows a draft. Nothing
is written until you approve. It writes three files:

- `.goal/<slug>.json` is the contract. Its key part is `done_when`, a list of
  end states. Each one carries its own proof. From the example above:

  > Every cited preprint ID is real, proven by printing the final-turn output
  > of `python quantum-materials/verify_citations.py`, showing [...]
  > `cited IDs: N | found in corpus: N | missing: 0` with N ≥ 18.

- `.goal/<slug>.ledger.json` gets one entry per iteration: what was
  delegated, and what is new.
- `.goal/<slug>.claims.json` holds the campaign's key claims, each tagged
  `verified`, `inherited` or `inferred`.

The `/goal` evaluator reads only conversation text. It cannot open files. So
every end state must be something the agent prints. The ledger digest printed
each turn is what makes your iteration range enforceable.

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

### Worked example: maximizing srsvd-jax throughput

`srsvd-ooc-throughput` set out to beat the best steady-state throughput of
srsvd-jax (a randomized SVD library written in JAX) on one GPU, with a
dataset twice the size of host memory. The prompt, with details trimmed:

```
/make-goal

we have done a lot of work to improve the performance of srsvd-jax (right
now, codebase at @work/srsvd-jax at the whole-job branch, which was a result
of running the @.goal/srsvd-whole-job.json campaign).  [...]  I wrote down my
lessons in @.ai/research/performance-optimization-method.json.  [...]

you could easily get some gpu node (called ada node) allocation [...].  you
should be able to get even exclusive node (full memory, full CPU, and full
GPU, the whole node) easily, and I suggest you do that unless the node
becomes super busy.

You need to collect setup information you need to run the campaign - remote,
remote storage (wekafs, nvme, etc), bridge, slurm, data.  Data can be
artificially generated on disk.

SRSVD-JAX uses JAX heavily, so you might want to do some specific deep dive
into the JAX repo [...] before you even plan what to do.

The specific scenario I want to optimize is the out of core scenario, where
data are much larger than not only the GPU memory on a single GPU card, but
the host memory too.  We should consider using one synthetic data set which
is 2X host memory.

[...] the goal should really be beating the previous best record across.
[...] We should really measure a steady state throughput (excluding cold
start like compilation, for example).  Let's give it a finite budget of 5
minute steady state, and meausre how much data have been processed through
the end-to-end pipeline.

Please run it for 100 iterations.

In each iteration, please follow @.goal/base.goal_cast.json.  Actually, leave
this base goal cast file alone, also create your own goal cast file for this
campaign specifically.

<delegation-guard>
[... delegation-guard.md, slots filled ...]
</delegation-guard>
```

What each part of the prompt does:

- **The goal** is a measured record ("beating the previous best record"),
  with the measurement defined: data processed in a 5-minute steady-state
  window.
- **"Please run it for 100 iterations"** is the stopping rule. "As fast as
  possible" has no natural finish line, so a fixed number of iterations
  replaces one. The contract paired it with a result that must be reported
  either way: "If no candidate ever beat the origin, THAT IS THE FINDING."
- **Resource pointers** are the earlier contract, the method notes, and the
  instruction to collect a setup file.
- **The goal-cast** is an earlier JSON version of implementer-runner. After
  seeing the draft, the user tuned it with a follow-up message (quoted in
  Level 2).
- **The delegation guard** is pasted in full, because each iteration runs
  long GPU jobs.

Result: the champion reached 2.54× the starting throughput, at 94% of the
one-GPU read limit. The scoreboard ends at iteration 21 of the 100 requested,
so this is the best result reached, not a finished 100-iteration run.

Other campaigns that used all three blocks:

- **zenodo-osti-bench:** 32 datasets registered in parallel, met in 6
  iterations.
- **elog-route-expansion:** many API routes implemented by 2–4 agents per
  iteration.

---

## Short reference

- Install [`/make-goal`](https://github.com/carbonscott/make-goal). Run it in
  plan mode.
- State one goal, several goals, or, when "done" is hard to define, a fixed
  number of iterations with a report that is due either way.
- State the agent budget and "Workflow over Agent" in the prompt.
- Approve the draft, then `/clear` and paste the `/goal` command.
- Read `inferred` and `inherited` claims first.
- Add levels only as needed: resource pointers when agents can't reach
  something, a cast when results need checking, a guard when agents run at
  scale.
- Split big work into chained campaigns. For example, the single-node
  throughput campaign, then the multi-node campaign that built on it, then a
  merge campaign that landed the result.
