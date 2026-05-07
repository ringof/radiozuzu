# Upstream Parity Checklist

Tracks feature parity between the new Radio Zuzu (FastAPI + ka9q-python)
and the upstream C `ka9q-web` we're replacing.

**Reference pins:**

| Project | Role | Pinned at | Date |
|---|---|---|---|
| [`wa2n-code/ka9q-web`](https://github.com/wa2n-code/ka9q-web) | Feature parity reference (the C/JS stack we're replacing) | `88b28ee9` | 2026-03-25 |
| [`mijahauan/ka9q-python`](https://github.com/mijahauan/ka9q-python) | Runtime dependency (`server/pyproject.toml`) | **v3.12.0** ([`db2aba9`](https://github.com/mijahauan/ka9q-python/commit/db2aba92bb444ea39872a0313e231bf9977d94fd)) | 2026-05-07 |
| [`ka9q/ka9q-radio`](https://github.com/ka9q/ka9q-radio) | Submodule, tracking `main` | (floating) | — |

Re-fetch monthly during the transition. When you bump a pin, add a row
under "Pin history" at the bottom and update any new features into the
tables.

---

## WebSocket commands (client → server)

The upstream protocol is text (`F:<kHz>`, `M:<preset>`, …). The new app
uses typed JSON; this table maps each upstream command to its JSON
equivalent and to the `RadiodControl` call it routes through.

| Upstream | New JSON message | `RadiodControl` call | Status |
|---|---|---|---|
| `S:` (start spectrum) | `{type:"spectrum_start"}` | `SpectrumStream(...)` (v3.12.0+) | ☐ |
| `A:START` / `A:STOP` (audio) | `{type:"audio", action:"start"\|"stop"}` | `RTPRecorder.start()` / `stop()` | ☐ |
| `E:<low>:<high>` (filter edges) | `{type:"set_filter", low_hz, high_hz}` | `set_filter(low_edge=, high_edge=)` | ☐ |
| `G:<n>` (spectrum average) | `{type:"set_spectrum_avg", n}` | `set_spectrum(avg=n)` | ☐ |
| `W:<type>:<param>` (window) | `{type:"set_window", window_type, shape}` | `set_spectrum(window_type=, shape=)` | ☐ |
| `V:<frac>` (FFT overlap) | `{type:"set_spectrum_overlap", frac}` | `set_spectrum(overlap=frac)` | ☐ |
| `F:<kHz>` (tune) | `{type:"tune", freq_hz}` | `set_frequency` | ☐ |
| `M:<preset>` (mode) | `{type:"set_mode", preset}` | `set_preset` | ☐ |
| `T:<hz>` (audio shift) | `{type:"set_shift", shift_hz}` | `set_shift_frequency` | ☐ |
| `R:<ms>` (poll interval) | `{type:"set_poll", ms}` | (server-side timer) | ☐ |
| `Z:+`, `Z:-`, `Z:<lvl>`, `Z:c:<freq>`, `Z:SIZE` (zoom) | `{type:"zoom", op, value?}` | (server-side state + `set_spectrum` re-config) | ☐ |
| `C:<clientId>:<seq>:<payload>` (reliable wrapper) | `seq` field on every message + `{type:"ack", seq}` reply | (envelope) | ☐ |

## Server → client frames

| Upstream | New form | Status |
|---|---|---|
| Spectrum text/binary frame on WS | `{type:"spectrum", center_hz, bin_width_hz, base_db, step_db, bins:[...]}` | ☐ |
| Opus audio binary frame on WS | WS binary opcode 0x2: `[u32 ts_delta][u16 seq][u16 opus_len][opus_bytes]` (see `PROTOCOL.md`) | ☐ |
| `S:<ssrc>` reply | `{type:"spectrum_started", ssrc}` | ☐ |
| `ACK:<clientId>:<seq>` | `{type:"ack", seq}` | ☐ |
| Status / S-meter fields (baseband_power, noise_density, output_level, frontend rf params) | `{type:"status", channel:{...}, frontend:{...}}` (subset of `ChannelStatus.to_dict()`) | ☐ |

## UI / frontend behaviors

| Upstream behavior | Where it lives now | Status |
|---|---|---|
| Spectrum + waterfall canvas | `web/src/spectrum.js` (kept) | ☐ |
| S-meter | `web/src/smeter.js` (kept) | ☐ |
| PCM/Opus playback | `web/src/pcm-player.js` + `opus-decoder.min.js` (kept; framing harmonized with server) | ☐ |
| Passband drag handles | palomar (kept) | ☐ |
| Click-to-tune on frequency scale | palomar (kept) | ☐ |
| Wheel zoom / drag pan | palomar (kept) | ☐ |
| Help modal (mouse, keyboard, fullscreen, panel) | palomar (kept) | ☐ |
| CW left/right auto-flip on cwu↔cwl transitions | rebuild in `web/src/app.js` | ☐ |
| Reliable command envelope (per-msg `seq` + ack) | new in `web/src/app.js` | ☐ |
| Fullscreen on `#p-overlay` (not the spectrum canvas) | palomar (kept; Phase 2 of fullscreen still TBD per `overlay-bugs.md`) | ☐ |

## Stuck-on-server features (Phase 6)

| Feature | Path forward |
|---|---|
| Record button | `RTPRecorder.start_recording()` / `stop_recording()` + server-side file naming |
| Memories (save/recall freq + mode + label) | server-side JSON or SQLite |
| Band presets (amateur/broadcast/utility/CB) | static config + server-side enum |
| DX labels toggle + database | server-side dx.json hosting + WS toggle |
| Ext menu (CSV, screenshot, settings reset) | menu in `app.js` |
| Filter edge numeric entry | UI in `app.js` (paired with the existing drag handles) |
| Touch gestures (single-finger pan, tap-to-tune, pinch zoom, two-finger slide) | rewritten input layer in `app.js` |

## Operational parity

| Upstream feature | New form | Status |
|---|---|---|
| Real client IP behind Cloudflare Tunnel (currently shows 127.0.0.1) | Read `CF-Connecting-IP` / `X-Forwarded-For` from `request.headers`, trust list configurable. Closes `docs/issue.md`. | ☐ |
| Watchdog for stalled multicast / Status_fd / Input_fd / spectrum stream | `ManagedStream` `on_stream_dropped` / `on_stream_restored` + `restore_interval_sec` | ☐ |
| Spectrum-restart exponential backoff | Same as above | ☐ |
| Per-IP and global connection limits (nginx `limit_conn`) | unchanged — keep the nginx `limit_conn` zones | ☐ |

## Pin history

| Date | Project | Pin | Notes |
|---|---|---|---|
| 2026-05-07 | wa2n-code/ka9q-web | `88b28ee9` | Initial reference pin alongside Phase 0. |
| 2026-05-07 | mijahauan/ka9q-python | v3.12.0 (`db2aba9`) | Adds `SpectrumStatus.bin_data` / `bin_byte_data` / `bin_power_db` and the `SpectrumStream` class — resolves R1 (the gating ka9q-python question for Phase 2). |
