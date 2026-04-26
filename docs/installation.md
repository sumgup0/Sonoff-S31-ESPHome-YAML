# Installation

This document covers everything from "I just unboxed a Sonoff S31" to "it shows up in Home Assistant with diagnostic entities working." For the *configuration* meaning of each YAML block, see [`design-rationale.md`](design-rationale.md). For calibrating against a reference meter after first deploy, see [`calibration.md`](calibration.md).

> ## ⚠️ ESPHome Dashboard → ESPHome Builder rename
>
> The Home Assistant add-on commonly referenced as "ESPHome Dashboard" was officially renamed to **"ESPHome Device Builder"** (often shortened to **"ESPHome Builder"**). They are **the same tool**.
>
> - **Older tutorials and forum posts** still say "Dashboard." Treat them as synonymous.
> - **The CLI command** is still `esphome dashboard` for backwards compatibility.
> - **The HA add-on icon and sidebar label** show the new name (Device Builder).
>
> If you've installed the "ESPHome Builder" add-on, you have everything you need — there's no separate "Dashboard" to install.
>
> Citation: [ESPHome Getting Started — Home Assistant guide](https://esphome.io/guides/getting_started_hassio).

---

## Hardware overview

The Sonoff S31 is a smart power outlet with the following internals:

| Component | Role | Notes |
|---|---|---|
| **ESP8266** | Microcontroller | Specifically ESP8266EX with 4 MB flash. The board profile this template uses is `esp12e` to match. |
| **CSE7766** | Power-meter IC | Measures voltage, current, real power, apparent power, power factor. Communicates with the ESP via UART (one-way: chip → ESP). |
| **Relay** | Switches mains | Driven by GPIO12. State changes are audible (mechanical click). |
| **Pushbutton** | Manual control + flash-mode trigger | On GPIO0, active-low. Held during power-up to enter the ESP8266 bootloader. |
| **Blue LED** | Status indicator | On GPIO13, active-low (energized = LOW). Used as ESPHome's `status_led`. |

**GPIO map summary:**

```
GPIO0  → Pushbutton (input, pull-up, inverted) — also used to enter flash mode
GPIO12 → Relay coil (output)
GPIO13 → Blue LED (output, inverted)
RX     → CSE7766 serial data (4800 baud, 8E1)
```

### Why you can't tuya-convert the S31

`tuya-convert` is a popular OTA-replacement tool for Tuya-based smart devices that exploits an unsigned-firmware bug to install custom firmware without opening the case. The S31 doesn't ship with Tuya firmware — Sonoff uses their own (eWeLink) implementation that doesn't have the relevant vulnerability. First flash *requires* opening the case and using a USB-to-serial adapter.

Reference: [ESPHome Sonoff S31 device page](https://devices.esphome.io/devices/sonoff-s31).

---

## ⚠️ Safety: Never connect serial AND mains simultaneously

Important rule when flashing the S31:

> **Do NOT have the S31 plugged into a wall outlet while a USB-to-serial adapter is connected to its serial header.**

Why:

- The S31's neutral line is *not* electrically isolated from its low-voltage logic side. The 5 V/3.3 V on the serial header sits at mains-line potential when the device is plugged in.
- Connecting your USB-to-serial adapter (and through it, your laptop's USB ground) to a S31 that's also plugged into mains creates a path between mains and your laptop.

What can fail:

- **Your USB-to-serial adapter** (most common — magic smoke, replacement cost ~$5)
- **The S31** (relay, CSE7766, ESP8266 — bricks the device)
- **Your laptop** (USB controller, motherboard — most expensive failure mode)

The S31 gets its 3.3 V supply *from* the USB-to-serial adapter during flashing. Don't power it any other way.

---

## First flash procedure (USB serial)

### What you need

- USB-to-serial adapter at **3.3 V logic levels** (CP2102, FTDI FT232RL, or similar — *not* a 5 V Arduino)
- **Silicone test hooks** ([example on Amazon](https://www.amazon.com/gp/product/B08M5Z5YFG)) — clip directly onto the through-hole pads, no soldering needed. Strongly recommended over soldering for one-time flashes.
- A larger-head Phillips screwdriver (PH1 or PH2). **Avoid small-head/precision screwdrivers** — the screws on the S31 strip easily without enough torque.
- A computer running **Chrome or Microsoft Edge** (other browsers, including Firefox, lack the WebSerial API that ESPHome Builder's "Plug into computer" flow needs). Or run ESPHome from the CLI on Linux/macOS without a browser dependency.
- A pry tool (guitar pick, plastic spudger, or fingernail) for the snap-off panel

### Step 1 — Open the S31 case

(Procedure adapted from [Sonoff S31 disassembly guide on phreakmonkey.com](https://www.phreakmonkey.com/2018/01/sonoff-s31-disassemble-and-flash.html), which has photos.)

1. **Pry off the darker square panel** with the button on it. It snaps off — start prying carefully from one side. No screws under it.
2. **Slide out the two corner pieces** on the male-plug side of the device. Slide them *toward* the button side. They're held by friction; once they start moving they come off easily.
3. **Remove the three screws** that were hidden under the corner pieces. Use a PH1 or PH2 Phillips screwdriver — the screws strip easily under a small-head driver. Bear down with enough torque so the bit doesn't slip in the head.
4. **The case separates** with the female-outlet half lifting away. No further prying needed.

### Step 2 — Locate the 6-pin serial header

On the smaller of the two PCBs (the boards are joined in an "L" shape), look for the row of through-hole pads. From top down they're labeled:

```
VCC   RX   TX   N/C   N/C   GND
```

Only four of the six pads are used: `VCC`, `RX`, `TX`, `GND`. The two `N/C` pads aren't connected. There's **no separate IO0 pin on the header** — flash mode is entered via the front-panel button (which is wired to GPIO0).

### Step 3 — Connect to your USB-to-serial adapter

Use silicone test hooks (one per pad). Wiring:

| S31 pad | Adapter pin | Notes |
|---|---|---|
| **VCC** (3.3 V) | 3V3 | Power. **Never use 5 V.** |
| **RX** | TX (adapter's TX) | Crossover — RX listens to what the adapter transmits |
| **TX** | RX (adapter's RX) | Crossover — TX speaks to the adapter's listener |
| **GND** | GND | Common ground |

That's all four wires you need. No IO0 connection — see the next step.

### Step 4 — Enter flash mode (button-hold method)

To put the ESP8266 into bootloader mode:

1. Make sure all four test hooks are clipped securely.
2. **Press and hold the front-panel button** on the S31.
3. **While holding the button**, plug the USB-to-serial adapter into your computer (this is when the S31 gets power for the first time this session).
4. **Continue holding the button** for ~2 seconds after USB power applies, then release.

The ESP8266 sees GPIO0 held LOW at power-up and enters its bootloader, ready to accept new firmware. **Do not plug the S31 into a wall outlet** at any point during flashing — it draws all 3.3 V from the USB adapter.

### Step 5 — Flash with ESPHome Builder

1. Open ESPHome Builder in Home Assistant (or run `esphome run <device.yaml>` from the CLI).
2. For the device's YAML, click **Install**.
3. Choose **"Plug into the computer running ESPHome Dashboard"** (yes, the option still says "Dashboard" — same tool).
4. Pick the COM/serial port your USB-to-serial adapter is on (Windows: `COM3`, `COM4`, etc.; Linux: `/dev/ttyUSB0`; macOS: `/dev/cu.usbserial-*`).
5. Wait for the build + upload to complete.

   **Note on compile time**: on a Raspberry Pi 4 or HA Yellow, the *first* compile for a given device can take **8–10 minutes** before the upload phase begins. During this time the UI may show "Preparing Install" or seem to hang — that's normal; the SDK is being compiled from scratch. Subsequent compiles for the same device are much faster (typically under a minute) because of the build cache.

6. When ESPHome reports success, **disconnect the USB-to-serial adapter from your computer** before doing anything else.

### Step 6 — Test before reassembly

This step is optional but strongly recommended:

1. Disconnect the USB-to-serial adapter, then **plug it back in** to power-cycle the S31 from clean. The new firmware should boot now.
2. Confirm the S31 joins your WiFi (check your router's DHCP table, or browse to `http://<device_name>.local/` in Chrome/Edge — the local web UI should prompt for the credentials you set in `secrets.yaml`).
3. Try an **OTA flash** with a trivial change (e.g., bump a comment) before closing the case for good. If OTA is broken, you want to find that out *now* — not after the case is reassembled and the outlet is behind furniture.

After verifying both WiFi connection and a successful OTA, snap the case back together (corner pieces slide back in, button panel snaps back on).

---

## OTA flashing for subsequent updates

Once the first flash is in place, every subsequent firmware update is wireless:

1. Open ESPHome Builder → your device → **Install**.
2. Choose **"Wirelessly"**.
3. Wait. The device reboots into the new firmware; HA briefly marks it as unavailable and then back online.

The OTA password (from `secrets.yaml`) is asked once on first OTA per device, then cached by the Builder. If you ever rotate `secrets.yaml`'s `ota_password`, you'll need to update HA's stored password too — see [`troubleshooting.md`](troubleshooting.md).

---

## ESPHome Builder file orientation

In Home Assistant's standard add-on layout:

```
/config/                            ← HA root config
└── esphome/                        ← ESPHome Builder's working directory
    ├── secrets.yaml                ← your real secrets (gitignored)
    ├── outlet-kitchen.yaml         ← per-device file 1
    ├── outlet-office.yaml          ← per-device file 2
    ├── ...                         ← more per-device files
    └── common/
        └── s31-base.yaml           ← shared template
```

**Key paths:**

- `!include common/s31-base.yaml` in a per-device file resolves **relative to that file's directory**. So a file at `/config/esphome/outlet-kitchen.yaml` correctly finds `/config/esphome/common/s31-base.yaml`.
- `!secret <key>` looks for `secrets.yaml` in the **same directory** as the YAML file being compiled.
- **Examples in this repo** (`examples/my-deployment/outlet-*.yaml`) use `!include ../../common/s31-base.yaml` because they live two directories deeper. When you copy them out as your own configuration, the path becomes `!include common/s31-base.yaml` again.

**Editing options:**

- **In-Builder YAML editor** — opens the device file via the Builder UI. Best for quick tweaks. Cannot edit files that aren't device YAMLs (e.g., `common/s31-base.yaml` won't show up as a "device").
- **File Editor add-on** — installs alongside ESPHome Builder, gives you tree-view file editing of `/config/esphome/`. Best for editing `common/s31-base.yaml` and `secrets.yaml`.
- **Samba / SSH** — share `/config/` over your LAN and edit with VS Code or whatever you prefer. Best for substantial editing sessions.

---

## First-adoption verification checklist

After first successful flash and HA adoption, verify these in Home Assistant:

| Check | Where | Expected |
|---|---|---|
| Device appears in Devices list | Settings → Devices & Services → ESPHome | Visible with `friendly_name` from your YAML |
| Area assigned correctly | Same page | Area matches what you set in `area:` substitution |
| Diagnostic entities populating | Device page → Diagnostic section | `Uptime` increments, `WiFi Signal` shows dBm, `Status` is "On" |
| Power readings flowing | Sensors section | `Voltage`, `Current`, `Power`, `Apparent Power` all populate within ~90 s |
| Energy counter ticking | Same | `Energy` and `Daily Energy` increase when a load is plugged in |
| Relay responds | Controls section | Toggling the `Relay` switch produces an audible click and changes the load state |
| Power Factor behavior | Sensors section | May show "Unknown" or a numeric value depending on chip — see [`troubleshooting.md`](troubleshooting.md) |
| Web UI accessible | `http://<device_name>.local/` (Chrome/Edge) | HTTP basic auth prompts; entering `web_username` / `web_password` opens monitoring page |

If any of these fail, see [`troubleshooting.md`](troubleshooting.md) for known symptoms.

---

## After successful adoption: calibration

The CSE7766 chip has a **±2% factory calibration tolerance**. Across the four outlets in `examples/my-deployment/`, real-world errors ranged from −0.6% to +3.5% across V/I/P channels. For accurate energy monitoring, calibration is recommended.

See [`calibration.md`](calibration.md) for the procedure (Kill-A-Watt + space heater + 3 measurement sets per outlet).

---

## References

- [Sonoff S31 disassembly guide on phreakmonkey.com](https://www.phreakmonkey.com/2018/01/sonoff-s31-disassemble-and-flash.html) — disassembly procedure with photos, source for this doc's case-opening steps
- [ESPHome Sonoff S31 device page](https://devices.esphome.io/devices/sonoff-s31)
- [ESPHome Getting Started — Home Assistant add-on](https://esphome.io/guides/getting_started_hassio)
- [ESPHome Getting Started — Command line](https://esphome.io/guides/getting_started_command_line) — alternate flash workflow without the HA add-on
- [ESPHome `esp8266:` component](https://esphome.io/components/esp8266) — `early_pin_init`, `restore_from_flash`, framework versioning
- [ESPHome `cse7766:` component](https://esphome.io/components/sensor/cse7766) — power-meter chip integration
- [Silicone test hooks (example product)](https://www.amazon.com/gp/product/B08M5Z5YFG) — the no-solder approach used here
