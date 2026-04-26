# Design Rationale

This document captures *why* the YAML in `common/s31-base.yaml` is structured the way it is. Every non-obvious choice — filter ordering, network timeouts, naming conventions, lambda guards — has a reason that isn't visible from the YAML alone. The intent of this doc is that anyone (including future-you) modifying the template can do so without having to re-derive these decisions.

For *what* each block does mechanically, refer to the [ESPHome component documentation](https://esphome.io/) linked inline. For *how* to deploy the template, see [`installation.md`](installation.md). For calibration math, see [`calibration.md`](calibration.md).

---

## Table of contents

1. [Power vs energy: a useful mental model](#power-vs-energy-a-useful-mental-model)
2. [Throttle filter strategy](#throttle-filter-strategy)
3. [Filter ordering rules](#filter-ordering-rules)
4. [The power-factor lambda gate](#the-power-factor-lambda-gate)
5. [Naming convention: friendly_name + short entity names](#naming-convention-friendly_name--short-entity-names)
6. [Areas as ESPHome metadata](#areas-as-esphome-metadata)
7. [Boot and state-restore behavior](#boot-and-state-restore-behavior)
8. [Network resilience](#network-resilience)
9. [Logger discipline](#logger-discipline)
10. [Web server philosophy](#web-server-philosophy)
11. [Diagnostic entity discipline](#diagnostic-entity-discipline)
12. [The `packages:` pattern and per-device calibration](#the-packages-pattern-and-per-device-calibration)

---

## Power vs energy: a useful mental model

**Power** (watts) is what's flowing *right now* — analogous to a car's speedometer.
**Energy** (kilowatt-hours) is what's flowed *in total* — analogous to an odometer.

The relationship: `energy = power × time`. Energy is the area under the power-vs-time curve.

Why this matters for filter choice:

- **Power is a *gauge***: it can go up or down, at any rate, at any time. Smoothing makes sense (averaging removes noise without distorting meaning).
- **Energy is a *counter***: it only ever increases. Averaging a counter is meaningless — the average of `12.340 / 12.341 / 12.342` is `12.341`, which is neither when the window started nor when it ended.

A reasonable downsampling for a gauge is `throttle_average` (smooth + emit periodically). For a counter, `throttle` is appropriate (sample the latest value, drop the rest).

This drives the filter choices for `power` (gauge → throttle_average) and `energy` (counter → throttle) in `common/s31-base.yaml`.

`★ Two energy sensors:` We expose both `Energy` (chip's lifetime counter) and `Daily Energy` (computed by ESPHome from integrated power). They serve different purposes: the chip's counter is more accurate (hardware-integrated at full sample rate); the daily counter is convenient for at-a-glance ("how much have I used today?"). HA's Energy dashboard works with either.

---

## Throttle filter strategy

[ESPHome's sensor filters](https://esphome.io/components/sensor/) include a few related-but-distinct primitives:

| Filter | Behavior |
|---|---|
| `throttle: Xs` | Drop intermediate samples; emit the latest one every X seconds |
| `throttle_average: Xs` | Accumulate samples for X seconds; emit the *mean* of all samples in window |
| `delta: N` | Only emit when the value has changed by ≥ N from the last emit |
| `or: [a, b]` | Emit when *either* of two filters would emit |

The shared template uses these in a specific pattern per channel:

| Channel | Filter chain | Reasoning |
|---|---|---|
| Voltage | `multiply` → `filter_out: nan` → `clamp 80–280` → `throttle_average: 60s` | Mains is slow; 60-s averaging is invisible. Clamp drops sentinel garbage from the chip during init. |
| Current | `multiply` → `filter_out: nan` → `throttle_average: 10s` → `or:[throttle: 60s, delta: 0.05]` | Tracks load changes; 10-s smoothing + heartbeat + delta-on-change |
| Power | `multiply` → `filter_out: nan` → `throttle_average: 10s` → `or:[throttle: 60s, delta: 5]` | Same dynamics as current; threshold of 5 W catches load on/off |
| Energy | `throttle: 60s` | Counter — sample, don't average |
| Apparent Power | `filter_out: nan` → `throttle_average: 30s` | Less time-critical than real power |
| Power Factor | `lambda` → `throttle_average: 60s` | Only meaningful at steady load; lambda guards against NaN/impossible values |

### Why the `or:` combinator on power and current

The naive choices are:

- `throttle_average: 60s` only — clean graphs, but a kettle that turns on takes up to 60 s to register in HA. Bad for automation.
- `throttle_average: 5s` only — fast response, but bloats the recorder DB with tiny fluctuations.

The `or:[throttle: 60s, delta: 5]` combinator solves both:

- A **steady load** publishes once per minute (the `throttle` arm fires).
- A **load change** of ≥5 W publishes immediately (the `delta` arm fires).

Result: graphs stay clean during steady operation, and automations react within ~10 seconds (the `throttle_average` window) of any real change.

`★ Reference:` see [ESPHome filters documentation](https://esphome.io/components/sensor/) for the full list of available filters and their composition rules.

---

## Filter ordering rules

Filters chain like a Unix pipeline: each filter receives the previous filter's output. Order matters.

The convention in `common/s31-base.yaml`:

1. **`multiply: ${cal}` first** — calibration is applied to the raw chip value before anything else sees it. Otherwise, downstream filters operate on uncalibrated values, which makes their thresholds wrong.
2. **`filter_out: nan` early** — drop garbage before downstream filters can be confused. The CSE7766 occasionally emits NaN frames during init; without this filter, those NaNs propagate into averages and corrupt them.
3. **`clamp` middle (where used)** — enforce physical sanity (voltage 80–280 V) after calibration but before averaging. Catches sentinel values from the chip during boot.
4. **Time-based throttling last** — smooth or sample the survivors.

If you ever need to add a filter, decide which "tier" it belongs to and slot it in at the right point.

---

## The power-factor lambda gate

The power-factor channel has a small lambda filter that drops invalid samples:

```yaml
power_factor:
  filters:
    - lambda: |-
        if (isnan(x) || x < 0.0 || x > 1.0) return {};
        return x;
    - throttle_average: 60s
```

Three things this guards against:

1. **NaN samples** at idle. The chip can't compute PF with no current flowing (division by zero on apparent power). Some chips emit NaN explicitly; some emit garbage. The `isnan(x)` check filters NaN; downstream filters (especially `throttle_average`) see only valid samples.
2. **Negative values**. Should never happen physically; if the chip emits one, it's a bug or a frame-decode glitch.
3. **Values > 1.0**. By definition, `PF = real_power / apparent_power ≤ 1` for any valid measurement. Some CSE7766 chips emit PF > 1 at high resistive load due to **calibration mismatch** between the real-power and apparent-power channels. The `x > 1.0` check drops these as physically impossible.

### Why this version replaced an earlier one

An earlier draft of the template used:

```yaml
- lambda: |-
    if (isnan(id(my_power).state) || id(my_power).state < 3.0) return {};
    return x;
```

This version *cross-references the power sensor's state* (`id(my_power).state`) rather than the local PF sample (`x`). Two problems with that approach:

1. **Race condition at boot**. ESPHome doesn't guarantee execution ordering between filter chains for sibling sensors. The PF lambda might evaluate before `my_power.state` has been updated for the current frame, even though both come from the same chip frame. Result: false NaN flags during boot.
2. **Doesn't catch PF > 1 from calibration mismatch**. The original gate only filters at low loads; at the high resistive loads where calibration-mismatch PF errors actually surface, the gate passes garbage through.

The replacement version inspects only `x` (the local sample), so there's no cross-sensor dependency, and it explicitly handles the chip's calibration-mismatch failure mode that we observed during real testing across four chips.

---

## Naming convention: friendly_name + short entity names

In the template, the device-level `friendly_name` is set:

```yaml
esphome:
  name: ${device_name}
  friendly_name: ${friendly_name}
  area: ${area}
```

And every entity uses a **short** name:

```yaml
sensor:
  - platform: cse7766
    power:
      name: "Power"        # NOT "${friendly_name} Power"
```

This works because of an ESPHome feature added in **2023.2.0**: when `friendly_name` is set on the device, **HA automatically prefixes it to entity names** at adoption time.

[ESPHome 2023.2.0 changelog](https://esphome.io/changelog/2023.2.0):

> ESPHome now supports setting a `friendly_name` which is sent to Home Assistant. This name will be used for the config entry, the device name, and **will be automatically prefixed to all of the entities where needed**.

So `name: "Power"` + `friendly_name: "Kitchen Outlet"` → HA displays "Kitchen Outlet Power."

### The duplication bug if you don't trust this

If you set `friendly_name` *and* prepend `${friendly_name}` manually to every entity name, HA prepends *again*, producing:

> "Kitchen Outlet Kitchen Outlet Power"

Older community configs that predate the 2023.2.0 changelog often manually prefix to control naming explicitly. After 2023.2.0, that prefix duplicates. The template here uses the post-2023.2 pattern.

### What about entity_id (the slug)?

The `entity_id` (used in templates and automations) is generated once at adoption time as `<domain>.<friendly_name>_<entity_name>` slugified:

> `friendly_name: Kitchen Outlet` + `name: Power` → `sensor.kitchen_outlet_power`

If you later change `friendly_name`, the *displayed* name updates but the `entity_id` stays frozen. Pick `friendly_name` values you can live with long-term.

---

## Areas as ESPHome metadata

`area: ${area}` in the `esphome:` block sends the area name to HA at adoption time. HA auto-creates the area if it doesn't exist and assigns the device to it.

This is **pure metadata** — the device doesn't know or care about its area. You can override the area in HA's UI later without changing the YAML. The YAML just supplies the *default* assignment.

Why bother: HA's automation system can target by area ("turn off everything in the office"), and dashboards can auto-group. Saving 4 manual area assignments at adoption time is small but non-zero ergonomic value.

`★ Reference:` see the [`esphome:` core component documentation](https://esphome.io/components/esphome.html) for `friendly_name`, `area`, and related fields.

---

## Boot and state-restore behavior

Three settings in the `esp8266:` block need explanation:

```yaml
esp8266:
  board: esp12e
  early_pin_init: false
  restore_from_flash: false
  framework:
    version: recommended
```

### `early_pin_init: false`

By default, ESPHome drives GPIO outputs to a known state very early in boot — before WiFi, before the API, before user logic runs. On a relay-controlling device, that "known state" is whatever the GPIO defaults to, which **briefly clicks the relay** on every boot or OTA flash. Audible, annoying, and bad for connected devices that don't appreciate spurious power cycles.

Setting `early_pin_init: false` defers GPIO initialization until *after* state-restore logic has run, so the relay shouldn't move out of its restored state during boot.

### `restore_from_flash: false`

ESP8266 flash has limited write endurance (~10,000–100,000 cycles per sector). Writing relay state to flash on every change would burn through that budget surprisingly fast on a frequently-toggled outlet.

Setting `restore_from_flash: false` tells ESPHome to *not* write switch state to flash. Instead, the GPIO switch's `restore_mode` controls boot behavior:

- `ALWAYS_ON` / `ALWAYS_OFF` — no need for stored state; the choice is hardcoded
- `RESTORE_DEFAULT_ON` / `RESTORE_DEFAULT_OFF` — uses RTC memory (volatile but survives soft reboots) instead of flash

This trades *some* state persistence (across full power cycles, the device returns to the default rather than the last commanded state) for flash longevity.

### Per-device `restore_mode`

The eight-substitution template lets each device pick its boot behavior:

| Mode | Use case |
|---|---|
| `ALWAYS_ON` | Routers, fridges, anything that should never be off after a power blip |
| `ALWAYS_OFF` | Heaters, 3D printers, anything you want explicit control over |
| `RESTORE_DEFAULT_OFF` | Lamps, electronics — restore last state, default off if uncertain |
| `RESTORE_DEFAULT_ON` | Same, but default on |

`★ Reference:` see [ESPHome `esp8266:` component](https://esphome.io/components/esp8266) for `early_pin_init`, `restore_from_flash`, and framework versioning.

---

## Network resilience

Three settings collectively make the S31 *not* reboot itself on transient network issues:

```yaml
wifi:
  reboot_timeout: 0s          # don't reboot if WiFi AP is briefly gone

api:
  reboot_timeout: 0s          # don't reboot if HA is briefly down

wifi:
  fast_connect: true          # cache BSSID/channel for fast reconnect
  power_save_mode: NONE       # mains-powered, prioritize responsiveness
  min_auth_mode: WPA2         # silence forward-compat warning (2026.6.0 default change)
```

### `reboot_timeout: 0s` on both `wifi:` and `api:`

By default, ESPHome reboots the device if it can't reach WiFi for 15 minutes (`wifi.reboot_timeout: 15min`) or can't reach Home Assistant's API for 15 minutes (`api.reboot_timeout: 15min`). The intent is "if we're in a bad state, restart and try again."

For an outlet, that behavior is undesirable. If your HA goes down for 30 minutes during an upgrade, you don't want every smart outlet **rebooting and possibly cycling its relay** during that window. Setting both timeouts to `0s` disables the auto-reboot — the device just keeps trying to reconnect indefinitely.

These are **independent** timers; both must be set explicitly.

### `fast_connect: true`

Caches the BSSID and channel of the last successful WiFi connection. On reconnect, the radio doesn't scan all channels; it goes directly to the known AP. Reconnect time drops from ~5 seconds to ~1 second.

The trade-off: if you replace your router or its primary AP fails permanently, the device won't search for alternate APs. For a stationary wall outlet, this trade-off is acceptable.

### `power_save_mode: NONE`

ESPHome defaults to `power_save_mode: LIGHT`, which puts the WiFi radio to sleep between AP beacon intervals. Saves ~30 mA on battery-powered devices. On a mains-powered outlet, the saving is meaningless, but the latency cost is real: HA sometimes registers the device as "unavailable" for a few seconds when it's actually mid-nap.

Setting `NONE` keeps the radio fully awake. Generally appropriate for AC-powered ESPHome devices.

### `min_auth_mode: WPA2`

Silences a forward-compatibility warning ESPHome emits about the default minimum WiFi auth mode changing from WPA to WPA2 in version 2026.6.0. Setting it explicitly to `WPA2` makes our intent clear and the warning go away. WPA2-or-better is recommended anyway — WPA (using TKIP) has been deprecated for years.

If your router only supports WPA (not WPA2), set this to `WPA` instead. (Routers that lack WPA2 are quite old at this point and worth replacing for general security reasons.)

`★ Reference:` see [ESPHome `wifi:` component](https://esphome.io/components/wifi) and [ESPHome `api:` component](https://esphome.io/components/api).

---

## Logger discipline

```yaml
logger:
  baud_rate: 0
  level: INFO
  logs:
    sensor: WARN
    cse7766: WARN
    component: WARN
```

### `baud_rate: 0`

Disables UART logging. **Critical for the S31** because the CSE7766 power-meter chip uses the ESP8266's UART RX line for its serial frames. If ESPHome's logger also tries to write to UART, the two compete and the CSE7766 frames become garbage — every sensor reading turns to noise.

Logs are still available via the API (visible in ESPHome Builder's live log view, or via `esphome logs <device.yaml>` from the CLI). You just don't get them on a serial cable.

### Per-component log levels

The `cse7766` and `sensor` components are chatty by default — they log every measurement event at INFO level. With four S31s reporting at ~22 Hz raw, the live log view becomes hard to read.

Setting these to WARN suppresses the per-sample noise while preserving real events (errors, init messages). Drop to DEBUG for one specific component when troubleshooting it:

```yaml
logger:
  level: INFO
  logs:
    cse7766: DEBUG    # temporarily, to investigate a calibration issue
```

`★ Reference:` see [ESPHome `logger:` component](https://esphome.io/components/logger).

---

## Web server philosophy

```yaml
web_server:
  port: 80
  version: 3
  auth:
    username: !secret web_username
    password: !secret web_password
```

Three deliberate choices:

### Monitor-only UI, no firmware upload

In current ESPHome, the web server's OTA capability is a **separate opt-in platform** under `ota:`. We deliberately don't enable it:

```yaml
ota:
  platform: esphome             # ← only this; not "- platform: web_server"
  password: !secret ota_password
```

Reasoning: we already have a working OTA path through ESPHome Builder (which uses the `esphome` OTA platform). Adding the `web_server` OTA platform creates a second firmware-upload mechanism with weaker auth (HTTP basic over plain HTTP, no encryption). Why expand the attack surface? If you ever want web-based OTA for emergency recovery, you can flash a temporary config with it enabled.

### `version: 3`

The current web UI version. Lighter on JS than v1/v2; built specifically to be lightweight on ESP8266-class hardware.

### Auth is non-negotiable

The web UI **can toggle the relay** even without OTA enabled. Anyone on your LAN who hits `http://<device>.local/` can flip your outlets. Auth via HTTP basic protects this — credentials live in `secrets.yaml` and are checked on every request.

This is HTTP basic over plain HTTP (no TLS, since the ESP8266 doesn't have resources for it). Acceptable on a trusted LAN; **never expose this UI to the internet** without putting it behind a reverse proxy with TLS.

`★ Reference:` see [ESPHome `web_server:` component](https://esphome.io/components/web_server) and [ESPHome `ota:` component](https://esphome.io/components/ota).

---

## Diagnostic entity discipline

Several entities use these two attributes:

```yaml
entity_category: diagnostic
disabled_by_default: true
```

What each does:

- **`entity_category: diagnostic`** — HA UI hint. Places the entity in a "Diagnostic" section on the device's page rather than mixing with primary entities. Cosmetic, but keeps dashboards uncluttered.
- **`disabled_by_default: true`** — entity is registered with HA but **not added to the recorder** until the user explicitly enables it. Prevents DB bloat for entities that exist for diagnostic purposes only.

Applied to:

| Entity | Both flags? |
|---|---|
| `Status` (binary_sensor.status — connected/disconnected) | `entity_category: diagnostic` only |
| `WiFi Signal` (sensor.wifi_signal) | `entity_category: diagnostic` only |
| `Uptime` (sensor.uptime) | `entity_category: diagnostic` only |
| `ESPHome Version` (text_sensor.version) | both |
| `IP` / `SSID` / `BSSID` / `MAC` (wifi_info text sensors) | both |
| `Restart` (button.restart) | `entity_category: diagnostic` only |

The text_sensors are hidden by default because most users won't care about them until something breaks. The signal/uptime sensors are still visible because seeing "uptime: 6 hours" on a device card is useful at a glance.

`★ Reference:` ESPHome's `entity_category` is documented per-entity-type (see e.g. [`sensor:`](https://esphome.io/components/sensor/) docs).

---

## The `packages:` pattern and per-device calibration

Each per-device file ends with:

```yaml
packages:
  base: !include common/s31-base.yaml
```

This pulls in the entire shared template at compile time. ESPHome merges the per-device file's substitutions with the template's references and produces one merged config to compile.

### Why this scales

- **Edit-once template** — change `common/s31-base.yaml` and all devices benefit on next flash.
- **Per-device isolation** — only substitutions vary; no copy-paste of YAML.
- **Failure mode is loud** — if you forget to add a substitution to a per-device file, the compile fails with "substitution not found" rather than silently using a wrong default.

### The substitution preprocessor

ESPHome's substitution mechanism is **text replacement at compile time**, not runtime variables. `${voltage_cal}` in `common/s31-base.yaml` is replaced with the literal string `0.9866` (or whatever) before the YAML is even parsed. This is why the substitution values are quoted strings (`"0.9866"`) rather than numbers — they're text being inlined.

`★ Reference:` see [ESPHome `packages:`](https://esphome.io/components/packages) and [ESPHome `substitutions:`](https://esphome.io/components/substitutions).

### Why per-device calibration is essential

Across the four S31 chips in `examples/my-deployment/`, the calibration values span:

| Channel | Range | Spread |
|---|---|---|
| `voltage_cal` | 0.9944 → 1.0059 | 1.15% |
| `current_cal` | 0.9848 → 0.9961 | 1.13% |
| `power_cal` | **0.9662 → 0.9864** | **2.02%** |

If you applied the home-theater outlet's `power_cal: 0.9731` to all four:

- home-theater: perfect (it's its own calibration)
- office: power would be calibrated by 0.9731 instead of 0.9864 → over-correction by 1.4%
- misc: power would be calibrated by 0.9731 instead of 0.9662 → under-correction by 0.7%
- computer: power would be calibrated by 0.9731 instead of 0.9852 → over-correction by 1.2%

Three of four would be **less accurate than uncalibrated**.

This is the empirical case for per-device calibration. Each chip's factory-calibration fingerprint is independent random variation within the ±2% spec; the only way to know it is to measure.

Per-device YAML files (driven by substitutions) make capturing this variation cheap. Without `packages:`, you'd be maintaining four nearly-identical YAML files with slight divergence — a maintenance burden.

---

## Wrap-up: why this template instead of the official example

[The official ESPHome S31 example](https://devices.esphome.io/devices/sonoff-s31) is a useful starting point and intentionally minimal. This template adds:

- **Reliability**: `fast_connect`, `power_save_mode: NONE`, `min_auth_mode: WPA2`, `reboot_timeout: 0s` × 2
- **Security**: API encryption, OTA password, fallback AP password, web auth, all via secrets
- **Calibration**: V/I/P multiply filters, voltage clamp + NaN filter
- **Automation primitives**: power-threshold number, "Active" binary sensor, PF lambda gate
- **Diagnostics**: uptime, version, wifi_info, restart button, all categorized
- **UX**: short entity names + friendly_name + area for HA naming convention, `device_class: outlet`
- **Logging**: silenced sensor spam
- **Local fallback**: web UI on port 80 with auth, no firmware upload

Each addition is justified by something in this doc. The trade-off is YAML weight — `common/s31-base.yaml` is ~250 lines vs. ~70 in the official example — appropriate for a deployment with ongoing maintenance needs.

---

## References

Component docs (linked inline above):

- [`esphome:` core component](https://esphome.io/components/esphome.html)
- [`esp8266:` platform](https://esphome.io/components/esp8266)
- [`wifi:`](https://esphome.io/components/wifi)
- [`api:`](https://esphome.io/components/api)
- [`ota:`](https://esphome.io/components/ota)
- [`web_server:`](https://esphome.io/components/web_server)
- [`logger:`](https://esphome.io/components/logger)
- [`uart:`](https://esphome.io/components/uart)
- [`sensor:`](https://esphome.io/components/sensor/) and `cse7766:`
- [`packages:`](https://esphome.io/components/packages)
- [`substitutions:`](https://esphome.io/components/substitutions)

Changelog references:

- [ESPHome 2023.2.0 — `friendly_name` introduction](https://esphome.io/changelog/2023.2.0)

Origin reference:

- [Official Sonoff S31 device page on devices.esphome.io](https://devices.esphome.io/devices/sonoff-s31) — the upstream example this template extends
- [Home Assistant community thread](https://community.home-assistant.io/t/sonoff-s31-outlet-with-esphome-example-yaml/434828/11) — the substitution-and-package pattern that inspired this repo's structure
