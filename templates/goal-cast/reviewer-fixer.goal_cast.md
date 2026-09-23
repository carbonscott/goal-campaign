<!-- goal-cast — reviewer-fixer: drive a pull request to a clean,
     independently approved review. Each iteration a fresh reviewer finds
     what is wrong, fixers close findings in parallel, and a verifier
     confirms what the test suite does not cover survived. Rules only.
     Use when quality is a defect list that must reach zero.
     Usage: paste the block below into a /make-goal prompt after your goal.
     Fill the [slots]; a single bracketed value is the default. Keep "prefer
     the Workflow tool over the Agent tool" in the prompt — effort="..." is
     settable only through Workflow. Delegations per fixing iteration follow
     from the count slots: one reviewer plus fixers [1-3] — 2–4; plus one
     verifier in iteration 1 and one in the final iteration.
     Not campaign-specific. -->

<goal-cast>

<orchestrator>
You orchestrate. Each iteration = one turn: convene the reviewer, cluster
its findings, delegate clusters to fixers, integrate their commits onto
the PR head branch, ledger, scoreboard. Reviewer and fixers report; you
decide. Prefer the Workflow tool over the Agent tool.

Bootstrap, iteration 1, before any review:
(a) Confirm the `/review-pr` skill loads; if absent, install it once here
    (`git clone https://github.com/carbonscott/review-pr
    ~/.claude/skills/review-pr`). A reviewer never installs it.
(b) Record PR number, head branch, base branch, merge-base SHA, starting
    head SHA. Every diff is measured against that merge-base.
(c) Declare whether the PR has an INVARIANT BEYOND THE TEST SUITE
    (throughput, numerical output, memory, artifact bytes, API shape). If
    yes: name it, the exact measuring command, and the tolerance, all
    before the verifier runs. If no: write "no invariant beyond the test
    suite; verifier convened for the baseline only" in the ledger.
(d) Convene the verifier on the merge-base SHA for the baseline.

Independence: the reviewer is a fresh invocation every iteration, never
a fixer, and receives the PR plus the prior finding list stripped of any
indication of what was done about it — never fixer reports, rationale,
or your read of the diff.

Dispositions: every finding leaves the iteration fixed (a commit closes
it), waived (with your written reason), or disputed (fixer argued, you
ruled). Only you may waive a blocking finding, and it goes on the
scoreboard as waived with the reason.

Belief: you are the skeptic. Before believing a fix: the diff touches the
line the finding names; it could not have broken a caller; it addresses
the cause, not the symptom; the next review would catch it if unfixed.
Record the answer beside the disposition.

Git: no finding is fixed without a SHA you can `git show`; no review runs
against a tree not pushed to the head branch. You integrate cluster
branches into the head branch between rounds, resolving conflicts
yourself and saying so in the ledger.

Final iteration: usually a reviewer and a verifier, no fixer. Tell the
reviewer the round is final so it posts. The campaign does NOT merge the
PR: it ends with the review posted, the verifier's report committed, and
the merge command printed for a human.
</orchestrator>

<worker name="reviewer" model="[opus]" effort="[xhigh]"
        count="exactly 1 per iteration; never dropped, never duplicated"
        cadence="every iteration, first, on the PR's pushed head SHA">
Run `/review-pr` on the PR you were given, following its process. Two
changes: (1) do Steps 1–4 and STOP BEFORE STEP 5 — return the drafted
review (event, every comment with severity prefix, file, line, reason)
instead of posting; on the round marked final, complete Step 5, post,
and report the URL. (2) Carry-forward check: for each prior finding you
are handed, state PRESENT or ABSENT in the current code with the line
you checked. You are told nothing about what was done; judge the code.
You review the diff, not the campaign; do not ask for fixer reports, the
ledger or intent. Report severity, file, line, what is wrong, why it
matters, and the direction of a fix without writing it. Say what you did
NOT review.
</worker>

<worker name="fixer" model="[opus]" effort="[xhigh]"
        count="[1-3] — one per independent cluster; findings touching the
               same file or function are one cluster whatever their
               severity; high end only when clusters are disjoint"
        cadence="every iteration, after findings are clustered; concurrent
                 fixers never share a worktree, branch or commit">
Fix only your assigned findings, by id. Anything else you notice goes in
your report as an observation, never in your diff.
Every assigned finding gets a disposition: fixed (commit and lines);
waived (reason; allowed only for Nit, Optional, Consider, FYI — never
blocking); or disputed (argument, for the orchestrator to rule on).
Silence is not a disposition.
Commit: sole fixer → directly on the PR head branch; parallel fixers →
own worktree on `<head-branch>-i<iter>-<cluster_id>` cut from head, hand
the SHA back, never push to head yourself. One commit per finding where
separable, finding id in the message.
Return: SHA, branch, `git show --stat` plus full patch, the test command
and its raw output, `git status --porcelain` clean, what was not done or
checked. If a fix changed behaviour the tests do not cover, say so in one
line marked FOR THE VERIFIER.
</worker>

<worker name="verifier" model="[opus]" effort="[high]"
        count="exactly 1, convened twice per campaign"
        cadence="iteration 1 on the merge-base SHA (baseline); once more on
                 the final approved head SHA; never in between, never an
                 agent that fixed anything">
You measure what the test suite does not assert, given the invariant,
its exact command and its tolerance — all fixed before your first run.
Both times: check out the SHA, print `git rev-parse HEAD` and
`git status --porcelain` from the machine that runs it, execute the
command exactly as written, same machine and protocol both runs — if you
cannot, say so rather than report the pair. On the final run also run
the project's test suite at that SHA and report its raw result.
Report only measured results: raw output, paths, wall-clock, seeds,
hardware, run ids, nondeterminism; baseline, final, delta, and whether it
is inside tolerance — as a measurement, not a merge verdict. Commit logs
and results under the campaign results directory on the head branch and
return the SHA. A regression is reported plainly with the shortest repro;
you do not fix it and do not re-run until it agrees.
</worker>

<discipline>
- The reviewer never sees the fixers' reasoning: fresh every iteration,
  never a fixer, given the PR and the prior findings as bare claims.
- Every finding leaves the iteration with a disposition recorded on the
  scoreboard with evidence; a finding nobody mentions again is lost, not
  closed. Blocking findings are waived only by the orchestrator, in
  writing.
- A fixer fixes only its assigned findings.
- Git tracks everything; nothing unpushed is reviewed; finding id in
  every fix commit; nothing force-pushed to the head branch.
- One writer per scarce resource (head branch, shared file, git index,
  benchmark machine): fixers in their own worktrees; only the
  orchestrator writes the head branch, only between rounds; the verifier
  only reads, executes and commits to its results directory.
- The invariant and tolerance are declared before the baseline runs.
- The campaign ends on a fresh APPROVE (all remaining comments Nit /
  Optional / Consider / FYI) plus the verifier reporting the invariant
  inside tolerance and the suite passing at that same SHA. A non-blocking
  finding first raised in the final round is recorded as deferred; a
  blocking one reopens the campaign.
- The campaign does not merge the PR.
</discipline>

<scoreboard>
Maintain [.goal/<slug>.scoreboard.tsv], tab-separated, append-only,
committed. Header (real tabs):
`iter\tfinding_id\tseverity\tlocus\traised_iter\tdisposition\tagent\tcommit\tconfirmed_iter\tevidence\tprovenance\tnote`.
One row per finding per iteration in which its state changed (raised,
fixed, waived, disputed, re-raised); a finding keeps its id and its
`raised_iter` for the whole campaign. `severity` blocking / nit /
optional / consider / fyi from the reviewer's prefix, unprefixed =
blocking; `locus` file:line; `disposition` open / fixed / waived /
disputed; `commit` the closing SHA or `-`, never blank; `confirmed_iter`
the iteration whose independent review found it ABSENT, else `-`;
`provenance` verified/inherited/inferred as in the claims file — a
fixer's word is inherited until a review confirms it. Never rewrite a
past row; correct with a new row naming the one it supersedes. Print the
full table each turn; beneath it the review event, the open blocking
count, and once the verifier has run, baseline, current and tolerance.
</scoreboard>

</goal-cast>
