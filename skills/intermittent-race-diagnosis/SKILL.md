---
name: intermittent-race-diagnosis
description: >
  Use when a failure is intermittent: a flaky test that passes 8 of 10 on the same input, a
  random failure that works most of the time, a system that sometimes drops events under
  concurrency, a heisenbug, or a bug that only fails under load or in CI. Routes the investigation
  with one split: a random N of M outcome on identical input is a timing race or concurrency bug,
  while a consistent N of N outcome on a given input is a format or type gap, so you stop blaming
  the data and look at timing instead. Covers reproducing the real path faithfully (synthetic
  repros of a non-deterministic bug lie), the read-in-the-gap mechanism where a poll or event loop
  reads a value mid-write and consumes it as empty, ruling out confounded measurement channels
  (dedup, replication and read-after-write lag, buffering, dropped logs) before trusting any count,
  and the bounded re-read that recovers a transient without spinning. Stack-agnostic: web, backend,
  distributed systems, native, mobile.
license: MIT
version: 1.0.0
author: Demiao Chen
tags: [debugging, concurrency, race-conditions, flaky-tests, diagnosis, verification]
---

# Intermittent race diagnosis

A failure that happens sometimes, on input that looks identical each time, is the hardest kind to
chase, because the evidence in front of you is the data and the data is usually not the problem.
The discipline is to classify the failure before forming any theory, reproduce the real path
instead of a synthetic stand-in, and distrust your own measurements before you distrust the system.
Most wasted hours on a flaky bug come from skipping straight to a fix for a mechanism that was never
the cause.

## 1. Classify first: random N of M is a race, consistent N of N is a gap

This single split routes the whole investigation. Before you read any code, get the failure pattern
on identical input: run the same input through the failing path several times and watch what
happens.

| Pattern on identical input | What it means | Where to look |
|---|---|---|
| Random: fails some fraction of the time (2 of 5, 8 of 10, worse under load) | a timing race or concurrency bug. The data is fine; what varies is when this runs relative to another actor | ordering, read/write interleaving, retries, timeouts, contention, shared mutable state |
| Consistent: input A always fails, input B always passes | a gap in what you handle. The inputs differ in a way the code does not cover | a format, encoding, type, size threshold, null, or boundary the parser or branch misses |

The trap is to see a failure and reach for the data theory, because the data is what you can see.
Resist it until you have the pattern. If identical input fails non-deterministically, stop
investigating the data and start investigating timing. (One case: a reader dropped about 2 of 5
items of near-identical content while a sibling tool captured all of them; the instinct was to
blame the payload format, but the 2-of-5 randomness on near-identical content was itself the proof
it was a timing race. The real cause was a read landing mid-write, nothing to do with format.)

"Only fails under load," "only in CI," "only in production," and "cannot reproduce locally" all
point the same way: load and contention compress timing windows, so a gap rarely hit on an idle
machine is hit constantly under pressure. That is a race signature, not a logic bug.

## 2. Reproduce the real path; synthetic repros lie

A hand-built reproduction is worthless until you prove it is faithful to the real trigger's timing.
A synthetic repro that calls a slightly different code path, skips the real timing, or mocks away
the contended resource will reproduce a different bug or no bug, and send you to fix code that was
never the cause. The failure mode is specific and expensive: you build a plausible repro, form a
theory on it, fix the theory, and the real bug survives, because your repro never had it. (One
investigation burned through four progressively wrong theories, each built on an unfaithful
synthetic harness, before instrumenting the real system under the real trigger settled it in a
single trace.)

- Prefer instrumenting the real system over rebuilding it. Add a high-resolution trace to the
  actual path (timestamp, the value observed, the generation or version, the outcome) and capture it
  while the real trigger fires. The real trace is ground truth; a synthetic is a hypothesis.
- If you must synthesize, prove fidelity: diff your harness output against a real trace before you
  trust it. A repro that reproduces the symptom by a different mechanism is worse than no repro,
  because it is convincing and wrong.
- This is the faithful-reproduction discipline from the `drive-the-fix` skill applied to a moving
  target: the trigger is timing, so the repro has to reproduce the timing, not just the inputs.

## 3. The classic mechanism: a read that lands in the gap

The most common intermittent-loss mechanism is a read inside a poll or event loop that observes a
value mid-write and consumes it as final. A writer that updates in two steps (clear, then fill; or
bump a version, then publish the payload) leaves a window where a reader sees the intermediate
state. A reader that treats that intermediate as the end state, advances its cursor, and never
re-reads, loses the write permanently.

Two shapes:

- Read the gap. The writer clears then refills. The poller reads the empty intermediate, records
  "nothing there," advances past that generation, and never sees the payload that lands a beat
  later.
- Block the loop. A synchronous or blocking read inside the loop (a blocking IPC, a blocking DB
  call, a synchronous fetch) freezes the loop while one item is mid-render, so other items arrive
  and leave unseen during the freeze.

Per stack:

- Poller or watcher: reads a record between a delete and the matching insert, or while a field is
  still being populated, and treats the half-written row as final.
- Queue or event consumer: acks or advances the offset before the handler is durable, so a crash or
  a superseding event drops the message.
- Distributed read: a read-after-write hits a replica that has not caught up and returns the
  pre-write value; the caller treats "not found" as "does not exist."
- Concurrent writers: two writers race on one slot; one's clear-then-write straddles the other's
  read.
- Native event loop: a blocking system call (a file read, a socket, a device) inside the run loop
  freezes it mid-operation while new events arrive and are missed.

The tell: the loss correlates with how close in time two operations are, and a slower reader misses
it less, because by the time it reads, the write has landed. If "polling slower fixes it" or
"adding a sleep fixes it," you have a read-in-the-gap race, and the sleep is hiding it, not fixing
it.

## 4. Trust no count: the measurement is often the bug

Before you believe any number that says "N of M arrived," enumerate the channels between the event
and your count that can hide a real loss or fake one. On a hard intermittent bug the measurement is
confounded more often than not, and a wrong count sends you chasing a loss that did not happen or
hides one that did.

Channels to rule out first:

- Deduplication. A repeated identical payload is swallowed as a duplicate by design, so your count
  is low but nothing was lost. Vary the payload per trial so each is distinguishable.
- Read-after-write or replication lag. A WAL, a read replica, a cache, or a CDN can hide a
  just-written record for seconds. The record is there; your fresh read is too early. (One case:
  counting rows right after a write showed 1 of 8 when it was really 8 of 8, because the durable
  read lagged the write by 10 to 15 seconds.)
- Buffering. A write buffer, a batch, or a flush boundary holds the value; you measured before the
  flush.
- Dropped or sampled logs. Logging systems drop low-priority lines under burst, sample, or lag
  their index, so a missing log line is not a missing event. Count the thing through a durable,
  high-priority signal, not a debug line that gets dropped exactly when load is highest.
- Retention, capping, or GC. A size cap, a TTL, or a pruning job removes the evidence between the
  event and your count.

Pick one signal and verify it is reliable: a high-resolution trace at the source, a log level that
is not dropped, or a count taken after the lag window with dedup and capping accounted for.
Distrust counts taken too soon, under burst, or through a layer that mutates them.

## 5. The fix: recover the transient, terminate on the genuine

The fix for a read-in-the-gap race is not a fixed sleep. A sleep hides the race at one timing and
breaks at another. It is a bounded re-read keyed to the same generation: a transient empty or
missing read arms a short, bounded retry of that same version, so a write that lands a beat later
is caught, while a genuine end state still terminates the retry quickly.

The shape, per stack:

- Poller: an empty read of a generation does not consume it as final; it arms a bounded re-read of
  that same generation (a few iterations, a sub-second budget) before advancing. A genuine clear
  stops within the budget; a mid-write gap is recovered.
- Blocking loop: move the blocking read off the loop. Detect the item cheaply and non-blockingly on
  the loop, hand the blocking read to a worker, and let the loop keep observing new events. Run the
  workers concurrently and key each to its own generation; serializing them can make a later item
  observe a superseded state.
- Queue or consumer: ack only after the work is durable; on a transient miss, retry with a bounded
  budget rather than dropping.
- Distributed read: read-your-writes (route the read to the primary, or pass a version token the
  read waits for) or a bounded retry against the replica, instead of treating an early "not found"
  as final.

Hold the invariant in both directions: a transient intermediate is recovered, and a genuine
terminal state still ends the retry promptly. A fix that recovers the gap but never terminates is a
spin; a fix that terminates but never recovers is the original bug. Test both: a write-after-empty
is captured, and a real empty stops within the budget.

## Before you claim it fixed

- You classified the failure on identical input (random equals race, consistent equals gap) and
  chased the right one.
- Your reproduction is faithful to the real trigger's timing, proven against a real trace, not a
  synthetic that reproduces the symptom by a different mechanism.
- You named the mechanism (which two operations race, where the read lands in the gap, what blocks
  the loop), not just "added a retry."
- You ruled out the confounded measurement channels: the loss is real (not a dedup, a lag, a
  buffer, or a dropped log), and the recovery is real (not those same artifacts hiding it).
- The fix recovers the transient and terminates on the genuine end state, with a test for each
  direction.
- You drove the real artifact under the real trigger and read a reliable signal, not a green suite
  alone. A race survives a green suite by definition. See `drive-the-fix` for proving a fix on the
  running artifact, and `adversarial-qa-loop` for looping a sweep until it comes back clean.
