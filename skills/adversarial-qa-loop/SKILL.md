---
name: adversarial-qa-loop
description: >
  Use when you want an AI agent to find the bugs that matter and drive a project to zero
  defects with proof, not a list of plausible problems read out of the source. Trigger on:
  run a QA sweep, find all the bugs, run the QA loop, hunt regressions, do a full QA pass,
  drive the app bug-free, adversarial QA. It fans out parallel finder lanes (static
  self-contradiction, contract-oracle, flow/journey, differential), routes every finding
  through a skeptical default-reject verifier with three verdicts (confirmed,
  needs-runtime-check, rejected), proves behavioral bugs by driving the running software
  through a per-stack adapter (web, backend, CLI, native desktop, mobile), fixes them one at
  a time with a red-then-green test, and re-sweeps round after round until a full pass is
  clean with no regressions. Stack-agnostic.
license: MIT
version: 1.0.0
author: Demiao Chen
tags: [qa, testing, verification, regression, debugging, agents]
---

# Adversarial QA loop

The job is not to list plausible bugs. It is to find the ones that matter, prove they are real
by running the software, fix them, prove the fix, and repeat until a full sweep is clean.
Reading code finds the bugs that contradict themselves: a missing clamp, a rule that matches
everything, a branch that cannot be reached. The bugs users actually hit live in the runtime: a
control wired but dead in one state, a request that reaches the wrong handler, a value that
diverges between two code paths. Those are invisible to reading. The gate is running it. A green
test suite is the floor, not the proof, because visible and behavioral bugs routinely survive a
green suite.

This is a loop, run in rounds. Scale it to the ask: a quick pass is one round with fewer lanes;
"drive it bug-free" is the full set of lanes, looped until two consecutive clean sweeps.

## The round at a glance

| Phase | What happens |
|---|---|
| Discover | map this project's subsystems, real user journeys, and contracts with two implementations; derive the lanes from that, do not hardcode them |
| Find | run the finder lanes in parallel, each a different lens on a different bug class |
| Verify | route every finding through a skeptical, default-reject reviewer: confirmed, needs-runtime-check, or rejected |
| Drive | prove each confirmed-behavioral and every needs-runtime-check finding by running the software through a per-stack adapter |
| Fix | one bug at a time, each with a red-then-green test, each landed before the next starts |
| Converge | re-sweep for regressions; require the top severity to drop each round; stop when a full sweep is clean |

## 0. Discover and baseline

Derive the lanes from the project in front of you, not from a fixed list.

- Subsystems. List the top-level modules or services and what each owns. You get one static
  lane per subsystem.
- Journeys. List the real end-to-end paths a user (or a calling system) takes across layers:
  the primary action, the create-edit-delete lifecycle of the main object, the capture or
  intake path. You get one flow lane per journey. The seams between subsystems are nobody's job
  under a per-module split; this lane owns them.
- Twin contracts. Find every contract that has two implementations: an old path and a new
  one, two storage backends behind one interface, two views over the same data, a server rule
  and its client mirror. You get one differential lane per pair.
- Write this down as a short per-project config the loop reads, so the next round reuses it.

Then set the floor. Build and run the full suite first; it must be green before you change
anything. Record the test count and the current commit. Regressions cluster in what just
changed, so note the recent commits and feed the changed files to the next round as focus.

Keep a short exclusions list the loop reads every round: bugs already fixed, decisions that are
locked or intended, items out of scope. Do not re-flag them. Update it as fixes land.

## 1. The four finder lanes

Run them in parallel: dispatch one subagent per lane, each returning a ranked list of findings.
If you have no parallel subagent mechanism, run each lane as a separate focused pass. Keep the
finder and the verifier as distinct roles so the verifier is genuinely independent.

| Lane | Lens | The class it catches |
|---|---|---|
| Static | read each subsystem in isolation for internal contradiction | a missing clamp, an empty rule that matches all, an off-by-one, an unreachable branch |
| Contract-oracle | trace every control, shortcut, and endpoint to its handler in every state | wired-but-dead: bound but no effect, or works in one state and not another |
| Flow/journey | own one end-to-end path across all layers | seam bugs the per-module split leaves to no one: stale captured context, focus not relinquished, a value not settled before the next step |
| Differential | diff two implementations of one contract | divergence: one side clamps, guards, or orders where the other does not |

The contract-oracle lane is the one that catches the class static sweeps miss. Enumerate every
control, shortcut, and endpoint, then trace each one to its handler in every state the system
can be in (each view, each role, each focus target, each open dialog). Flag any that cannot
reach its handler, reaches the wrong one, or is documented but unimplemented.

Tune the finders for recall. Ask for a ranked list that includes medium-confidence suspicions,
each tagged with its confidence. A behavioral suspicion you cannot fully prove by reading is
still worth reporting; the verifier routes it to a live run rather than dropping it. Do not tell
a finder that "returning empty is valuable": that phrase licenses under-production, and the
result is a list of only slam-dunks.

Each finding carries: title, location (file:line, route, or endpoint), severity (P1
data-loss/crash/security/core-flow-broken, P2 wrong behavior in a real path, P3 polish),
category/lane, description, concrete repro steps, the invariant it violates, and a confidence.

## 2. Adversarial verification, three verdicts

The finder's job is recall. The verifier's job is precision. The verifier defaults to reject and
only confirms what it independently re-derives at the cited location. It sorts each finding:

- confirmed: real, not already fixed, not intended, and provable from source alone.
- needs-runtime-check: plausibly real, not refuted, not intended or fixed, but only running
  the software can settle it (focus, timing, event routing, visual, concurrency). It must emit a
  drive recipe: the exact steps to reproduce on the running software.
- rejected: refuted, already fixed, intended or locked, or code-readable but unproven.

Precision comes from default-reject. Recall comes from the middle bucket. The line to hold: a
code-readable suspicion that you cannot prove is rejected, but a behavioral suspicion is not
rejected, it is routed to a drive. Silently dropping behavioral suspicions is exactly how the
wired-but-dead class escapes a code review.

Watch the rejection rate. If a whole round rejects nothing, the finders under-produced; they
reported only certainties. Push them harder next round: more lanes, a completeness-critic pass,
a lower confidence floor.

## 3. Drive: prove behavioral bugs on the running software

A confirmed-behavioral finding and every needs-runtime-check finding is proven by running the
built software and observing it, not by reading more code. A runtime question (does the keypress
reach the handler while a field is focused, does the request hit the right route) cannot be
settled by re-reading; only a run settles it.

The drive pattern is the same on every stack:

1. Stage a unique known input (a marker value).
2. Put the system into the exact state the bug needs (the specific view, role, or data shape).
3. Trigger the action through the surface a user uses (the real keypress, click, or request).
4. Read the real output back from the target, not from the source.
5. Assert the marker landed or the action fired. Add a log line on the handler path and confirm
   it was reached: this separates "the handler ran and did the wrong thing" from "the handler
   never ran", which is the wired-but-dead signature.
6. Clean up anything you mutated.

Pick the adapter for your stack. These are parallel options; none is the only path.

- Web app: drive the running app with Playwright or Puppeteer (`goto`, `locator.click()`,
  `keyboard.press()`), or use CDP `Input`/`Runtime` to dispatch events and read computed state
  and network responses. Assert on the live DOM or the response, not the source.
- HTTP backend or API: call the real endpoint with a crafted payload (an HTTP client or your
  integration harness); assert on status, body, and the persisted record, and cover the boundary
  and the error path, not only the happy one.
- CLI tool: run the built binary under a PTY (`expect`, `pexpect`, `script`) so it behaves as
  in a real terminal; feed stdin and keystrokes; assert on stdout, exit code, and side-effect
  files.
- Native desktop: launch the built app and drive it through the OS automation layer (the
  accessibility API, scripting, or synthetic events) or a UI test target; capture a screenshot or
  a log line and read it back. (On macOS, for example: scripting to drive a target app, a window
  capture for the view, the unified log for the handler line.)
- Mobile: install on a simulator or emulator and drive with the platform UI automation
  (XCUITest, Espresso, or an accessibility driver); assert on screen content or a log.

Some behavior is not drivable headlessly: timing races, hardware you do not have (a second
display), wall-clock cliffs (a prune that fires after N days), and any input that needs a
permission or a foreground window you cannot script. For those, unit-test the logic and hand the
user one exact manual repro. Never call something verified because it compiled. A
needs-runtime-check item a drive refutes is dropped; one a drive confirms graduates to a real
bug.

## 4. Fix, one at a time

File each survivor in your tracker (an issue list, a checklist, whatever you use) with location,
severity, repro, the invariant, and the drive evidence, deduped against the exclusions list.
Then fix sequentially:

- Isolate the work on a branch or worktree off the current main HEAD, so fixes compound.
- Root-cause it, then make a surgical fix.
- Add a behavioral test that fails before the fix and passes after (red then green), exercising
  the populated state, not the empty one.
- Build, run the full suite green, then land the fix before starting the next. Sequential
  landing keeps every fix gated and avoids build and merge races.
- A fix that breaks a pinned or locked test means that behavior is intentional. Revert the fix;
  do not override the test. The locked test is the spec.

## 5. Converge

Fixes introduce regressions, so re-sweep after every fix wave. Two rules end the loop:

- The maximum severity must strictly drop each round. A round that still surfaces a P1 after a
  wave that targeted P1s means a fix missed or regressed; do not advance.
- Stop when a full sweep returns zero confirmed and zero needs-runtime-check, twice in a row (a
  single empty round can be luck), or when every survivor is below an agreed noise floor.

The differential and flow lanes are the standing regression net: they re-diff the seams a fix
just touched. Point the next round's focus at the files you just changed.

## Before you call the campaign done

- Did you derive the lanes from this project's real subsystems, journeys, and twin contracts,
  not a generic list?
- Did the verifier reject some findings? A 0% reject rate means the finders under-produced.
- Is every behavioral claim backed by a run of the real software, not a code-read?
- Did every fix add a test that was red before and green after, on the populated state?
- Did any fix break a pinned test? If so you reverted it, because that behavior is locked.
- Did the top severity drop this round, and did two consecutive sweeps come back clean?
