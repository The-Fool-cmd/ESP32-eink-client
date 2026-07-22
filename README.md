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

First flash must be over USB (`esphome run trmnl-dashboard.yaml`). Subsequent
updates can go over OTA once the device is on the network — though note this
device deep-sleeps almost all the time, so OTA only has a ~25s window each
wake cycle to catch it (see `awake_timeout` below).

## Configuration knobs

All in the `substitutions:` block at the top of `trmnl-dashboard.yaml`:

- `render_host` / `render_port` / `image_path` — where to fetch the PNG from.
  Defaults to `foolaiserver:8000/image.png`. Nothing else in this config
  assumes anything about how that service is deployed beyond host:port.
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

## Why no `api:` component

This device has no Home Assistant entities to expose and is fully driven by
the render server, so the ESPHome `api:` component is omitted to save RAM and
a bit of awake time. Add it back only if HA integration becomes a requirement.
