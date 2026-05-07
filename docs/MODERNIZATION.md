# Radio Zuzu Modernization Plan

Status: in progress (Phase 0 in flight on
`claude/plan-radiozuzu-modernization-IynQ4`)

## Goal

Transition Radio Zuzu from a mix of a (broken) ka9q-web C server plus a
JavaScript overlay into a self-contained project that uses the latest
upstream code from:

- [ka9q-radio](https://github.com/ka9q/ka9q-radio) — base radio service
  (`radiod` and friends)
- [ka9q-python](https://github.com/mijahauan/ka9q-python) — Python API for
  controlling and streaming from `radiod` over RTP / TLV

Radio Zuzu becomes a small Python web app that bridges the browser to
`radiod` via `ka9q-python`, plus a first-class JS frontend (the
ex-overlay, promoted).

## Decisions (locked in)

| Topic | Decision |
|---|---|
| Server stack | **FastAPI + uvicorn** (single Python process, async WS, native fit for `ka9q-python`) |
| Cutover style | **Fork**: archive the C/JS legacy stack into `legacy/`, build the new project at the repo root |
| Frontend | **Promote `palomar-overlay.user.js` to first-class UI**; drop `radio.html` / `radio.js` / `overlay.js` |
| Audio | **Opus over WS**, transcoded server-side from `radiod` PCM (or — preferred — forwarded as-is when `radiod` is configured for Opus output on the channel) |
| Multi-radiod | **Yes, supported** — the server holds a `RadiodControl` registry keyed by host |
| Frontend tooling | **Plain ES modules**, no bundler; revisit Vite/TypeScript only if the bug surface or component count justifies it |

## Target repo layout (post-transition)

```
radiozuzu/
  server/                    # FastAPI app (replaces ka9q-web.c + admin/admin.py)
    radiozuzu/
      main.py                # app, static mount, /ws, /admin, /healthz
      radio.py               # RadiodControl registry, one per radiod host
      audio.py               # ManagedStream → Opus → WS binary
      spectrum.py            # request_powers + poll → WS JSON
      admin.py               # connection tracking, GeoIP, login
      models.py              # pydantic control-message schemas
      config.py              # TOML/env settings
    pyproject.toml
    tests/
  web/                       # plain-JS frontend (palomar promoted)
    index.html
    favicon.ico
    src/
      app.js                 # was palomar-overlay.user.js
      spectrum.js, pcm-player.js, opus-decoder.min.js,
      colormap.js, smeter.js
      style.css
  config/
    radiod@rx888-web.conf    # encoding = opus on the channel
    radiozuzu.toml
  systemd/radiozuzu.service  # one unit replaces ka9q-web + ka9q-admin
  nginx/radiozuzu.conf       # no more sub_filter injection
  docs/
    MODERNIZATION.md         # this file
    INSTALL.md, DEVELOPMENT.md, PROTOCOL.md
    overlay-improvements.md, touch-gestures-plan.md  (carried over)
  legacy/                    # archived: ka9q-web.c, html/, admin/, services
  ka9q-radio/                # submodule, tracking upstream main
  README.md  CHANGELOG.md
```

## Phases

### Phase 0 — Fork layout, no behavior change

- `git mv` legacy assets (`ka9q-web.c`, `Makefile`, `html/`, `admin/`,
  `ka9q-web*.service`, `update-w1euj.sh`, `radiod.commit`, `latest.png`)
  into `legacy/`.
- Move docs (`overlay*.md`, `touch-gestures-plan.md`,
  `local-dev-test.md`, `issue.md`) into `docs/`.
- Fix `legacy/Makefile` paths so the legacy build still works from
  `legacy/`.
- Update `.gitmodules` to track ka9q-radio `main`.
- Rewrite root `README.md` for transition state.

**Acceptance:** `git log --follow` preserves history on moved files;
legacy build still works via `cd legacy && make ka9q-web-dev`.

### Phase 1 — FastAPI skeleton

- `server/pyproject.toml`: `fastapi`, `uvicorn[standard]`, `ka9q-python`,
  `pydantic`, `jinja2`, Python ≥3.11.
- `main.py`: app, `/healthz`, static mount of `web/`, lifespan opens a
  `RadiodControl` **registry** (multi-radiod), `/ws` placeholder.
- `models.py`: pydantic schemas for `Tune`, `SetMode`, `SetCenter`,
  `SetZoom`, `SetFilter`, `SetSpectrumAvg`, `SetSpectrumOverlap`,
  `SetWindow`, `SetPoll`. JSON in, JSON or binary out.

**Acceptance:** server starts; `discover_channels()` returns the
configured radiods' active SSRCs.

### Phase 2 — Spectrum stream

- `spectrum.py` task per WS session: `RadiodControl.request_powers(...)`
  + `RadiodControl.poll(...)` on a 100 ms tick, settable via `SetPoll`.
- WS frame: `{type:"spectrum", center_hz, bin_width_hz, base_db, step_db,
  bins:[...]}`.
- Wire palomar's spectrum/waterfall path to consume the new frame.

**Acceptance:** browser shows a live spectrum sourced through
`ka9q-python`.

### Phase 3 — Audio (Opus)

- Update `config/radiod@rx888-web.conf` to set `encoding = opus` on the
  audio channel; document min/max samprate per mode.
- `audio.py` per-session task: `ManagedStream(ssrc=…)` configured for
  Opus → strip RTP header → `await ws.send_bytes(opus_frame)`.
- Browser side: keep `pcm-player.js` + `opus-decoder.min.js`; harmonize
  framing prefix with the new server (server and player change together,
  see PROTOCOL.md).
- Optional 3b: add an `opuslib`/PyAV encode path for PCM-output channels,
  off by default.

**Acceptance:** audio plays at parity latency, bandwidth ≈ Opus rate
(~32 kbps SSB).

### Phase 4 — Full control + admin fold-in

- Implement the rest of the WS control surface against `RadiodControl`.
- Rewrite palomar's WS calls from text protocol (`F:`, `M:`, `Z:c:`, …)
  to typed JSON. Closes `overlay-bugs.md` "frequency stutter" — one
  Python authority replaces the radio.js/server-state race.
- Fold `admin/admin.py` into FastAPI as `/admin` routes; tracking comes
  from in-process WS session state. Keep SQLite for history only.
- Closes `issue.md` (real client IP) — read `CF-Connecting-IP` /
  `X-Forwarded-For` directly from `request.headers` with a trust list.

**Acceptance:** every overlay button works; admin dashboard works;
client IPs correct behind Cloudflare.

### Phase 5 — Packaging, install, docs

- Single `radiozuzu.service` replaces both old units.
- `nginx/radiozuzu.conf`: drop `sub_filter` injection block.
- `pipx install .` (or `pip install .` + install script) replaces
  `make install`.
- README rewritten end-to-end. Tag `v0.legacy` immediately before
  removing `legacy/` from main.

### Phase 6 — Stuck-on-server features now unstuck

- Record button → `RTPRecorder`.
- Memories → server-side persistence.
- Band presets, DX labels toggle, Ext menu, filter edge numeric entry.
- Touch gestures land cleanly in the rewritten app.

## ka9q-python tasks (gating Phase 4)

These are filed/to-be-filed against `mijahauan/ka9q-python`:

### Control commands

| # | Method | TLVs | Target SSRC |
|---|---|---|---|
| 1 | `set_filter_edges(ssrc, low_hz, high_hz)` | `LOW_EDGE` (float), `HIGH_EDGE` (float) | audio ssrc |
| 2 | `set_spectrum_average(spec_ssrc, n)` | `SPECTRUM_AVG` (int32) | audio ssrc + 1 |
| 3 | `set_spectrum_overlap(spec_ssrc, frac)` | `SPECTRUM_OVERLAP` (float) | audio ssrc + 1 |
| 4 | `set_window(spec_ssrc, type, shape=None)` | `WINDOW_TYPE` (int), optional `SPECTRUM_SHAPE` (float) | audio ssrc + 1 |
| 5 | `request_powers(spec_ssrc, center_hz, bins, bin_bw_hz)` | `DEMOD_TYPE=SPECT2_DEMOD`, `RADIO_FREQUENCY`, `BIN_COUNT`, `RESOLUTION_BW` | audio ssrc + 1 |
| 6 | `stop_spectrum(spec_ssrc)` | `DEMOD_TYPE=SPECT2_DEMOD`, `RADIO_FREQUENCY=0`; sent **3×** | audio ssrc + 1 |
| 7 | `discover_ssrcs()` via `poll(ssrc=0)` | first-byte=1 + `OUTPUT_SSRC=0` + `COMMAND_TAG` | n/a |

### `WindowType` enum

`KAISER=0, RECT=1, BLACKMAN=2, EXACT_BLACKMAN=3, GAUSSIAN=4, HANN=5,
HAMMING=6, BLACKMAN_HARRIS=7, HP5FT=8` (mirror `ka9q-web.c` L1322–1331).

### Status fields to expose

**`ChannelStatus`:** RADIO_FREQUENCY, FIRST_LO_FREQUENCY,
SECOND_LO_FREQUENCY, LOW_EDGE, HIGH_EDGE, PRESET, IF_POWER,
BASEBAND_POWER, NOISE_DENSITY, OUTPUT_LEVEL, BIN_COUNT, RESOLUTION_BW,
BIN_DATA, BIN_BYTE_DATA, GPS_TIME, SPECTRUM_AVG, SPECTRUM_OVERLAP,
WINDOW_TYPE, SPECTRUM_SHAPE, spectrum.base, spectrum.step,
spectrum.noise_bw, clocktime.

**`FrontendStatus`:** samprate (double), rf_agc, rf_atten, rf_gain,
rf_level_cal, samples (uint64), overranges (uint64),
samp_since_over (uint64).

### Streaming

- Codec-agnostic Opus payload passthrough on `ManagedStream` (or a
  `RawStream` flavor with `decode=False`).
- GPS_TIME / RTP_TIMESNAP wallclock retained.
- `MultiStream` shared-socket fan-out (verify).

### Test fixtures

- Loopback fake-radiod fixture for CI.
- Recorded TLV byte-stream golden files for control commands 1–6.

## Risks (open)

| ID | Risk | Status |
|---|---|---|
| R1 | TLV coverage in ka9q-python (the list above). | Open — gate Phase 4 on items 1–7 + status fields. |
| R2 | Per-channel Opus output on rx888 / radiod for SSB/AM/FM at the rates we use; IQ and the wide-bin spectrum paths may need PCM. | Open — verify on real hardware before committing Phase 3 design. |
| R3 | WS audio framing. Server framing and `pcm-player.js` change together; PROTOCOL.md will define `[u32 ts_delta][u16 seq][u16 opus_len][bytes]`. | Accepted; resolved by joint change in Phase 3. |
| R4 | Multi-radiod scope. | **Resolved**: supported from day one. `radio.py` holds a `RadiodControl` registry keyed by status host. |
| R5 | Frontend tooling (Vite vs plain JS). | **Resolved**: plain JS for now. |

## What Phase 0 changes (this PR)

1. Move legacy C/JS/admin into `legacy/`.
2. Move design and ops docs into `docs/`.
3. Patch `legacy/Makefile` so it builds from its new location.
4. Update `.gitmodules` to track ka9q-radio upstream `main`.
5. Rewrite root `README.md` to describe the transition state and point
   contributors at the right tree.

No source code in `ka9q-web.c`, the JS files, or `admin.py` is modified.
The legacy build still works from inside `legacy/`.
