# ESP32 dashboard client

ESPHome firmware for a TRMNL-style e-ink display. Two board/panel
combos are supported, with three yamls between them:

- `trmnl-7.5in-og-diy-kit.yaml` — TRMNL 7.5" (OG) DIY Kit, monochrome.
- `trmnl-ee04-7.3in-spectra6.yaml` — Seeed EE04 driver board + Waveshare
  7.3" E Ink Spectra 6 (E6), full color, landscape mount.
- `trmnl-ee04-7.3in-spectra6-vertical.yaml` — same EE04 + Spectra 6
  combo, for a device mounted in portrait orientation.

## How it works

This is the firmware half of a two-repo setup. All the actual rendering —
layout, images, battery icon — happens on the server side:
[ESP32-eink-server](https://github.com/The-Fool-cmd/ESP32-eink-server), a
Flask + Playwright service that turns a dashboard page into a PNG. This
repo just knows how to fetch and display it.

Each cycle:

1. Wake up (from deep sleep, or a fresh boot/reset).
2. Connect to Wi-Fi.
3. Read the battery, and GET a PNG from the render server (device id and
   battery level go in as query params).
4. Draw it on the e-ink panel.
5. Go back to deep sleep until next time.

No scheduling, state, or rendering logic lives here — if you want to change
what shows up on the screen, that happens in the server repo, not this one.

On a fresh flash/reset/power-cycle, the device stays awake a bit longer
than usual (`cold_boot_awake_time`) instead of immediately sleeping again,
so there's a real window to reflash it over USB.

## Hardware

### `trmnl-7.5in-og-diy-kit.yaml`

XIAO ESP32-S3 Plus driver board + 7.5" monochrome e-paper panel
(Seeed/Waveshare `7.50inv2`). Display/SPI pins are per Seeed's ESPHome
cookbook:
https://wiki.seeedstudio.com/ogdiy_kit_works_with_esphome/

Battery sensing (GPIO1 ADC behind a GPIO6 load switch) is per Seeed's actual
TRMNL firmware instead, which covers this board more specifically:
https://github.com/Seeed-Projects/Seeed_TRMNL_Eink_Project

This board's Boot button isn't wired (only Reset is) — see the Flashing
section below.

### `trmnl-ee04-7.3in-spectra6.yaml`

Seeed EE04 driver board (also XIAO ESP32-S3 Plus based) + Waveshare 7.3"
E Ink Spectra 6 (E6) 6-color panel. Shares the OG kit's SPI/display pins
and battery-sense pins, but uses ESPHome's `epaper_spi` component
(`model: "7.3in-Spectra-E6"`) instead of `waveshare_epaper` — the E6
panel's controller protocol is completely different from the monochrome
one. Pins per Seeed's EE04 cookbook:
https://wiki.seeedstudio.com/EE04_with_esphome_advanced/

Being a color panel, `online_image` uses `type: RGB` instead of `BINARY` —
the render server needs to serve an image already quantized to the Spectra
6 palette (black/white/red/green/blue/yellow) for this to look right.

`test.yaml` is a minimal, standalone config for this board/panel combo —
no wifi, no dashboard, just static colored text drawn on boot. Useful for
confirming the panel's wired correctly in isolation, without needing the
render server involved at all.

### `trmnl-ee04-7.3in-spectra6-vertical.yaml`

Identical to `trmnl-ee04-7.3in-spectra6.yaml` — same board, panel, and
pins — except the display is rotated 90° for a portrait mount, so the
logical canvas is 480x800 instead of 800x480. The render server needs to
serve a portrait-oriented image for whichever device id uses this yaml.
If the image comes out upside down, flip `rotation` to `270` instead.

## Setup

```
cp secrets.yaml.example secrets.yaml
```

Fill in `secrets.yaml`:
- `wifi_ssid` / `wifi_password`
- `device_id` — shared across all yamls, used to build the image URL
  (`/image/<device_id>.png`) and to derive the device's ESPHome
  name/hostname. Flashing is always over USB, one device at a time, so
  edit this by hand to match whichever physical device is on the wire
  before you compile/upload — it's not meant to hold every device at once.

## Flashing

Always over USB — these devices deep-sleep almost all the time and only
stay awake briefly per cycle, so they're not reliably reachable for OTA.
Substitute whichever yaml matches your board:

```
esphome compile trmnl-7.5in-og-diy-kit.yaml
```

```
esphome upload trmnl-7.5in-og-diy-kit.yaml --device /dev/ttyACMx
```

```
esphome logs trmnl-7.5in-og-diy-kit.yaml --device /dev/ttyACMx
```

(`esphome run` does both in one go, but compiling first means the upload
itself — the part racing the device's awake window — is as fast as
possible.)

To find the port:

```
udevadm monitor --subsystem-match=tty --property
```

and watch for the `add` event when you plug the board in.

Neither board has a wired Boot button, so you can't force download mode
manually — flashing relies on the native USB auto-reset working, which
needs the device actually awake (not mid deep-sleep). The cold-boot grace
period above exists mostly for this reason: reflash it, and you get a
minute of awake time to catch it again before it sleeps.

## Configuration

Everything tunable lives in the `substitutions:` block at the top of each
yaml:

- `render_host` / `render_port` / `image_path` — where to fetch the PNG from.
- `sleep_duration` — time between refreshes.
- `awake_timeout` — hard cap on awake time per cycle, sized to cover
  `max_fetch_retries` worth of retries plus Wi-Fi connect/display draw time.
- `max_fetch_retries` — how many times to retry a failed image fetch within
  the same wake cycle before giving up until the next one.
- `cold_boot_awake_time` — extra awake time after a fresh flash/reset, to
  leave room for reflashing.

`trmnl-ee04-7.3in-spectra6.yaml` additionally has `refresh_wait_time` — how
long to wait after triggering a display update before sleeping, so the
panel's refresh actually finishes first.
