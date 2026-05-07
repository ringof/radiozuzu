# Overlay Bugs

Bugs found during cross-browser testing of the legacy palomar overlay
(`legacy/html/palomar-overlay.user.js`).

> **Status:** kept for historical reference during the transition.
> See `docs/MODERNIZATION.md` for how each open item is resolved by
> the rebuild.

## Open against the legacy overlay

### Occasional frequency stutter on arrow button clicks

**Symptom:** Clicking `< > << >>` buttons, the gold frequency display and
spectrum cursor occasionally flicker back to the old frequency for one
frame before correcting. ~1 in 10 clicks.

**Root cause:** `radio.js` overwrites `window.frequencyHz` with the
server's stale value every WS frame, before the server has processed the
overlay's `F:` command, racing the overlay's own `_rjsTarget` guard.

**Mitigated** by the overlay's `_rjsTarget` tracking + 50ms minimum
window (commits aa1ea84, 6b265f8) — eliminates most of the gold-number
stutter, but the underlying race remains because radio.js writes
`spectrum.frequency` directly.

**Resolved by Phase 4 of the rebuild.** The new app makes the FastAPI
server the single authority over tuning state, eliminating the
client-vs-radio.js race entirely. No client-side guard needed.

### Fullscreen support incomplete (Phase 2)

**Symptom:** Pressing `f` makes the screen go black because spectrum.js
fullscreens its own `#waterfall` canvas, not `#p-overlay`.

**Phase 1 (already merged in the legacy overlay):** intercept `f`,
fullscreen `#p-overlay` instead.

**Phase 2 (TODO, will be picked up in radiozuzu Phase 4):** the
"fullscreen-only" keyboard shortcuts (space, c, arrows, s/w, +/-, m, z,
i/o, a) are gated by `Spectrum.prototype.onKeypress` on
`this.fullscreen === true`, which stays false because we fullscreen
`#p-overlay`. In the rebuild we own the keydown handler and re-implement
the shortcuts directly — no need to spoof spectrum.fullscreen.

## History

A handful of overlay bugs were fixed in 2026-04 (passband drag handles,
~1 s tune/mode latency, AudioContext autostart, mode-button startup
sync, scale-bar accidental-tune). Commit hashes are in the legacy
overlay's git history; not reproduced here because the C+overlay
codepath is being retired.
