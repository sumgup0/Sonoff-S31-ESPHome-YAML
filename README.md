# sonoff-s31-esphome

ESPHome configuration for **Sonoff S31 smart outlets**, with a shared template, a generic per-device starter, a worked example using real measurements, and design notes.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![ESPHome 2026.4+](https://img.shields.io/badge/ESPHome-2026.4%2B-blue.svg)](https://esphome.io/)
[![Hardware: Sonoff S31](https://img.shields.io/badge/Hardware-Sonoff%20S31-green.svg)](https://devices.esphome.io/devices/sonoff-s31)

This repo is for anyone running one or more **Sonoff S31** outlets on **ESPHome + Home Assistant** who wants:

- A shared template (`common/s31-base.yaml`) covering reliability, security, calibration, and HA integration, with inline comments explaining each section.
- A pattern (ESPHome `packages: !include`) for managing many similar devices with per-device customization.
- A worked example showing four-outlet calibration data measured against a reference meter, including the chip-level variance observed across them.
- Documentation explaining the reasoning behind each design choice.

Built on top of the [official ESPHome S31 example](https://devices.esphome.io/devices/sonoff-s31) and inspired by the [Home Assistant community thread](https://community.home-assistant.io/t/sonoff-s31-outlet-with-esphome-example-yaml/434828/11) that applied the substitution-and-package pattern to S31s.

---

## Quick start

For someone with one or more S31s and an existing Home Assistant + ESPHome Builder setup:

1. **Clone this repo** into your ESPHome configuration directory (typically `/config/esphome/` if you're using the HA add-on):

   ```bash
   cd /config/esphome
   git clone https://github.com/<your-account>/sonoff-s31-esphome.git
   # Or download as ZIP and extract.
   ```

2. **Set up secrets**. Copy and edit:

   ```bash
   cp sonoff-s31-esphome/secrets.yaml.example secrets.yaml
   ```

   Fill in real values for `wifi_ssid`, `wifi_password`, `fallback_password`, `ota_password`, `web_username`, `web_password`. Generate the `api_encryption_key` with the inline PowerShell snippet, or with `openssl rand -base64 32` if you have OpenSSL.

3. **Create a per-device file** for each S31. Copy the starter:

   ```bash
   cp sonoff-s31-esphome/outlet-template.yaml.example outlet-kitchen.yaml
   ```

   Edit the substitutions block: pick `device_name`, `friendly_name`, `area`, and `restore_mode` for the outlet. Leave the calibration values at `"1.00"` initially.

4. **First flash via USB serial** (the S31 doesn't support tuya-convert or known wireless replacements of stock firmware). See [`docs/installation.md`](docs/installation.md) for the case-opening, pinout, and ESPHome Builder workflow. **Don't skip the safety warning** about not connecting USB serial and mains simultaneously.

5. **Calibrate** (optional but recommended) using a Kill-A-Watt and a steady resistive load. Update the per-device file's `voltage_cal`, `current_cal`, `power_cal` substitutions and OTA-flash. See [`docs/calibration.md`](docs/calibration.md).

After step 4, the device shows up in HA with all sensors and the relay control. After step 5, the four outlets in `examples/my-deployment/` settled within ~0.2% of the reference meter; your results may vary by chip.

---

## Repo tour

```
sonoff-s31-esphome/
├── README.md                              ← you are here
├── LICENSE                                ← MIT
├── .gitignore                             ← excludes secrets.yaml + build dirs
├── secrets.yaml.example                   ← seven placeholder secrets
├── outlet-template.yaml.example           ← starter for new outlets
├── common/
│   └── s31-base.yaml                      ← shared ESPHome template (~250 lines)
├── examples/
│   └── my-deployment/                     ← four-outlet example using real measurements
│       ├── outlet-home-theater.yaml       ← V_cal 1.00,    I_cal 0.9866, P_cal 0.9731
│       ├── outlet-office.yaml             ← V_cal 1.0059, I_cal 0.9961, P_cal 0.9864
│       ├── outlet-misc.yaml               ← V_cal 0.9944, I_cal 0.9848, P_cal 0.9662
│       ├── outlet-computer.yaml           ← V_cal 1.0045, I_cal 0.9961, P_cal 0.9852
│       └── calibration-data.csv           ← 16 rows of KAW + S31 measurements
└── docs/
    ├── installation.md                    ← hardware + flashing + Builder orientation
    ├── calibration.md                     ← KAW methodology + math + worked example
    ├── design-rationale.md                ← reasoning behind each YAML choice
    └── troubleshooting.md                 ← symptoms + fixes
```

---

## What's included in the template

`common/s31-base.yaml` provides:

- **Power monitoring** via the CSE7766 chip — voltage, current, real power, apparent power, power factor, energy counter, daily energy
- **Per-device calibration** via three multiply filters on V/I/P (driven by substitutions)
- **An "Active" binary sensor** with a user-tunable power threshold — fires when power exceeds the threshold
- **Diagnostic entities** — uptime, ESPHome version, IP/SSID/BSSID/MAC, restart button — categorized as diagnostic and disabled-by-default in HA
- **Local web UI on port 80** with HTTP basic auth (monitoring only — no firmware upload from web)
- **Encrypted HA ↔ device API** via Noise protocol, with a required OTA password
- **WiFi resilience** — fast_connect, no power-save sleep, WPA2-or-better, AP fallback for recovery, no auto-reboot on transient HA outages
- **Boot-click prevention** — relay shouldn't briefly toggle on every reboot/OTA
- **Per-device boot behavior** via `restore_mode` substitution (ALWAYS_ON, ALWAYS_OFF, RESTORE_DEFAULT_OFF, RESTORE_DEFAULT_ON)
- **HA naming convention** — `friendly_name` + short entity names, `area:` for auto-grouping, `device_class: outlet`

For the reasoning behind each of these, see [`docs/design-rationale.md`](docs/design-rationale.md).

---

## Hardware

The Sonoff S31 is a smart power outlet with:

- **ESP8266EX** microcontroller, 4 MB flash (board profile: `esp12e`)
- **CSE7766** power-meter IC (4800 baud 8E1 UART → ESP RX line)
- Mechanical relay, audible click on switch
- Pushbutton on GPIO0 (also used to enter flash mode at boot)
- Blue status LED on GPIO13

Doesn't support tuya-convert or other known wireless flashing methods. **First flash requires opening the case and using a USB-to-serial adapter.** Subsequent flashes are wireless via OTA.

See [`docs/installation.md`](docs/installation.md) for the full first-flash procedure.

---

## A note on ESPHome Builder

The Home Assistant add-on commonly referred to as "ESPHome Dashboard" was renamed to **"ESPHome Device Builder"** (often shortened to "ESPHome Builder"). They are the same tool. Older tutorials and forum posts still use "Dashboard," and the CLI command remains `esphome dashboard` for backwards compatibility.

If you've installed the **"ESPHome Builder"** add-on in HA, you have everything you need.

---

## References & Acknowledgments

### Primary references

- [Official ESPHome S31 device page](https://devices.esphome.io/devices/sonoff-s31) — the upstream starter YAML this template extends
- [Home Assistant community thread: Sonoff S31 outlet with ESPHome example YAML](https://community.home-assistant.io/t/sonoff-s31-outlet-with-esphome-example-yaml/434828/11) — source of the substitution-and-package pattern that inspired this repo's structure
- [Sonoff S31 disassembly guide on phreakmonkey.com](https://www.phreakmonkey.com/2018/01/sonoff-s31-disassemble-and-flash.html) — the case-opening procedure used in `docs/installation.md`

### ESPHome documentation (linked inline throughout the docs)

**Components used in the template**:
- [`esphome:`](https://esphome.io/components/esphome.html) — device identity (name, friendly_name, area)
- [`esp8266:`](https://esphome.io/components/esp8266) — platform config, early_pin_init, restore_from_flash, framework versioning
- [`wifi:`](https://esphome.io/components/wifi) — connection, AP fallback, reboot_timeout, fast_connect, power_save_mode, min_auth_mode
- [`api:`](https://esphome.io/components/api) — encrypted HA channel
- [`ota:`](https://esphome.io/components/ota) — over-the-air firmware updates
- [`web_server:`](https://esphome.io/components/web_server) — local monitoring UI
- [`captive_portal:`](https://esphome.io/components/captive_portal) — recovery AP behavior
- [`logger:`](https://esphome.io/components/logger) — log levels and per-component overrides
- [`uart:`](https://esphome.io/components/uart) — CSE7766 serial link
- [`sensor:`](https://esphome.io/components/sensor/) and the `cse7766` platform — power-meter integration and filters
- [`binary_sensor:`](https://esphome.io/components/binary_sensor/) — button, status, template "Active"
- [`number:`](https://esphome.io/components/number/) — power threshold input
- [`switch:`](https://esphome.io/components/switch/) — relay control with restore_mode
- [`button:`](https://esphome.io/components/button/) — restart button
- [`text_sensor:`](https://esphome.io/components/text_sensor/) — version + wifi_info
- [`status_led:`](https://esphome.io/components/status_led) — blue LED indicator
- [`time:`](https://esphome.io/components/time/) — SNTP for daily-energy reset

**Configuration patterns**:
- [`packages:`](https://esphome.io/components/packages) — shared template via `!include`
- [`substitutions:`](https://esphome.io/components/substitutions) — per-device variations
- [`!secret`](https://esphome.io/guides/configuration-types) — credentials via secrets.yaml
- [Security best practices](https://esphome.io/guides/security_best_practices) — auth, encryption, secrets hygiene

### ESPHome changelog references

- [2023.2.0 — `friendly_name` introduction](https://esphome.io/changelog/2023.2.0) — the change that lets HA auto-prefix device name to entity names

### Hardware

- **CSE7766** datasheet — Chipsea Technology / 朝陽微電子. ±2% calibration tolerance.
- [Sonoff S31](https://itead.cc/product/sonoff-s31/) — iTead/Sonoff product page

### Tools

- [Home Assistant](https://www.home-assistant.io/) — the smart-home platform this repo targets
- [ESPHome Device Builder](https://esphome.io/guides/getting_started_hassio) — the HA add-on (formerly "ESPHome Dashboard")
- [Kill-A-Watt P3 P4400](https://www.p3international.com/products/p4400.html) — reference meter for calibration
- [Silicone test hooks (example product)](https://www.amazon.com/gp/product/B08M5Z5YFG) — the no-solder approach used during first flash
- [Microsoft .NET `RandomNumberGenerator`](https://learn.microsoft.com/en-us/dotnet/api/system.security.cryptography.randomnumbergenerator) — used in the PowerShell one-liner for generating `api_encryption_key`

### Acknowledgments

Thanks to the participants of the [original Home Assistant community thread](https://community.home-assistant.io/t/sonoff-s31-outlet-with-esphome-example-yaml/434828/11) for working out the substitution-and-package pattern that inspired this repo's structure. The template-split approach lets a single shared file scale to many devices with minimal per-device boilerplate.

This work builds on the [ESPHome project](https://esphome.io/), the YAML-based firmware authoring framework this repo uses.

---

## License

[MIT](LICENSE) — © 2026 sumgup0

---

This project was developed with the assistance of Claude (Anthropic).
