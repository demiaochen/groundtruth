# visual-self-verification

Part of [groundtruth](../../README.md).

An agent that screenshots its own UI work and reads the image back is both author and reviewer,
and it tends to trust the image. LLM vision is reliable for layout and structure but is fooled in
four specific, repeatable ways. This skill names each and gives the countermeasure. The
agent-facing instruction is in [SKILL.md](./SKILL.md); the failure modes are below.

## 1. Animations

A screenshot captures only the settled state. A reveal, slide, or fade can be broken
mid-transition (content over the wrong layer, a mask that fails to clip, a flash) and then snap to
a correct rest state, so every resting capture looks fine.

```
at rest (screenshot)        mid-transition (recording, t≈0.2s)
┌───────────────┐           ┌───────────────┐
│   panel       │  looks    │ panel ░░░░░    │  slides over its
│               │  correct  │   neighbor     │  neighbor, unclipped
└───────────────┘           └───────────────┘
```

Record the transition and read the moving frames (`screencapture -V`, Playwright `recordVideo`,
CDP screencast). Steady states are fine to read at rest; transitions are not.

## 2. Color and contrast

At display scale, rows of light text on dark fills average into the opposite of what is rendered,
so a downscaled screenshot can show light bands with dark text that are not there. This is a
common source of phantom dark-mode and contrast bugs.

Sample the pixels (`getComputedStyle`, a buffer sampler) or crop the region 1:1 and compare
against the intended value before trusting the appearance.

## 3. Interaction states

Proving a hover, focus, or pressed effect has two failure modes: moving the pointer may not fire
the framework's hover hook, so the state never engages; and a full-screen capture draws the cursor
over the target, so the state is hidden. A faint cue looks identical in both states and proves
nothing.

Trigger the state for real (a cursor warp plus `mouseMoved` events, or Playwright `hover()`),
capture the isolated window so the cursor does not occlude it (`screencapture -l <windowid>`), and
use an unmistakable cue (a 1.18x scale, not a 0.07-opacity fill).

## 4. Freshness

Everything else can be correct while the capture is of stale output: a dev server that hot-reloaded
JSX but failed to recompile CSS, an un-rebuilt binary, or a leftover screenshot from a previous
run.

Restart the server or use a fresh build, grep the served artifact for the change (not just the
source), rebuild before capturing native UI, and name capture files per run.

## The common thread

Each failure is the same: reading a subtle signal out of a lossy capture. Make the thing you are
verifying unambiguous, with a large cue, a 1:1 crop, a frame-by-frame recording, or a numeric
comparison. A passing test suite is not a substitute; visible bugs routinely survive it and are
caught only by reading the right capture.
