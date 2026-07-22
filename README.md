# ESP32 dashboard client

ESPHome firmware for a TRMNL 7.5" (OG) DIY Kit e-ink display. Firmware does the
minimum possible: wake → Wi-Fi connect → HTTP GET a PNG from the render server
→ blit it to the panel → deep sleep. All layout/rendering logic lives
server-side (separate Playwright + Flask service); this repo has no scheduling
or state of its own.

## Hardware

XIAO ESP32-S3 Plus driver board + 7.5" monochrome e-paper panel (Seeed/Waveshare
`7.50inv2`), per Seeed's official ESPHome cookbook:
https://wiki.seeedstudio.com/ogdiy_kit_works_with_esphome/

Pins, board type (`esp32-s3-devkitc-1`), and PSRAM config (`octal`, `80MHz`) in
`trmnl-dashboard.yaml` are copied from that cookbook, not guessed.

## Setup

```
cp secrets.yaml.example secrets.yaml   # already done locally; edit with real values
```

Fill in `secrets.yaml`:
- `wifi_ssid` / `wifi_password`
- `ota_password` — used for future OTA updates
- `device_id` — this device's id as registered on the render server (used to
  build the per-device image URL, e.g. `/image/<device_id>.png`)

First flash must be over USB (`esphome run trmnl-dashboard.yaml`). Subsequent
updates can go over OTA once the device is on the network — though note this
device deep-sleeps almost all the time, so OTA only has a ~25s window each
wake cycle to catch it (see `awake_timeout` below).

## Configuration knobs

All in the `substitutions:` block at the top of `trmnl-dashboard.yaml`:

- `render_host` / `render_port` / `image_path` — where to fetch the PNG from.
  Server now routes per-device (`/image/<device_id>.png` instead of a single
  shared `/image.png`); `device_id` itself lives in `secrets.yaml` (see
  Setup above) and is pulled in via `!secret`, not hardcoded here. Nothing
  else in this config assumes anything about how that service is deployed
  beyond host:port.
- `sleep_duration` — time between refreshes (default 10min).
- `awake_timeout` — hard cap on awake time per cycle (default 25s). If Wi-Fi
  or the HTTP fetch hangs, the device force-sleeps after this long rather than
  retrying. No backoff/retry loops by design — a bad wake just tries again
  next cycle.

## Open decision: dithering

The render server currently serves a plain grayscale/RGB PNG — it does not yet
dither down to the panel's native 1-bit depth. On the firmware side,
`online_image` converts the fetched PNG to `type: BINARY` (1bpp) for display,
but **`online_image` has no `dither` option** — only ESPHome's separate,
*local* `image:` component (for images compiled into firmware) supports one.
So today, on-device conversion is a naive per-pixel threshold, not real
dithering.

This is functional but will look flat/blocky for anything with gradients or
anti-aliased text/icons. Two ways forward, not yet decided:

1. **Server-side dithering (recommended)** — have the render server convert
   to 1-bit with Floyd–Steinberg (e.g. Pillow's `Image.convert("1")`) before
   serving the PNG. Firmware doesn't change at all: it already requests
   `type: BINARY`, and thresholding a PNG that's already 1-bit is a no-op.
2. **Leave it on-device** — accept the plain threshold, or explore whether a
   different ESPHome component/path exposes dithering for HTTP-fetched images
   (nothing found in current ESPHome docs as of this writing).

Recommend (1), since it keeps the firmware dumb (matches the "server decides
what's on screen" design) and gives more control over the dither algorithm.
Not implemented here — flagging for a decision before spending more time on
either side.

## Open decision: battery indicator

The render server accepts an optional `?battery=<0-100>` query param on the
image request and draws a small battery glyph if present. No battery sensor
is configured in this file yet — `image_path`'s URL has no query string, so
the server just omits the indicator, which is fine as a default.

To wire it up, this repo would need a `sensor:` block reading the battery
ADC, then `online_image`'s `url:` would switch from a plain substitution to
a `lambda` appending `"?battery=" + to_string(int(id(battery_percent).state))`.
Seeed's cookbook for this exact board (XIAO ESP32-S3 Plus) documents a
battery voltage sensor on **GPIO1** (`platform: adc`, `attenuation: 12db`,
`multiply: 2.0` to undo an onboard divider) plus a calibration curve mapping
~3.27V–4.15V to 0–100%. Whether that's worth adding depends on whether this
particular board was actually built/wired with the battery ADC tap — not
all DIY kit assemblies populate it. Not implemented here; flagging for a
decision once that's confirmed.

## Why no `api:` component

This device has no Home Assistant entities to expose and is fully driven by
the render server, so the ESPHome `api:` component is omitted to save RAM and
a bit of awake time. Add it back only if HA integration becomes a requirement.
