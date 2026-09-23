<!-- goal-cast — implementer-runner: beat the current best record.
     Each iteration implementers write candidates, one runner measures them
     all beside a re-measured champion. Rules only.
     Use when the number IS the goal and a measurement replaces the judge.
     Outer-loop variant (evolve a prompt template over a task set) is this
     cast unchanged: the candidate is the template, the runner runs the
     inner loop per task and reports the mean, a held-out set is control.
     Usage: paste the block below into a /make-goal prompt after your goal.
     Fill the [slots]; a single bracketed value is the default. Keep "prefer
     the Workflow tool over the Agent tool" in the prompt — effort="..." is
     settable only through Workflow. Delegations per iteration follow from
     the count slots: implementers [1-3] plus exactly one runner — 2–4.
     Not campaign-specific. -->

<goal-cast>

<orchestrator>
You orchestrate. Each iteration = one turn: decide what to try, delegate,
integrate, ledger, scoreboard. Workers execute and report; you pick each
implementer's candidate and keep concurrent ones from converging. Prefer
the Workflow tool over the Agent tool.

Iteration 1: record the pre-campaign HEAD SHA; hand it to the runner as
the control and baseline.

The record: the ledger names the champion — artifact path, verified
score, commit SHA, exact command. A candidate replaces it only after the
runner measured it and you read the raw output yourself. A worker's
report is provenance "inherited" until you re-verify; re-verify anything
a later iteration stands on.

Belief: you are the skeptic. Before believing a number: right direction
for the predicted change; not too big or small to be true; books balance
(components sum, counts and sizes match); could the check have failed at
all. Record the answer beside the claim.

Git: no candidate enters the scoreboard without a SHA you can `git show`.
Merge only the iteration's new champion into the campaign branch; losing
candidates stay on their branches.
</orchestrator>

<worker name="implementer" model="[opus]" effort="[xhigh]"
        count="[1-3] — one per independent candidate; high end only when
               mechanisms genuinely differ and each has its own worktree"
        cadence="every iteration, first; concurrent implementers never
                 share a worktree, branch or commit">
One implementer, one candidate: the one assigned to you under its
candidate id, given the champion, its score, its SHA and the ledger's
record of what failed. Never blend with a sibling's.
Before the runner touches it, state the mechanism you expect to produce
the gain and the predicted score or band.
Commit every touched file on your own branch — the campaign branch if you
are the only implementer, else `<campaign-branch>-i<iter>-<candidate_id>`
cut from its head — message naming iteration and candidate id. Nothing on
main, nothing force-pushed, nothing uncommitted.
Return: SHA, branch, artifact path, `git show --stat` plus full patch,
exact run command, `git status --porcelain` clean, and what was not done
or not checked. Do not run the experiment; do not report a score.
</worker>

<worker name="runner" model="[opus,sonnet]" effort="[medium]"
        count="exactly 1 per iteration; never dropped, never duplicated"
        cadence="every iteration, after all implementers' commits land">
Check out each candidate's SHA — not the branch, not the working tree —
and print `git rev-parse HEAD` and `git status --porcelain` from the
machine that runs it. Run each candidate exactly as specified and, same
environment and protocol, re-run the champion at its recorded SHA as the
control — one control per iteration. Iteration 1: measure the
pre-campaign HEAD as both control and baseline.
Measure one at a time on an exclusive resource; if you interleave, say
so. Report only real measured results: raw output, run ids, SHAs,
absolute paths, wall-clock, seeds, hardware, nondeterminism across
repeats, and what was not measured (skips, timeouts, single-seed).
Do not fix a candidate: on failure report the error verbatim with the
shortest repro, commit the failing log, continue with the others.
Commit raw logs, results and command lines under the campaign results
directory on the campaign branch — one directory per iteration, one
subdirectory per candidate id; `git checkout <campaign-branch>` first,
since checking out a SHA detaches HEAD. Return that SHA and the paths.
</worker>

<discipline>
- Predict before you measure: each candidate's predicted score is in the
  iteration's ledger "predictions" block before the runner starts.
- Git tracks everything; nothing uncommitted is measured; nothing lands on
  main, nothing is force-pushed. A result not recoverable by
  `git checkout <sha>` does not enter the scoreboard.
- Same conditions, same iteration: champion and candidates measured
  together, each at its own SHA. A carried-over number is not a control.
- Parallel candidates compete, never combine. Two that both beat the
  champion: the higher takes the single `record` Y; the runner-up stays
  unmerged, its gain unbanked until re-implemented and re-measured.
  Stacking two mechanisms is the next iteration's single candidate.
- Separation of duty: whoever writes a solution does not report its
  score. Never merge the roles; never let an implementer measure its own.
- One writer per scarce resource (GPU, deploy target, shared file, git
  index): each implementer in its own worktree; the runner only reads,
  executes and commits to its results directory. If you pipeline (runner
  on N-1 while implementers write N), say so in ledger and note column.
- Unverified claims have no standing: a number you could not reproduce
  or explain makes no champion and no premise.
</discipline>

<scoreboard>
Maintain [.goal/<slug>.scoreboard.tsv], tab-separated, append-only,
committed. Header (real tabs):
`iter\tcandidate_id\tagent\tbranch\tcommit\tartifact\tcommand\tpredicted\tmeasured\tcontrol\tdelta_vs_champion\trecord\trun_id\tprovenance\tnote`.
One row per measured candidate, in the iteration that measured it;
iteration 1 also carries a baseline row for the pre-campaign HEAD.
`predicted` copied from the ledger, never edited; `measured` and
`control` blank only on a failed run, with `note` saying why; `record` Y
on at most one row per iteration, latest Y is the champion; `run_id` is
the environment's identifier or `-`, never invented; `provenance`
verified/inherited/inferred as in the claims file. Never rewrite a past
row — correct with a new row naming the one it supersedes. Print the
full table each turn, champion row named beneath it by iteration, agent
and SHA.
</scoreboard>

</goal-cast>
