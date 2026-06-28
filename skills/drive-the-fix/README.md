# drive-the-fix

Part of [groundtruth](../../README.md).

A green build and a green test suite prove the code compiles and the tests pass. They do not prove
the bug is fixed. The defects a user actually reports live in UI, interaction, timing, and the OS or
framework glue that unit tests miss, and an agent that stops at "the suite is green" hands back fixes
that are still broken. This skill makes the agent earn the claim: reproduce the real trigger, change
the smallest thing, and prove the fix by running the real artifact and reading what it does. It also
scales the ceremony to the blast radius, so a one-line copy fix is not buried in process and a change
to storage or a hot path is not waved through. The agent-facing instruction is in
[SKILL.md](./SKILL.md); the walkthrough is below.

## A worked example

The report: a popup pastes into the wrong window. Two ways to handle it.

| Stop at the green suite | Drive the fix |
|---|---|
| read the code, spot a plausible cause | reproduce the real trigger first |
| change it, run the suite, see green | the suite was green before the change too |
| report done | launch the real artifact, fire the real trigger, read the evidence |
| the fix is unverified | the fix is proven, with a log line or recording attached |

The unit test calls the paste function directly with a window reference passed in. The real trigger
captures the front window at the moment a popup steals focus, a path the unit test never exercises.
So the suite stays green for a bug that is live, and a fix "verified by the suite" is verified
against the wrong code. Driving it means: copy a marker, fire the real shortcut into a real document,
read the document back, and read the log line on the capture path. The marker either landed in the
right window or it did not. That is the evidence the user wanted.

The same shape shows up off the desktop. A backend handler that passes its unit test but drops a row
under concurrent requests, a frontend transition that looks right in a screenshot but stutters when
recorded: in each case the direct-call test misses the trigger the user hits.

## The gate

For any visible or behavioral fix, do three things, in order, and attach the evidence:

1. Build and the full suite green. Necessary, not sufficient.
2. A regression test that asserts the invariant against the populated state, not the empty one.
3. A drive of the real artifact: launch the thing the user runs, exercise the actual trigger, read
   the runtime evidence on the path under test.

The recipe is stack-agnostic. On the web, a fresh server, a Playwright or CDP interaction, and the
DOM, network, and console. On a backend, a real request against a running instance and the response,
contract, and structured log. On native desktop, a rebuilt binary, a synthesized event or debug hook,
and the platform log plus a single-window capture. On a CLI, the built binary with real arguments and
the exit code, output, and on-disk side effects. On mobile, a simulator, a UI test runner, and the
device log. If a path genuinely cannot be driven in your environment, say so and name the one manual
check left, rather than claiming a fix you only compiled.

## Faithful reproduction

Synthetic reproductions are the most common way to waste an hour. A repro that calls a different code
path, skips the timing of the real event, or mocks away the race will reproduce a different bug, or
no bug, and point you at the wrong fix. Reproduce the trigger the user actually hits before you form
a theory. If you cannot reproduce it, that is a finding worth reporting on its own.

## Risk-tiered ceremony

Not every fix earns the same process. The tier is set by what the change touches.

| Tier | Touches | Gets |
| --- | --- | --- |
| 1 | hot path, storage, concurrency, public API, or a large or multi-file change | full ceremony: plan, drive, regression test, performance check, fresh-eyes pass, one scoped review |
| 2 | a contained change in one module | drive, regression test, self-review |
| 3 | copy, doc, token, or a one-liner with no behavioral branch | build and a targeted test |

The point is symmetric: do not over-process a trivial change, and do not under-process a dangerous
one. When unsure between two tiers, go up one.

## Stop and hand back

Autonomy ends where judgment is required. Hand the bug back, rather than landing a change, when the
root cause is still unclear after real investigation, when a gate fails and the fix is not obvious,
when the "bug" may be intended or test-pinned behavior, when the fix needs a scope or product
decision, or when it implies a release or a push to a shared surface. Confirm a behavior is a defect
before you change it; a deliberate default or a test that locks behavior on purpose is not a bug to
fix.

## The common thread

The suite tells you the code did not break in the ways you already test for. It cannot tell you the
bug is gone, because the bug is, by definition, something the tests did not catch. Close that gap by
running the real thing once, watching it do the right thing, and keeping the proof. Everything else
in the skill, the tiers, the faithful repro, the stop conditions, exists to make that single drive
trustworthy and proportionate.
