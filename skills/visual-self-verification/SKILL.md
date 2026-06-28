---
name: visual-self-verification
description: >
  Use when an AI agent verifies its OWN UI/frontend work by taking and reading screenshots,
  recordings, or renders, so it isn't fooled by its own vision. Trigger whenever you are about
  to claim a visual or interactive change is correct from an image: an animation or transition,
  a hover/focus/pressed state, color/contrast/theming, or "it looks fixed now." Covers the four
  repeatable ways an LLM misreads its own captures (resting frames hide broken animations,
  downscaled renders invert perceived contrast, interaction states are never triggered or get
  occluded by the cursor, and stale builds or caches show old output) with concrete,
  stack-agnostic countermeasures: record don't rest-shot; pixel-sample don't eyeball tone;
  trigger-then-isolate the state; prove the artifact is current. Applies to web, native
  desktop, and mobile.
license: MIT
version: 1.0.0
author: Demiao Chen
tags: [verification, ui, frontend, screenshots, qa, testing]
---

# Visual self-verification

When you screenshot your own UI work and read the image back, you are both author and reviewer,
and you tend to trust the image. Your vision of a capture is reliable for layout and structure,
but it is fooled in four specific, repeatable ways: a "fixed" animation that is still broken, a
contrast bug that does not exist, a hover effect that never fired, a CSS change verified against a
server still serving the old file.

Before claiming a visual or interactive change is correct, decide which kind of question you are
answering. If your eyes cannot be trusted on it, use an instrument: a recording, a pixel sample, a
triggered and isolated capture, or a freshness check. A passing test suite is not a substitute;
visible and behavioral bugs routinely survive a green suite and are caught only by reading a
capture.

## The four traps

| About to claim | The trap | Do instead |
|---|---|---|
| the animation or transition looks right | a resting frame looks perfect while the moving frames are broken | record it; read the moving frames |
| the colors, contrast, or theme are right | a downscaled render averages adjacent tones and can invert perceived contrast | pixel-sample or crop 1:1; compare to known values |
| the hover, focus, or pressed state works | the state never fired, the cursor occludes the target, or the cue is too faint | trigger the state, isolate the capture, use an unmistakable cue |
| it looks fixed now | you are reading a stale build, cached server, or leftover screenshot | confirm the artifact is current first |

---

## 1. Animations: record, do not rest-shot

A screenshot tests only the end state. A reveal, slide, or fade can be broken mid-transition
(content over the wrong layer, a mask that fails to clip, a flash) and then snap to a correct rest
state, so the resting frame never exercises the broken path. One case: a reveal-from-behind-an-edge
animation did not clip during the motion, so a panel slid on top of its neighbor for ~200ms, then
landed correctly masked. Every resting capture looked fine; only a recording showed it.

- Record the transition, started in the same command that triggers it so you catch the first
  frames.
  - Web: Playwright `recordVideo` or trace, Puppeteer screencast, or CDP `Page.startScreencast`.
  - Native (macOS): `screencapture -V <sec> out.mov`, launched right after the app
    (`app & ; screencapture -V ...`).
  - Mobile: simulator or emulator screen recording.
- Extract frames, crop to the region, tile them, and read the sequence, not just the last frame:
  `ffmpeg -i out.mov -vf fps=30 f_%03d.png`, crop each, then `ffmpeg ... tile=NxM`.
- Know when the interesting frames are (an animation that fires ~300ms after a view appears is
  around 0.4 to 0.9s in) so you do not tile past it.
- A steady state (a settled hover, a static layout) is fine to read at rest. Only transitions
  require a recording.

## 2. Color, contrast, and theming: sample pixels, do not eyeball a thumbnail

LLM vision of a downscaled UI is reliable for layout but not for which of two adjacent tones is
background and which is text. At display scale, rows of light text on dark fills average into the
opposite of what is rendered, a contrast inversion. One case: this produced a confidently
diagnosed but nonexistent dark-mode tear bug; every instrument (view dump, pixel sample, 1:1 crop)
showed correct dark rendering, while the scaled full-window shot showed light rows.

- Sample exact pixels and compare to the intended value (a design-token RGB, a computed style).
  - Web: `getComputedStyle(el).color` or `.backgroundColor`, or read pixels off a canvas or CDP
    capture.
  - Native: a small image sampler over the captured buffer (mind the buffer's row order; a
    CoreGraphics buffer's row 0 is the top scanline, no y-flip).
- Or crop the region 1:1 with no downscale and read that (macOS: `sips --cropOffset Y X -c H W
  copy.png`, on a copy, since `sips` edits in place).
- Treat any color bug seen only in a scaled full-window shot as unconfirmed until a pixel sample
  agrees.

## 3. Interaction states: trigger correctly, then isolate

To prove a hover, focus, or pressed effect fires at runtime, you must both put the UI into that
state and capture it without the cursor covering the target. Moving the pointer may not trigger
the framework's hover hook; a full-screen capture draws the cursor over a small target; and a
faint cue looks identical in both states.

- Trigger the state for real.
  - Web: Playwright `locator.hover()`, `:focus`, or dispatched events; CDP
    `Input.dispatchMouseEvent` with `mouseMoved`.
  - Native (macOS): warp the cursor and post a few `mouseMoved` events; a warp alone often will
    not trigger `.onHover`. Read the cursor location back to confirm it landed.
- Isolate the capture so the state renders but the cursor does not occlude it: capture the single
  window or element (macOS: `screencapture -l <windowid>` renders the state without drawing the
  cursor; a full-screen `-C` draws the cursor over small targets and adds z-order and
  multi-display issues).
- Capture the right surface: a debug launch may have several windows (a blank helper plus the real
  one at a different layer); enumerate windows (for example `CGWindowListCopyWindowInfo`) and find
  the correct id.
- Use an unmistakable cue. A 1.18x scale is clearly measurable; a 0.07-opacity fill looks the same
  in both states. When you control the test, make the proof obvious.

## 4. Freshness: confirm the artifact is current

You can do everything else right and still read old output. A long-running dev server can serve a
stale CSS bundle (JSX hot-reloads so the HTML looks updated, but a CSS edit failed to recompile);
a cached build, an un-rebuilt binary, or a leftover screenshot file all show the past. One case:
"the page is a mess" while the crops looked fine, because the capture was of a server that had been
up since an earlier task, serving stale CSS.

- Do not trust a server or build that has been up since an earlier task. Restart it (kill, clear
  the build cache, relaunch) or verify against a fresh static build on a clean port.
- Grep the served artifact for your change, not just the source: find the served CSS URL and grep
  it for the new selector. HTML-only checks miss stale CSS.
- Rebuild before capturing native UI; confirm the binary's mtime or commit is the one you just
  changed.
- Name capture files per run (or delete the prior one first) so you never read a previous run's
  PNG.
- Do not run a production build while a dev server is live on the same checkout if they share a
  build directory; it can corrupt the running server.

---

## The principle behind all four

Each failure is the same: reading a subtle signal out of a lossy capture. Make the thing you are
verifying impossible to misread, with a large cue, a 1:1 crop, a frame-by-frame recording, or a
numeric comparison. If a proof depends on distinguishing two similar grays in a thumbnail, it is
not a proof.

## What a capture is reliable for

Do not over-apply this. Reading a capture is trustworthy for layout and alignment, structure and
hierarchy, presence or absence of an element, gross spacing, obvious overflow or clipping, and
text content. Use a screenshot freely for those, and reach for an instrument only for the four
traps above.

## Before claiming a visual or interactive fix is done

- Transition? You have a recording of the moving frames, not a rest shot.
- Color, contrast, or theme? A pixel sample or 1:1 crop agrees, not just a scaled view.
- Interaction state? You triggered it for real and captured it isolated, with an unmistakable cue.
- Is the artifact current? Fresh build or server, the served artifact greps for your change, the
  capture file is this run's.
- Could a green test suite have missed this? For a visible bug, almost always. The capture is the
  gate, not the passing check.
