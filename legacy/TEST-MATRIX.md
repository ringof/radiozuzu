# Overlay Test Matrix

Cross-browser functional testing checklist for `palomar-overlay.user.js`.

## Browsers

| Abbr | Browser |
|------|---------|
| CR | Chrome (latest) |
| FF | Firefox (latest) |
| SF | Safari (latest) |
| ED | Edge (latest) |
| iOS | Safari on iOS |
| AND | Chrome on Android |

## Test Cases

Mark each cell: **P** (pass), **F** (fail — log in overlay-bugs.md), **—** (not tested)

| # | Feature | CR | FF | SF | ED | iOS | AND |
|---|---------|----|----|----|----|-----|-----|
| 1 | Frequency direct entry (type + Enter) | | | | | | |
| 2 | Click-to-tune on waterfall | | | | | | |
| 3 | Step up/down (< >) | | | | | | |
| 4 | Big step (<< >>) | | | | | | |
| 5 | Step size selector dropdown | | | | | | |
| 6 | Mode buttons (AM, SAM, LSB, USB) | | | | | | |
| 7 | Mode buttons (CWU, CWL, FM, IQ) | | | | | | |
| 8 | Mode sync from radio.js on load | | | | | | |
| 9 | Audio start/stop button | | | | | | |
| 10 | Auto-start audio on first gesture | | | | | | |
| 11 | Volume slider | | | | | | |
| 12 | Zoom in / out buttons | | | | | | |
| 13 | Zoom center button | | | | | | |
| 14 | Pan left / right buttons | | | | | | |
| 15 | Drag-to-pan on waterfall | | | | | | |
| 16 | Wheel zoom (vertical scroll / pinch) | | | | | | |
| 17 | Horizontal two-finger scroll pan | | | | | | |
| 18 | Waterfall max dB slider | | | | | | |
| 19 | Waterfall min dB slider | | | | | | |
| 20 | Spectrum max dB slider | | | | | | |
| 21 | Spectrum min dB slider | | | | | | |
| 22 | Spectrum/waterfall height ratio slider | | | | | | |
| 23 | Colormap cycle button | | | | | | |
| 24 | Autoscale button | | | | | | |
| 25 | Pause/resume spectrum button | | | | | | |
| 26 | S-meter bar + dBm readout | | | | | | |
| 27 | Filter passband shaded display | | | | | | |
| 28 | Passband drag handles (low/high) | | | | | | |
| 29 | DX marker labels on scale | | | | | | |
| 30 | Frequency tooltip on hover | | | | | | |
| 31 | Tune label on frequency scale | | | | | | |
| 32 | Radio status panel (expand/collapse) | | | | | | |
| 33 | Radio status values update | | | | | | |
| 34 | UTC clock | | | | | | |
| 35 | Panel collapse/expand toggle | | | | | | |
| 36 | Spectrum settings section collapse | | | | | | |
| 37 | No tune latency (instant response) | | | | | | |
| 38 | No mode snap-back | | | | | | |
| 39 | Smooth pan/zoom (no jank) | | | | | | |

## Notes

- Test on live site: https://palomar-sdr.com/radio.html
- For touch tests (iOS/Android): items 15-17 use touch equivalents
- Item 30 (tooltip) is mouse-only, skip on touch devices
- Log all failures in `overlay-bugs.md` with browser, OS, and steps to reproduce
