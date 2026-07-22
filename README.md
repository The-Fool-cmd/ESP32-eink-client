# ESP32 dashboard client

ESPHome firmware for a TRMNL 7.5" (OG) DIY Kit e-ink display.

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

XIAO ESP32-S3 Plus driver board + 7.5" monochrome e-paper panel
(Seeed/Waveshare `7.50inv2`). Display/SPI pins are per Seeed's ESPHome
cookbook:
https://wiki.seeedstudio.com/ogdiy_kit_works_with_esphome/

Battery sensing (GPIO1 ADC behind a GPIO6 load switch) is per Seeed's actual
TRMNL firmware instead, which covers this board more specifically:
https://github.com/Seeed-Projects/Seeed_TRMNL_Eink_Project

This board's Boot button isn't wired (only Reset is) — see the Flashing
section below.

## Setup

```
cp secrets.yaml.example secrets.yaml
```

Fill in `secrets.yaml`:
- `wifi_ssid` / `wifi_password`
- `device_id` — this device's id on the render server, used to build the
  image URL (`/image/<device_id>.png`)

## Flashing

Always over USB — this device deep-sleeps almost all the time and only
stays awake briefly per cycle, so it's not reliably reachable for OTA:

```
esphome compile trmnl-dashboard.yaml
esphome upload trmnl-dashboard.yaml --device /dev/ttyACMx
```

(`esphome run` does both in one go, but compiling first means the upload
itself — the part racing the device's awake window — is as fast as
possible.)

To find the port: `udevadm monitor --subsystem-match=tty --property` and
watch for the `add` event when you plug the board in.

Since there's no Boot button on this board, you can't force download mode
manually — flashing relies on the native USB auto-reset working, which
needs the device actually awake (not mid deep-sleep). The cold-boot grace
period above exists mostly for this reason: reflash it, and you get a
minute of awake time to catch it again before it sleeps.

## Configuration

Everything tunable lives in the `substitutions:` block at the top of
`trmnl-dashboard.yaml`:

- `render_host` / `render_port` / `image_path` — where to fetch the PNG from.
- `sleep_duration` — time between refreshes.
- `awake_timeout` — hard cap on awake time per cycle. If Wi-Fi or the fetch
  hangs, the device sleeps anyway rather than retrying.
- `cold_boot_awake_time` — extra awake time after a fresh flash/reset, to
  leave room for reflashing.
