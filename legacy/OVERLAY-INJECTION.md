# Palomar SDR Overlay

A KiwiSDR-style overlay UI for ka9q-web's `radio.html`. The overlay
replaces the default interface with a streamlined panel layout featuring
a custom spectrum trace, waterfall, S-meter, frequency/mode controls,
and draggable passband handles.

The overlay is a standalone script (`palomar-overlay.user.js`) that
interacts with radio.html only through DOM manipulation — it does not
modify or depend on radio.js internals. It works two ways:

| Method | How it loads | Who sees it |
|--------|-------------|-------------|
| **nginx injection** | `sub_filter` inserts a `<script>` tag into radio.html | All visitors |
| **Tampermonkey** | Browser extension injects the script client-side | Only you |

---

## Option A: nginx Injection (server-side)

No changes to ka9q-web source files required. nginx injects the overlay
script into radio.html responses at the proxy layer.

### 1. Copy the script to a location nginx can read

```bash
sudo cp palomar-overlay.user.js /var/www/palomar-overlay.user.js
```

### 2. Add two location blocks to your nginx site config

Edit your site config (e.g. `/etc/nginx/sites-available/sdr`) and add
these two blocks inside `server { }`, **above** the existing
`location / { }`:

```nginx
    # Inject overlay into radio.html
    location = /radio.html {
        limit_conn sdr_conn 3;
        limit_conn sdr_global 20;
        limit_conn_status 429;

        proxy_pass http://localhost:8073;
        proxy_set_header Host             $host;
        proxy_set_header X-Real-IP        $remote_addr;
        proxy_set_header CF-Connecting-IP $http_cf_connecting_ip;
        proxy_set_header Accept-Encoding "";

        sub_filter '</body>' '<script src="/palomar-overlay.user.js"></script></body>';
        sub_filter_once on;
    }

    # Serve the overlay script
    location = /palomar-overlay.user.js {
        alias /var/www/palomar-overlay.user.js;
        default_type application/javascript;
    }
```

The existing `location / { }` block stays as-is and continues to handle
WebSockets and all other requests.

### 3. Test and reload

```bash
sudo nginx -t && sudo systemctl reload nginx
```

### 4. Verify

```bash
# Confirm the script tag is injected
curl -s http://localhost:8080/radio.html | grep palomar

# Confirm the script is served (should return 200)
curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/palomar-overlay.user.js
```

### Disabling

Comment out the two location blocks and reload nginx. No ka9q-web files
are modified, so there is nothing to revert.

### Without Cloudflare

If you're not using Cloudflare Tunnel, remove the `CF-Connecting-IP`
header and adjust `proxy_pass` and `limit_conn` directives to match
your setup. See the main [README](README.md) for the non-Cloudflare
nginx configuration.

---

## Option B: Tampermonkey (client-side)

Install the [Tampermonkey](https://www.tampermonkey.net/) browser
extension, then:

1. Open Tampermonkey → Create a new script
2. Paste the contents of `palomar-overlay.user.js`
3. Save (Ctrl+S)

The `@match *://*/radio.html*` directive in the script header will
activate the overlay on any ka9q-web radio.html page you visit.

---

## Configuration

Edit the `PRESETS` object near the top of `palomar-overlay.user.js` to
change default startup behaviour:

```javascript
const PRESETS = {
    autoStartAudio: true,    // click "▶ Audio" automatically on load
    // volume:      70,      // 0–100
    // mode:        'usb',   // am, sam, lsb, usb, cwu, cwl, fm, iq
    // frequency:   14225,   // kHz
    // wfMax:       -30,     // waterfall max dB
    // wfMin:       -120,    // waterfall min dB
    // spMax:       -30,     // spectrum max dB
    // spMin:       -130,    // spectrum min dB
};
```

Uncomment and set any value to apply it on page load. Set a key to
`null` to use the page default.

---

## Architecture

The overlay uses a **radio adapter** pattern. All interaction with the
host page goes through a `radio` object defined at the top of the
script:

- **Reads** use window globals (`window.frequencyHz`, `window.spectrum`,
  etc.) which are `var`-declared in radio.js and therefore accessible.
- **Writes** use DOM manipulation — setting input values and dispatching
  events, or clicking existing buttons — so that radio.js's own handlers
  do the real work.

This means the overlay is resilient to radio.js internal changes. If
the upstream developers refactor their code, only the ~50-line `radio`
adapter needs updating — the 1200 lines of UI code remain untouched.

The one exception is panning (`radio.setCenter`), which requires
WebSocket access for the `Z:c:` command. This is wrapped in try/catch
and degrades gracefully if unavailable.

---

## Copyright and License

Copyright (C) 2024-2026 W1EUJ. Licensed under the
[GNU General Public License v3.0](LICENSE) or later.
