# Overlay Improvements

Planned enhancements for `palomar-overlay.user.js`, prioritized.

## High Priority

### 1. Touch gesture support (mobile) — Dayton Hamvention demo
- Add `touch-action: none` CSS on canvas wrappers to prevent browser from stealing gestures
- Touch event handlers (`touchstart`/`touchmove`/`touchend`):
  - Single finger drag → pan
  - Single tap (short, no movement) → tune to frequency
  - Two-finger pinch → zoom in/out
  - Two-finger horizontal slide → pan left/right
- Test on iPhone Safari and Android Chrome
- If single overlay proves insufficient on mobile, consider a dedicated mobile layout

### ~~2. Help panel~~ ✅ Done (2026-04-25)
- ~~Click "Help" button → opens an overlay panel/frame with:~~
  - ~~Quick-start guide for new users~~
  - ~~All keyboard shortcuts~~
  - ~~Mouse/touch gestures (click-to-tune, drag-to-pan, wheel zoom, passband drag)~~
  - ~~Double-click to reset passband~~
  - ~~Panel collapse/expand~~
  - ~~Spectrum settings section~~
- ~~Should be dismissible (close button or click outside)~~
- ~~Styled consistently with the overlay panel~~
- Modal with 4 sections (mouse/touch, keyboard, fullscreen keyboard, panel controls)
- Dismisses via close button, backdrop/content click, or Escape
- User to verify each listed shortcut on live site (Step 4 of plan)

### 3. Wire up Record button
- `⏺ Rec` button exists but has no handler
- Connect to `player.startRecording()` / `player.stopRecording(freq, mode)`

### 4. Wire up Memories
- "Memories" button exists but has no handler
- Save/recall/delete frequency + mode + description
- Use localStorage (50 slots available in radio.js)

### 5. Band presets
- Quick-access buttons or dropdown for common bands
- Amateur, broadcast, utility, CB categories from radio.js `bandOptions`

## Medium Priority

### 6. Wire up DX labels toggle
- "DX labels" button exists but has no handler
- Toggle visibility of DX markers on the frequency scale

### 7. Keyboard shortcuts
- Spacebar: toggle audio
- Arrow keys: step frequency
- +/-: zoom in/out
- F: fullscreen
- A: autoscale
- M: cycle modes
- ?: open help

### 8. Ext dropdown menu
- "Ext" button exists but has no handler
- Dropdown for less-used features: CSV export, screenshot, settings reset

### 9. Filter edge numeric entry
- Complement drag handles with typed input for precise filter edges

## Low Priority

### 10. Panner (L/R audio balance)
### 11. FFT averaging controls (client + server)
### 12. Max/min hold trace toggles
### 13. Decay rate control
### 14. Auto mode-by-frequency toggle
### 15. Cursor mode
### 16. Band edge markers
### 17. WWV solar data display
