# Overlay Bugs

Bugs found during cross-browser testing of `palomar-overlay.user.js`.
Reference test cases from `overlay-test-matrix.md`.

## Open

### Occasional frequency stutter on arrow button clicks

**Symptom:** Clicking `< > << >>` buttons, the gold frequency display and
spectrum cursor occasionally flicker back to the old frequency for one frame
before correcting. Happens ~1 in 10 clicks. Waterfall clicks have the same
underlying issue but it's less frequent and harder to notice.

**Root cause:** `radio.js` overwrites `window.frequencyHz` with the server's
stale value every WS frame (line 716), before the server has processed the
overlay's `F:` command. This causes:
1. The overlay's loop to revert `tuneKhz` → gold number bounces
2. `spectrum.setFrequency(frequencyHz)` (radio.js line 769) → spectrum.js
   cursor bounces on its canvas → visible through the overlay's `drawImage` copy
3. `document.getElementById("freq").value` (radio.js line 791) → host page
   input bounces

**What's fixed (aa1ea84, 6b265f8):** The overlay loop now tracks a
`_rjsTarget` after `rjsTune()` and suppresses frequency sync until the server
confirms (within 0.5 kHz) or 300ms elapses. A 50ms minimum window prevents
premature confirmation from reading our own write. This eliminated most of the
gold-number stutter.


**What's NOT fixed:**
- The spectrum.js cursor still bounces because radio.js writes
  `spectrum.frequency` directly — outside overlay control.
- Rare cases where the 50ms minimum isn't long enough (WS frame arrives
  after 50ms but before server confirms). Bumping to 80-100ms would catch
  more but adds imperceptible latency.
- The only complete fix would be to prevent radio.js from overwriting
  `frequencyHz` / `spectrum.frequency` while a tune is in flight. This
  would require changes to radio.js itself (e.g., a `pendingTune` guard
  similar to the overlay's `_rjsTarget`).

### Fullscreen support incomplete

**Symptom:** Pressing `f` makes the screen go black. spectrum.js throws
`"Not in fullscreen mode"` on exit attempt.

**Root cause:** spectrum.js `toggleFullscreen()` calls `requestFullscreen()`
on its own `#waterfall` canvas. That canvas goes fullscreen but the overlay
(`#p-overlay`, a sibling, not a child) isn't visible — so the screen is black.

**Phase 1 (done):** Intercept `f` in the overlay, call
`requestFullscreen()` / `exitFullscreen()` on `#p-overlay` instead.
`requestFullscreen()` requires a direct user gesture (click/keypress) — it
cannot be called from console, timers, or promises. We confirmed that
fullscreening `#p-overlay` renders correctly.

**Phase 2 (TODO):** The help popup lists "fullscreen only" keyboard shortcuts
(space, c, arrows, s/w, +/-, m, z, i/o, a). These are handled by
`Spectrum.prototype.onKeypress` which gates on `this.fullscreen === true`.
Since we fullscreen `#p-overlay` (not spectrum's canvas), `spectrum.fullscreen`
stays false and those shortcuts won't fire. Options:
- Set `spectrum.fullscreen = true` when entering overlay fullscreen
- Reimplement the shortcuts in the overlay's own keydown handler
- Some combination — audit which shortcuts already work and fill gaps

## Fixed

| Bug | Test # | Browser | Fix | Date |
|-----|--------|---------|-----|------|
| Passband drag handles non-functional (z-index, disabled button, TDZ crash, no per-frame update) | 28 | All | d744991, 459e505, bd07558, 97f8cb6 | 2026-04-25 |
| ~1s tune/mode latency (sync-check revert) | 37 | All | a316210 — direct WS sends + frequencyHz sync | 2026-04-25 |
| AudioContext suspended on autostart | 10 | All | 37c42bc — defer to first user gesture | 2026-04-25 |
| Mode button shows wrong mode at startup | 8 | All | e0426e5 — sync mode in per-frame loop | 2026-04-25 |
| Frequency scale bar click-to-tune causes accidental retunes during passband adjust | — | All | d226ca1 — block mousedown/touchstart on p-sc in capture phase | 2026-04-25 |
