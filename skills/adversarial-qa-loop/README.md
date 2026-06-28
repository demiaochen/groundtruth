# adversarial-qa-loop

Part of [groundtruth](../../README.md).

Ask an AI agent to "find the bugs" and you usually get a long list of plausible problems read
out of the source, most of which are not bugs, while the ones users actually hit go unmentioned.
The problem is method: a single read-through with one lens, no proof, no second opinion, and no
second pass. This skill replaces it with a loop. Parallel finder lanes, each tuned to a
different bug class. A skeptical verifier that rejects by default. A runtime drive that proves
behavioral bugs by running the software. A convergence rule that re-sweeps after every fix. The
agent-facing instructions are in [SKILL.md](./SKILL.md); the reasoning is below.

## Why one lens misses bugs

Reading a module in isolation finds the bugs that contradict themselves: a missing bound, a rule
that matches everything, an unreachable branch. It does not find the bug where a button is wired
to a handler that never runs in the current state, or where two services that should agree
quietly diverge, or where a value goes stale at the seam between two modules that were each
audited alone. Different bug classes need different lenses, so the loop runs four in parallel:

- Static reads each subsystem for internal contradiction.
- Contract-oracle traces every control, shortcut, and endpoint to its handler in every state.
  This is the lane that catches the wired-but-dead class: bound but no effect, or working in one
  state and dead in another.
- Flow/journey owns one end-to-end path across every layer, so the seams nobody owns under a
  per-module split finally get an owner.
- Differential diffs two implementations of one contract (an old path and a new one, two
  backends, a server rule and its client mirror) and flags divergence.

## Why the verifier rejects by default

A finder tuned for recall reports medium-confidence suspicions on purpose. Most teams handle
that by trusting the finder, which fills the tracker with noise, or by telling the finder to
report only certainties, which drops the bugs that matter. The loop separates the two jobs. The
finder optimizes recall. An independent verifier re-derives each finding from source and rejects
unless it can confirm. That keeps precision high without muzzling the finder.

The verdict has three buckets, not two:

- confirmed: real and provable from source now.
- rejected: refuted, already fixed, intended, or code-readable but unproven.
- needs-runtime-check: plausibly real, but only running the software can settle it.

The middle bucket is the point. A behavioral suspicion you cannot prove by reading is not
dropped; it is routed to a drive. Two-bucket review (confirm or reject) silently discards
exactly the class that hurts users.

A second-order signal: if a whole round rejects nothing, the finders under-produced. They
reported only slam-dunks. A healthy round rejects some findings.

## Why running it is the gate

Worked example, stack-neutral. A finder reports: "the confirm action in the edit dialog is bound
to a handler, so it works." The verifier reads the same code, agrees the binding exists, but
cannot see from source whether the keypress reaches the handler when the dialog is open over a
focused field, so it returns needs-runtime-check with a recipe. The drive runs the built app,
opens the dialog, presses the key, and reads the result back. The handler never fired: the
field's editor swallowed the key. Reading the code proved the binding existed. Only running it
proved the binding was dead. A unit test would have called the handler directly and stayed
green, because the bug is in the path to the handler, not in the handler.

That is the whole reason the drive exists, and why a green suite cannot replace it. The recipe
is the same on every stack: stage a known input, force the exact state, trigger through the real
surface, read the output back from the target rather than the source, and log the handler so you
can tell "ran and did the wrong thing" from "never ran." On the web that is Playwright or CDP
reading the live DOM; on a backend it is a real request and the persisted record; on a CLI it is
a PTY and the exit code; on native or mobile it is the OS automation layer and a screenshot or
log line.

## Why it loops

Fixes cause regressions. A change that clamps one value can reorder another; a focus fix in one
view can break a sibling. So the loop re-sweeps after every fix wave, with the differential and
flow lanes as the standing regression net over the seams the fix just touched. Two rules end it:
the top severity must strictly drop each round, and a full sweep must come back clean twice in a
row before you call it done. One empty sweep can be luck.

One more rule keeps the loop honest. A fix that breaks a pinned or locked test means that
behavior was intentional. Revert the fix; the locked test is the spec, not an obstacle.

## A pass, end to end

1. Map the project: its subsystems, its real user journeys, its contracts that have two
   implementations. Derive the lanes from that, not from a fixed list. Build and run the suite;
   it must be green first.
2. Run the lanes in parallel. Collect ranked findings, medium confidence included.
3. Verify each one independently. Sort into confirmed, needs-runtime-check, rejected.
4. Drive every confirmed-behavioral and every needs-runtime-check finding on the running
   software. Drop the ones the drive refutes; promote the ones it confirms.
5. File the survivors, deduped against the known and excluded list. Fix them one at a time, each
   with a test that was red before and green after, each landed before the next.
6. Re-sweep. Confirm the top severity dropped. Repeat until two clean sweeps.

## The common thread

Precision and recall pull in opposite directions, and most QA picks one. This loop gets both by
splitting the work: finders tuned for recall, an independent verifier tuned for precision, a
third bucket so behavioral doubt is run rather than guessed, and a convergence rule so the fixes
that cause regressions get caught by the next sweep. The output is not a list of plausible bugs.
It is a small set of proven ones, fixed, with the proof attached.
