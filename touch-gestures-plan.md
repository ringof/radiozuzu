# Mobile Touch Gesture Support — Implementation Plan

**Commit:** `012bed3` — "Add mobile touch gesture support (tap, pan, pinch-zoom)"
**File modified:** `html/palomar-overlay.user.js`
**Branch:** `claude/convert-userscript-overlay-qp6JW`

## Rollback

```bash
git revert 012bed3
```

## Problem

The overlay's canvas interactions (pan, zoom, tap-to-tune) only worked with
mouse/wheel events. On mobile (iPhone Safari, Android Chrome), the browser
intercepts touch gestures for its own pinch-to-zoom and scroll, making the
overlay unusable. This was the #1 priority for the Dayton Hamvention demo.

## What Changed

### 1. CSS: `touch-action: none` (line 172)

```css
#p-sp-wrap,#p-wf-wrap,#p-sc-wrap{touch-action:none}
```

Tells the browser not to handle any touch gestures on the three canvas
wrappers. Without this, `preventDefault()` in touch handlers is unreliable
on iOS Safari.

### 2. State variables (lines 1265–1266)

```js
let pinch = null;          // { d0, mx0, sc0 } — two-finger pinch state
let _lastTouchEnd = 0;     // timestamp for ghost-click prevention
```

### 3. Ghost-click guard (line 1278)

Added to existing `mousedown` handler:

```js
if (Date.now() - _lastTouchEnd < 500) return;
```

After a touch gesture, browsers fire a synthetic `click` ~300ms later.
This guard prevents it from being misinterpreted as a tune click.

### 4. `touchstart` on canvases (lines 1339–1357)

Added inside the existing `[$('p-wf'),$('p-sp'),$('p-sc')].forEach()` loop.

- **1 finger:** Records start position and `centerKhz`, sets `drag` with
  `touch: true` flag (reuses existing drag variable).
- **2 fingers:** Cancels any single-finger drag, records baseline finger
  distance (`d0`) and midpoint X (`mx0`) for pinch/pan tracking.
- Uses `{ passive: false }` to allow `preventDefault()`.

### 5. `touchmove` on window (lines 1396–1446)

Window-level so the gesture continues even if finger moves off canvas.

- **Single-finger pan:** Same logic as mouse drag — computes dx from start,
  updates `centerKhz`, throttles `sendCenter()` at `PAN_SEND_MS` (50ms),
  refreshes visuals. Suppresses per-frame sync.
- **Two-finger pinch zoom:** Computes distance ratio vs baseline. Triggers
  `radio.zoomIn()` at ratio > 1.25, `radio.zoomOut()` at < 0.8, then resets
  baseline to prevent rapid continuous zooming.
- **Two-finger horizontal slide pan:** Computes midpoint X delta from
  baseline, pans `centerKhz` proportionally.
- Uses `{ passive: false }`.

### 6. `touchend` on window (lines 1448–1469)

- **Tap (finger didn't move):** Computes frequency from start X position,
  calls `rjsTune()` to tune.
- **Pan end (finger moved):** Calls `sendCenter()` to finalize, clears
  `_panSuppressSync`.
- **Pinch end (< 2 fingers remain):** Finalizes pan, clears pinch state.
- Sets `_lastTouchEnd = Date.now()` for ghost-click prevention.

### 7. Mouse handler guards (lines 1364, 1381)

Added `|| drag.touch` check to `mousemove` and `mouseup` handlers so
synthetic mouse events from touch don't double-fire the pan/tune logic.

## Key functions reused

| Function | Purpose |
|---|---|
| `rjsTune(khz)` | Tune to frequency (sends `F:` via WebSocket) |
| `sendCenter(khz)` | Send pan position to server (sends `Z:c:`) |
| `radio.zoomIn()` / `radio.zoomOut()` | Zoom via backend |
| `buildDX()`, `drawScale()`, `updatePB()` | Refresh visuals after pan |

## Design decisions

- **Reuses `drag` variable** with a `touch: true` flag rather than a
  separate touch-drag state, keeping the code DRY.
- **Pinch baseline resets** after each zoom step (not continuous) to give
  discrete zoom steps that match the backend's zoom behavior.
- **Threshold-based zoom** (1.25x / 0.8x) prevents accidental zoom from
  small finger movements during a horizontal pan.
- **5px dead zone** on two-finger pan midpoint prevents jitter.
- Follows the pattern already used by the split-handle drag on `#p-tune-wrap`
  (lines 1083–1126), which already handles both mouse and touch.

## Verification checklist

- [ ] iPhone Safari: single tap tunes to frequency
- [ ] iPhone Safari: single finger drag pans smoothly (no browser scroll)
- [ ] iPhone Safari: two-finger pinch zooms in/out
- [ ] iPhone Safari: two-finger horizontal slide pans
- [ ] Android Chrome: repeat all above
- [ ] Desktop mouse: all existing interactions still work (no regression)
- [ ] Desktop trackpad: wheel zoom and horizontal scroll pan still work
- [ ] Passband handle drag still works on both desktop and mobile
- [ ] Split-handle (spectrum/waterfall resize) drag still works
