<!-- delegation-guard — watch subagents for runaway loops and intervene.
     Paste into any prompt that fans work out to parallel agents.
     Why it matters: context is resent every turn, so an agent that loops
     while holding large inputs bills quadratically in turn count. One real
     case: 1 of 3 siblings looped on image reads for 1,160 turns, wrote
     nothing, and cost ~60x the two that finished the same job. -->

<delegation-guard>

<baseline>
Sibling agents doing the same job are a free control. Note each one's
transcript size and turn count as it finishes; the first finisher sets the
expected duration. Absent siblings, cap at [70] MB.
</baseline>

<poll cadence="[every 5 min]">
Watch transcript bytes, not wall-clock — wall-clock cannot tell a slow agent
from a stuck one, but a looping agent accumulates context it never
discharges.

```bash
for f in "$TRANSCRIPT_DIR"/agent-*.jsonl; do
  sz=$(stat -c%s "$f")
  [ "$sz" -gt "$CAP" ] && echo "SUSPECT: $(basename $f) $((sz/1024/1024))MB"
done
```

Trip on either: over the cap, or more than [3x] the median sibling.
</poll>

<interrogate>
On a trip, message the agent (SendMessage) — do not just wait, and do not
kill blind:

  "Status check, answer only this: how many items have you completed, how
   many have you written to [output file], which batch are you on, and which
   item are you reading right now?"

Circling looks like: the same items read again, nothing appended since the
last check, a plan restated rather than advanced, or an answer that names no
number. Progress looks like a rising written-count and a new item id.
</interrogate>

<handle>
- Circling → stop it (TaskStop). Keep whatever it already wrote; relaunch the
  remainder on a cheaper instrument with a batch cadence, not on the same
  instrument with a sterner prompt.
- Slow but advancing → let it run, record the revised ETA, poll again.
- Uncertain → check again in one interval. If the written-count has not moved
  between two checks, treat it as circling.

Kill early: the bill grows every turn a stall survives, so a runaway caught
at 2x baseline costs a fraction of the same runaway at 8x.
</handle>

<prevention>
Give every worker a cadence with a number in it, so a kill never loses much
and silence is legible as failure:

  "Work in batches of about [12] items. For each batch: read the [12], write
   results for every id, and append to [output file] before starting the next
   batch. Never read your whole list before writing anything. Read each item
   exactly once. Report items read, batches used, and reads per item."

An agent that has produced no artifact is not "still working."
</prevention>

</delegation-guard>
