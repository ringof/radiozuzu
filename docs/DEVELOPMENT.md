# Radio Zuzu Development Setup

The minimum environment to develop on Radio Zuzu. Two paths are
supported:

- **Path A (canonical)** — install [SigMonD](https://github.com/mijahauan/sigmond);
  it brings up `radiod`, manages the systemd lifecycle, and pins the
  ka9q-radio commit. radiozuzu develops as a SigMonD client.
- **Path B (frontend-only / no-SDR)** — bring your own radiod or use
  ka9q-radio's `sig_gen` synthetic frontend. Useful when you only want
  to iterate on `web/src/` or you don't have an SDR plugged in.

Both paths use [`uv`](https://docs.astral.sh/uv/) to manage Python
dependencies.

## Hardware

| Item | Required | Notes |
|---|---|---|
| Linux dev host | ✅ | One of apt / dnf / yum / pacman; Python ≥ 3.11. SigMonD's tested platforms (Debian 12+, Ubuntu 22.04+) are known-good; other modern distros work via the auto-detected package manager. macOS / Windows are not supported as the dev host because ka9q-radio is Linux-only. |
| SDR (rx888 mk II in production) | for Path A on real RF | Any ka9q-radio-supported frontend works for dev: rx888, RTL-SDR (cheap, ubiquitous), Airspy R2 / HF+, HackRF, BladeRF. |
| Antenna or signal source | for Path A on real RF | A wire to WWV at 5/10/15/20 MHz is the canonical "is anything working" smoke test. |
| Browser on your laptop / phone | ✅ | Audio output is browser-side. Any WS-capable modern browser. |
| Network multicast on the dev L2 segment | ✅ | radiod uses IP multicast for both audio streams and the control protocol. Develop on the SDR host directly, or be on the same VLAN. |

> **No SDR? No problem.** ka9q-radio ships a `sig_gen` daemon that
> synthesizes a frontend (gaussian noise + configurable carriers).
> See Path B below.

## Pinned versions

| Project | Pin | Source of truth |
|---|---|---|
| ka9q-radio | **inherited from SigMonD** (Path A) or whatever you check out (Path B) | We dropped our own submodule and `radiod.commit` in Phase 0.5. |
| ka9q-python | `>= 3.12.0` ([`db2aba9`](https://github.com/mijahauan/ka9q-python/commit/db2aba92bb444ea39872a0313e231bf9977d94fd)) | `server/pyproject.toml` |
| ka9q-radio commit ka9q-python is tested against | [`5498aefd`](https://github.com/ka9q/ka9q-radio/commit/5498aefd6fd4be7d4ff2f5e33c9b310ecd3b8574) | `ka9q.compat.KA9Q_RADIO_COMMIT`. Recorded for cross-check; if SigMonD's pin diverges far from this, file an issue. |

See [`UPSTREAM.md`](UPSTREAM.md) for the parity reference + pin history.

## Path A — SigMonD-canonical

Recommended whenever you'll be testing against real RF or you want the
deployment-shaped dev setup.

### 1. Install SigMonD

```bash
git clone https://github.com/mijahauan/sigmond.git
cd sigmond
./install.sh
```

`install.sh` auto-detects apt/dnf/yum/pacman, installs build deps
(libfftw3, libavahi-client, libopus, libsamplerate, libusb, …),
builds and installs `radiod`, creates the `radio` system user, and
sets up the systemd unit pattern `radiod@<station>.service`.

### 2. Configure your station

Use the radiod config under `config/` in this repo as a starting point:

```bash
sudo cp config/radiod@rx888-web.conf /etc/radio/radiod@dev.conf
sudo systemctl enable --now radiod@dev
```

For Phase 3 onward, the audio channel will be configured for Opus
output (`encoding = opus`); see [`MODERNIZATION.md`](MODERNIZATION.md)
Phase 3 for the exact directives.

### 3. Clone radiozuzu and install the dev environment

```bash
git clone https://github.com/ringof/radiozuzu.git
cd radiozuzu/server
uv sync                  # creates .venv, installs ka9q-python and the rest from uv.lock
```

### 4. Run the dev server

```bash
uv run radiozuzu --status-host dev-status.local
```

Then open `http://<host>:<port>/` (defaults set in `radiozuzu.toml`).
The server discovers SSRCs on the radiod's mDNS status address and
serves the frontend from `web/`.

### 5. Verify audio (remote dev)

For everyday work, the browser on your laptop is the audio output.
Open the radiozuzu URL → press the audio button → you should hear
demodulated audio.

For headless smoke tests, ka9q-radio ships two CLI tools:

- `monitor` — interactive multi-channel listener
- `pcmrecord <multicast-addr>` — saves the demod stream to a WAV file

Useful when you want to confirm "is *anything* coming out of the
demod" without involving the browser layer.

## Path B — Frontend-only / no-SDR

Use this when you don't have an SDR available, or you just want to
work on `web/src/` without standing up radiod through SigMonD.

### 1. Install ka9q-radio (manual)

If you want a minimal radiod install without SigMonD, follow ka9q-radio's
[INSTALL.md](https://github.com/ka9q/ka9q-radio/blob/main/docs/INSTALL.md).

### 2. Run sig_gen as a synthetic frontend

`sig_gen` is a daemon shipped with ka9q-radio that emits gaussian
noise and configurable single carriers. It plugs into the same
multicast control plane radiod uses, so as far as radiozuzu is
concerned it looks like a real SDR.

```bash
# example — see /etc/radio/sig_gen@*.conf or ka9q-radio docs for the full schema
sudo systemctl start sig_gen@dev
sudo systemctl start radiod@dev   # configured to read from sig_gen's multicast stream
```

This is enough to exercise tune / mode / spectrum / audio paths
end-to-end. Without an antenna, you'll hear noise plus whatever
carriers sig_gen is configured to emit — perfect for verifying
the WS protocol and the frontend.

### 3. Same dev-environment install as Path A

```bash
cd radiozuzu/server
uv sync
uv run radiozuzu --status-host dev-status.local
```

## Network and discovery

- radiod publishes its status address via mDNS (Avahi). Default name
  is `<station>-status.local`. From your dev host this should resolve
  via `avahi-resolve -n <name>` and `getent hosts <name>`.
- IP multicast lives in `239.0.0.0/8`. radiod chooses a deterministic
  group from a hash of the station name, plus a few well-known control
  groups.
- Dev typically runs on the SDR host directly. If your dev laptop is
  separate, both must be on the same L2 segment unless you've set up
  multicast routing — which is rarely worth it.

## Test signals

| Signal | Use |
|---|---|
| WWV @ 5 / 10 / 15 / 20 MHz | Canonical real-RF reference. Audible from anywhere with a wire antenna; gives a known-good AM signal with timing pulses. |
| sig_gen carrier | Reproducible, no antenna required. Good for protocol regression tests. |
| sig_gen wideband noise | Stress-tests spectrum/waterfall rendering and S-meter accuracy. |

## Updates

- **Path A:** `cd sigmond && ./install.sh --update` (or whatever
  update workflow SigMonD ships). This bumps the ka9q-radio pin.
- **Path B:** update your ka9q-radio checkout manually.
- **radiozuzu's ka9q-python pin** is bumped via `uv lock --upgrade-package ka9q-python`
  in `server/`, then committed.
- **Cross-check after any ka9q-radio bump:** ensure
  `ka9q.compat.KA9Q_RADIO_COMMIT` is satisfied by the new pin. If it
  isn't, file an issue and either revert or update both at once.

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| `discover_channels()` returns empty | mDNS not resolving; dev host not on the radiod's L2 segment; `radiod@<station>` not running. |
| Spectrum frame arrives but audio is silent | Browser AudioContext suspended (needs a user gesture); radiod channel not configured for Opus output (Phase 3). |
| Audio is choppy with `RESEQUENCE_TIMEOUT` gaps | Network loss between radiod and dev host; bump `resequence_buffer_size` on the `ManagedStream` / `RTPRecorder`. |
| `Address already in use` on dev port | A previous `uv run radiozuzu` didn't shut down. `ss -tlnp \| grep <port>` to identify, then kill. |

## See also

- [`MODERNIZATION.md`](MODERNIZATION.md) — phases, decisions, risks
- [`UPSTREAM.md`](UPSTREAM.md) — parity checklist + version pins
- [SigMonD](https://github.com/mijahauan/sigmond)
- [ka9q-radio](https://github.com/ka9q/ka9q-radio) /
  [INSTALL.md](https://github.com/ka9q/ka9q-radio/blob/main/docs/INSTALL.md)
- [ka9q-python](https://github.com/mijahauan/ka9q-python) /
  [API_REFERENCE.md](https://github.com/mijahauan/ka9q-python/blob/main/docs/API_REFERENCE.md)
