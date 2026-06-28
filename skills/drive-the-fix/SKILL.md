---
name: drive-the-fix
description: >
  Drive a bug from report to a verified, landed fix end to end. Reproduce the real trigger
  faithfully, root-cause it, isolate the work on a branch or worktree, make a surgical change,
  then prove it by running the real artifact (launch the thing the user runs, exercise the actual
  trigger, read the runtime evidence) and attach proof, not just a green test suite. Scales review
  to blast radius: full ceremony (performance check, scoped review, fresh-eyes pass) for hot-path,
  persistence, concurrency, public-API, or large multi-file changes, light for contained or trivial
  ones. Stack-agnostic, with concrete recipes for web, backend, native desktop, CLI, and mobile.
  Use when asked to fix this bug end to end, ship a verified fix, prove the fix works, fix and
  verify, handle this bug, or root-cause and fix. Stops and hands back when the root cause is
  ambiguous, the behavior may be intended or test-pinned, or the fix needs a scope or product call.
license: MIT
version: 1.0.0
author: Demiao Chen
tags: [debugging, verification, qa, testing, workflow, bugfix]
---

# Drive the fix

The user handed you a bug and wants a fix they do not have to re-check. A green build and a green
test suite are the floor, not proof: visible and behavioral defects routinely survive a green suite
and are caught only by running the real thing. The gate is running it. Scale the ceremony to blast
radius so you neither over-process a one-line fix nor under-process a dangerous one.

The loop: isolate, reproduce the real trigger faithfully, root-cause, classify the tier, change it,
prove it by driving the real artifact, review to tier, land it, record the non-obvious.

## 1. Isolate first

Work on a branch or a worktree so the fix stays contained and the main line stays buildable. Base
it on current mainline, not a stale local checkout: if your isolation tool branches from a remote
that lags your local trunk, merge or rebase the latest trunk in before you read or edit, or you
build on stale code and silently drop recent fixes. If the change touches generated or cached build
artifacts (a compiled store, a schema, a derived index, a bundler cache), clear that cache before
the first rebuild so a stale shadow does not corrupt the result.

## 2. Reproduce the real trigger, faithfully

Reproduce before you theorize. Trust only a reproduction that exercises the actual trigger the user
hits. Synthetic repros are routinely unfaithful: a unit test that calls a different code path, a
hand-built input that skips the timing of the real event, a mock that hides the race. An unfaithful
repro sends you chasing the wrong theory and fixing code that was never the cause. If you cannot
reproduce it, that is itself a finding: say so before changing anything.

Then root-cause. Trace the actual path from trigger to symptom. For "what calls what", "how does X
reach Y", and "what would this break", prefer structural code intelligence (call graph, callers,
impact) over text search.

## 3. Classify the tier

The tier sets the ceremony for everything after it. When unsure between two tiers, go up one.

| Tier | The change touches | Ceremony |
|---|---|---|
| 1 (risky) | a hot path, persistence or storage, concurrency or shared mutable state, a public API or contract, or it is large or spans many files | full: plan, behavioral drive, regression test, a performance check if it touches a hot path or large data, a fresh-eyes pass, and one scoped review |
| 2 (normal) | a contained change in one module or view | plan if multi-file, implement, behavioral drive, regression test, self-review |
| 3 (trivial) | a copy, doc, token, or one-line change with no behavioral branch | implement, then build and targeted test |

## 4. Plan (tier 1 always, tier 2 if multi-file, skip tier 3)

Four lines: root cause, fix approach, the invariant the fix must not break, how you will prove it.
Do not relitigate decisions that are settled or out of scope for this bug.

## 5. Implement

Surgical. Match the surrounding idiom. Push non-trivial logic out of UI, framework, or OS glue into
a plain function you can call directly: the glue is where the hard bugs live and the function is
where you can pin them with a test. Preserve debug hooks, logging call sites, and any platform glue
you do not fully understand.

## 6. Prove it: the gate

This is the step that makes the fix one the user does not have to re-check. Do all three.

1. Build and the full suite green. Necessary, not sufficient.
2. Add or adjust a regression test that asserts the invariant, not a single output, against the
   populated or real state, not the empty one. A test bound to a value with no behavioral assertion
   is not a test.
3. Drive the real artifact: (a) launch the actual thing the user will run, (b) exercise the real
   trigger, (c) observe the behavior and read the runtime evidence you added on the path under test.
   Attach it: a screenshot, a recording, a response body, a captured log line.

If a second instance of the same class of bug appears (or the ask is "do the same fix in N other
places"), stop point-fixing: fix the shared layer once and add a test that fails on regression.
Enumerate every call site from the call graph, not from guesswork.

Per-stack recipes for step 3, each in three parts (launch, trigger, read):

- Web and frontend: launch a fresh build or restart the dev server (do not trust one that has been
  up since an earlier task and may serve stale assets); drive the real interaction with a browser
  automator (Playwright `locator.click()` / `hover()`, or CDP `Input.dispatchMouseEvent`); read the
  DOM, the network response, and the browser or server log. For visual or interaction claims, apply
  the visual self-verification countermeasures: record transitions, pixel-sample color, trigger and
  isolate hover and focus states, and confirm the served artifact is current.
- Backend and API: send the real request against a running instance (an HTTP client such as `curl`
  or a request runner, not just a handler unit test); assert the status, body, and contract; read
  the structured request log and check the side effects (the row was written, the message reached
  the queue, the cache was invalidated).
- Native desktop: rebuild and confirm the binary's mtime or commit is the one you changed; launch
  it and exercise the path with a real input, a synthesized event, or a debug hook that runs the
  real code path; read the platform log (for example macOS `log show ... --info` over a logging
  subsystem you added) and capture a single window (macOS `screencapture -l <windowid>`, or
  `screencapture -V` to record a transition).
- CLI: run the built binary with the real arguments against a real working tree; assert exit code,
  stdout, and stderr; check the side effects on disk.
- Mobile: install on a simulator or emulator; drive with the UI test runner (XCUITest, Espresso) or
  scripted taps; capture a screenshot or screen recording and read the device log.

Be honest about what you cannot drive. If a path is unverifiable in your environment (a permission
you cannot grant headless, an interaction the harness cannot post, hardware you do not have), say
so plainly and name the one manual check left. Never report "verified" for something you only
compiled.

## 7. Review (tier-scoped)

- Tier 1: a fresh-eyes self-review, then one review (a person or an agent) given the specific
  invariant to confirm, not an open-ended "review this." A scoped check finds real defects, an open
  one produces noise. Run the performance check if the change touches a hot path or a large-data
  operation: the fix must not regress throughput or latency, and expensive work must stay off the
  hot path.
- Tier 2: self-review, plus the interaction or contract self-audit if one applies.
- Tier 3: skip.

## 8. Land it

Decide the landing target up front and follow the team's convention, do not invent one. The common
default is to push the branch and open a pull request for review. Some workflows merge to a local
trunk without pushing, or land behind a feature flag. Whatever the target: keep the change isolated
until every gate for the tier passes, then land it as one logical change with a clear message. A
change to a separate surface (a release, a published artifact, a changelog, a deploy) is its own
explicit step, never a side effect of the fix.

## 9. Record the non-obvious

Write down what the diff does not show: the real root cause and the trap that nearly fooled you (the
unfaithful repro, the misleading symptom, the call path that was not obvious). Put it where your
team keeps that knowledge (the commit body, the pull request, an issue, a notes file, a wiki).
Update the existing note for that area instead of duplicating. This is what saves the next person,
including you.

## Stop and hand back (do not land) when

- The root cause is still unclear after real investigation: ask one sharp question.
- A gate fails and the fix is not obvious: report the actual output, do not land.
- The "bug" may be intended or test-pinned behavior: a deliberate default, a documented choice, a
  behavior a test locks on purpose. Confirm it is a bug before you change it. Do not "fix" intended
  behavior.
- The fix needs a scope or product call, or reopens a settled decision: ask.
- It implies a release or a push to a shared surface: that is a separate, explicit action.

## Before you report it done

- Reproduced the real trigger faithfully, not a synthetic stand-in.
- Tier picked, ceremony matched: no heavy process on a one-liner, no skipped drive on a hot-path
  change.
- Build and suite green, plus a regression test on the invariant and the populated state.
- Drove the real artifact: launched it, fired the real trigger, read the runtime evidence, attached
  it. "Tests pass" alone is never enough for something a user can see or feel.
- Landed to the agreed target as one clean change, or stopped and handed back with the reason.
- The non-obvious root cause and the trap are written down.
