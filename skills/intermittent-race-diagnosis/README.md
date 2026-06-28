# intermittent-race-diagnosis

Part of [groundtruth](../../README.md).

An intermittent failure (sometimes it drops, mostly it works, the test passes 8 of 10 on the same
fixture) is the hardest kind to chase, because the evidence in front of you is the data, and the
data is usually not the problem. This skill front-loads one decision that routes the whole
investigation, then keeps you from fixing a mechanism that was never the cause. The agent-facing
instruction is in [SKILL.md](./SKILL.md); the reasoning is below.

## The one split that routes everything

Run identical input through the failing path several times and watch the pattern. Do this before
reading any code.

| Pattern on identical input | What it is | First move |
|---|---|---|
| Random: fails 2 of 5, 8 of 10, worse under load | a timing race or concurrency bug | stop investigating the data; look at ordering, interleaving, contention |
| Consistent: input A always fails, input B always passes | a format or type gap | the inputs differ in a way the code does not handle; timing is a red herring |

The reason this matters: the instinct on any failure is to blame the data, because the data is what
you can see. But a failure that is random on identical input cannot be about the data, by
definition. The randomness is the proof. One real case: a reader dropped about 2 of 5 items of
similar content while a sibling tool got all of them. The first theory was a format gap in the
payload. Several hours went into that theory. The 2-of-5-on-similar-content pattern had already
ruled it out: a format gap fails the same input every time. The real cause was timing, a read that
landed mid-write.

## Why a synthetic repro lies

A hand-built reproduction of a timing bug is a hypothesis, not evidence, until it is proven faithful
to the real trigger's timing. The expensive failure mode:

```
build a plausible repro  ->  form a theory on it  ->  fix the theory  ->  real bug survives
                                                                          (the repro never had it)
```

A repro that reproduces the symptom by a different mechanism is worse than no repro, because it is
convincing and points you confidently at the wrong fix. The way out is to instrument the real
system: put a high-resolution trace on the actual path and capture it while the real trigger fires,
rather than rebuilding the trigger from guesswork. One investigation went through four progressively
wrong theories on an unfaithful harness before a single real trace settled it.

## The mechanism, generalized

The most common intermittent-loss mechanism is a read that observes a two-step write mid-flight. The
writer clears then fills (or bumps a version then publishes the payload), and a reader catches the
empty middle:

```
writer:   [ clear ]------------[ fill: payload ]------>   time
reader:          ^ reads here
                 sees empty, records "nothing", advances past this generation,
                 never re-reads -> the payload is lost forever
```

It wears different clothes per stack: a poller reading a row between a delete and its insert; a
queue consumer acking before the handler is durable; a read-after-write hitting a replica that has
not caught up and returning "not found"; a blocking call inside an event loop freezing it while new
events arrive unseen. The shared tell: a slower reader misses it less, because by read-time the
write has landed. If "poll slower" or "add a sleep" makes it go away, it is this, and the sleep is
hiding it.

The fix is not a sleep. It is a bounded re-read keyed to the same generation: a transient empty
arms a short, bounded retry of that same version (so a late write is caught), while a genuine empty
still terminates the retry quickly (so it does not spin). Test both directions.

## Don't trust the count

Before believing "only N of M arrived," rule out the channels that fake or hide a loss:

- Dedup swallows an identical repeat by design (vary each trial's payload so they are
  distinguishable).
- Read-after-write lag (a WAL, a replica, a cache, a CDN) hides a just-written record for seconds;
  counting too soon showed 1 of 8 when it was really 8 of 8.
- Buffering holds the value until a flush you measured before.
- Dropped or sampled logs mean a missing log line is not a missing event, especially under the
  exact burst you are studying.
- Retention or capping prunes the evidence between the event and your count.

On a hard intermittent bug, the measurement is confounded more often than not. Pick one signal,
prove it is reliable, and distrust counts taken too soon or through a layer that mutates them.

## Where this sits

This is a diagnosis skill in the verification set. Once you have the mechanism and a candidate fix,
`drive-the-fix` covers proving it on the real running artifact (a green suite does not catch a race),
and `adversarial-qa-loop` covers looping a sweep until it comes back clean.
