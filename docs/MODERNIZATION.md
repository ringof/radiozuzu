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

- `spectrum.py` task per WS session: use ka9q-python's
  [`SpectrumStream`](https://github.com/mijahauan/ka9q-python) (added in
  v3.12.0) — channel creation, periodic polling, SSRC filtering, and
  retune are all built in. Per-frame callback receives `ChannelStatus`
  with `status.spectrum.bin_power_db` already populated (dB values
  decoded from either `BIN_DATA` float32 or `BIN_BYTE_DATA` uint8 form).
- WS frame: `{type:"spectrum", center_hz, bin_width_hz, base_db, step_db,
  bins:[...]}`.
- Wire palomar's spectrum/waterfall path to consume the new frame.

**Acceptance:** browser shows a live spectrum sourced through
`SpectrumStream`.

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

- Implement the rest of the WS control surface against `RadiodControl`,
  including post-detection audio shift (`SHIFT_FREQUENCY` →
  `set_shift_frequency`) and the CW left/right auto-flip behavior on
  cwu↔cwl mode transitions (frontend logic, mirrors upstream).
- Rewrite palomar's WS calls from text protocol (`F:`, `M:`, `Z:c:`, …)
  to typed JSON. Closes `overlay-bugs.md` "frequency stutter" — one
  Python authority replaces the radio.js/server-state race.
- Implement reliable command envelope in `docs/PROTOCOL.md`: every
  client→server JSON message carries `seq: int` (monotonic per-client);
  server replies `{type:"ack", seq}` for state-changing commands.
  Mirrors upstream's `C:<clientId>:<seq>:<payload>` / `ACK:<clientId>:<seq>`
  semantics in JSON form.
- Fold `admin/admin.py` into FastAPI as `/admin` routes; tracking comes
  from in-process WS session state. Keep SQLite for history only.
- Closes `issue.md` (real client IP) — read `CF-Connecting-IP` /
  `X-Forwarded-For` directly from `request.headers` with a trust list.

**Acceptance:** every overlay button works; admin dashboard works;
client IPs correct behind Cloudflare; UPSTREAM.md parity checklist all
ticked.

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

## ka9q-python coverage (gating Phase 4)

**Pinned version:** `mijahauan/ka9q-python` ≥ **v3.12.0**
([`db2aba9`](https://github.com/mijahauan/ka9q-python/commit/db2aba92bb444ea39872a0313e231bf9977d94fd),
2026-05-07) — adds spectrum bin decoding (`bin_data`, `bin_byte_data`,
`bin_power_db`) plus a high-level `SpectrumStream` class. **R1 is
resolved.**

After cross-referencing `mijahauan/ka9q-python`'s `docs/API_REFERENCE.md`,
the original ten-item task list is **fully covered**:

- Control: `set_frequency`, `set_preset`, `set_filter` (covers
  LOW_EDGE/HIGH_EDGE/kaiser_beta), `set_shift_frequency` (SHIFT_FREQUENCY),
  `set_spectrum` (with bin_bw / bin_count / and likely avg / overlap /
  window_type / shape kwargs), `poll_status`, `discover_channels` ×3
  flavors, `tune` one-shot — all present.
- Enums: `WindowType` (KAISER=0…HP5FT=8) is an **exact match**;
  `DemodType.SPECT2_DEMOD = 4`; `Encoding.OPUS = 3` plus full Opus
  parameter setters (`set_opus_bitrate`, `set_opus_dtx`,
  `set_opus_application`, `set_opus_bandwidth`, `set_opus_fec`).
- Status: `ChannelStatus` exposes frequency, first/second LO, low/high
  edge, preset, baseband_power, noise_density, output_level, gps_time,
  rtp_timesnap, plus nested `spectrum` (avg, overlap, window_type,
  shape, base, step, fft_n, resolution_bw, noise_bw, bin_count) and
  `frontend` (input_samprate, rf_agc, rf_atten, rf_gain, rf_level_cal,
  if_power).
- Streaming: `RTPRecorder(channel, on_packet=cb, pass_all_packets=True)`
  yields `(RTPHeader, bytes, wallclock)` — **codec-agnostic Opus
  passthrough is satisfied today** without any new ka9q-python work.
  `ManagedStream` provides `on_stream_dropped` / `on_stream_restored`
  + `restore_interval_sec`, replacing the upstream watchdog logic.
  `MultiStream` provides shared-socket fan-out. `rtp_to_wallclock()`
  helper is present.
- **Spectrum (new in v3.12.0):** `SpectrumStream` handles channel
  creation, periodic polling, SSRC filtering, retune via
  `set_frequency()`, and context-manager lifecycle. Each callback
  receives a `ChannelStatus` with `spectrum.bin_data`,
  `spectrum.bin_byte_data`, and the convenience
  `spectrum.bin_power_db` property already populated. Closes both the
  critical "where is the bin array" question (item 1) and the workflow
  documentation question (item 2).

### Remaining ka9q-python items (all optional)

The two critical items are resolved upstream. The remainder are
nice-to-haves and do not block any phase:

| # | Item | Severity |
|---|---|---|
| 3 | A `stop_spectrum(spec_ssrc)` helper that wraps the upstream "tune to 0 Hz × 3" idiom, or a docstring on `remove_channel()` clarifying it's the right call for a SPECT2_DEMOD channel. | Optional — `SpectrumStream`'s context manager likely handles teardown. |
| 4 | Surface `Frontend.samples`, `overranges`, `samp_since_over` if available in the TLV stream. Cosmetic — used for diagnostics. | Optional. |
| 5 | A loopback fake-radiod test fixture and TLV golden files — useful for radiozuzu CI. | Optional. |

## Upstream parity reference

The legacy stack we're replacing tracks
[wa2n-code/ka9q-web](https://github.com/wa2n-code/ka9q-web) as its
upstream. The new app must reach feature parity with it.

- **Reference pin:** `wa2n-code/ka9q-web` @ `88b28ee9` (2026-03-25).
- Re-fetch monthly during the transition; record bumps in
  [`UPSTREAM.md`](UPSTREAM.md), which holds the per-feature parity
  checklist (one row per WS command + per UI feature, ticked off as
  Phase 4 lands each).
- Notable upstream-only features vs the snapshot in `legacy/`:
  - `T:<shift>` WS command + `SHIFT_FREQUENCY` TLV (post-detection
    audio shift) — already covered by `set_shift_frequency`.
  - `C:<clientId>:<seq>:<payload>` reliable command envelope with
    `ACK:<clientId>:<seq>` replies — reproduced as a `seq` / `{type:"ack",
    seq}` JSON envelope in `docs/PROTOCOL.md`.
  - CW left/right auto-flip on cwu↔cwl transitions — frontend logic,
    rebuilt in `web/src/app.js`.
  - `ENABLE_MONITOR` watchdog with exponential backoff — superseded by
    `ManagedStream` recovery callbacks.

We do **not** plan to maintain a fresh fork of `ka9q-web.c`; the
`legacy/` copy is a known-working snapshot and stays as-is during
cutover.

## Risks (open)

| ID | Risk | Status |
|---|---|---|
| R1 | TLV coverage in ka9q-python. | **Resolved** — ka9q-python v3.12.0 (`db2aba9`, 2026-05-07) adds `SpectrumStatus.bin_data` / `bin_byte_data` / `bin_power_db` and a `SpectrumStream` class. Phase 2 is unblocked. Pin `ka9q-python >= 3.12.0` in `server/pyproject.toml`. |
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
