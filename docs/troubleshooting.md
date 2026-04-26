# Troubleshooting

Symptom-based fixes for issues encountered when deploying this template across the four S31s in `examples/my-deployment/`. Each entry is **symptom → cause → fix**, with cross-references to the relevant deeper doc.

If you hit something not listed here, check the [ESPHome Builder logs](installation.md#ota-flashing-for-subsequent-updates) first — most issues surface there as compile errors or runtime warnings.

---

## Power Factor stuck at "Unknown" or shows wrong value

**Symptom**: `sensor.<outlet>_power_factor` displays `Unknown` even with a load running, or shows an inaccurate value (e.g., 0.95 for a heater that should read ~1.00).

**Cause**: Chip-level apparent-power vs real-power calibration mismatch. The CSE7766 has independent calibration paths for V, I, real power, and apparent power. Small inter-channel offsets cause one of two failure modes at high resistive load:

- **P > AP** → ESPHome flags `PF > 1` as invalid → publishes NaN → HA renders "Unknown"
- **P < AP** with AP inflated → PF computes valid but wrong (~0.95 for resistive)

This is documented behavior across the four chips in [`examples/my-deployment/`](../examples/my-deployment/) — different chips show different patterns.

**Fix**: Calibrate V/I/P per [`calibration.md`](calibration.md). After calibration, three of the four reference chips populate PF cleanly at ~0.99. The fourth still reads ~0.95 (chip-level apparent-power mismatch can't be fully corrected from YAML without an `apparent_power_cal` substitution, which the template doesn't currently include).

For most use cases (energy monitoring, automation triggering), PF accuracy isn't critical. The relative differences between load types are still distinguishable (resistive ≈ 1, switch-mode ≈ 0.5).

See [`design-rationale.md` § the power-factor lambda gate](design-rationale.md#the-power-factor-lambda-gate) for the lambda guards that handle this.

---

## Build log shows a long `remove_scanf_float_flag` line

**Symptom**: When compiling/flashing via ESPHome Builder, the build log includes a line that wraps for many lines starting with `remove_scanf_float_flag([".pioenvs/...firmware.elf"], [...])` followed by a list of dozens of `.cpp.o` files.

**Cause**: PlatformIO build optimization. ESPHome injects a custom build action that removes the **floating-point version of `scanf()` / `printf()`** from the linked binary. That code is ~5–10 KB of flash that ESPHome doesn't need (it does its own float-to-string conversion).

**Fix**: None needed. This is a benign optimization that helps your firmware fit on the ESP8266's 1 MB program partition. The long file list is just every compiled object file that participates in the relink step — every component you've enabled adds to this list. More components = longer line.

If your build *succeeds* and shows reasonable Flash usage near the end of the log (e.g., `Flash: [======= ] 68.4% (used 715648 bytes from 1044464 bytes)`), you're fine.

---

## WiFi flicker / intermittent "unavailable" in HA

**Symptom**: HA shows the device as unavailable for a few seconds at a time, then back online. Happens repeatedly even when the router and HA are both healthy.

**Cause**: ESPHome's default `power_save_mode: LIGHT` puts the WiFi radio to sleep between AP beacon intervals. The device misses HA's API pings during sleep windows; HA marks it unavailable until the next ping succeeds.

**Fix**: The template already sets `power_save_mode: NONE`. If you're using the template and still seeing flicker:

1. Verify with a `grep "power_save_mode" common/s31-base.yaml` — should be `NONE`, not `LIGHT` or unset.
2. Check `sensor.<outlet>_wifi_signal` — if it's below −70 dBm, the connection is genuinely weak and the flicker is real WiFi range, not power-save.
3. Try `fast_connect: false` temporarily — if a roaming AP is the issue, fast_connect's BSSID lock will make it worse, not better.

See [`design-rationale.md` § network resilience](design-rationale.md#network-resilience) for the detailed reasoning.

---

## Apparent Power doesn't equal V × I

**Symptom**: You calibrate V and I to match KAW within 0.1%, but `sensor.<outlet>_apparent_power` doesn't equal `sensor.voltage × sensor.current`. Off by 1–7%, sometimes higher than V × I, sometimes lower.

**Cause**: This is by design. Two contributing reasons:

1. **ESPHome computes `apparent_power = V_raw × I_raw` at frame-decode time**, before per-sensor filters apply. The calibrated V and I displayed in HA are post-`multiply: ${cal}` values; apparent_power tracks the raw chip values.
2. **Some CSE7766 chips emit apparent_power on a separate calibration path** rather than computing it from V × I. ESPHome's component normally computes AP from V × I, but chip-level inter-channel mismatches can still produce surprising P/AP ratios.

**Fix**: Apparent_power is a diagnostic-quality reading on this template — useful for relative comparisons and PF derivation, not for billing-grade accuracy. If you need calibrated apparent_power, you can add an `apparent_power_cal` substitution and a multiply filter in `common/s31-base.yaml`. The template doesn't include this by default because most use cases don't need it.

See [`design-rationale.md` § filter ordering rules](design-rationale.md#filter-ordering-rules) for filter timing, and [`calibration.md` § what you can and can't calibrate](calibration.md#what-you-can-and-cant-calibrate) for the limits.

---

## Web UI returns 401 Unauthorized

**Symptom**: Visiting `http://<device_name>.local/` in a browser, you get an HTTP 401 prompt that won't accept any credentials.

**Cause**: Missing or mismatched `web_username` / `web_password` in `secrets.yaml`. The web server requires both to be set; auth is mandatory in this template.

**Fix**:

1. Confirm `secrets.yaml` (in the same directory as your device YAML) has both keys:
   ```yaml
   web_username: "admin"
   web_password: "your-password"
   ```
2. OTA-flash the device to pick up new secrets.
3. Try the credentials in the browser prompt. The username defaults to `admin` in our example but can be anything as long as the YAML and your input match.

If you've set the secrets and still hit 401: the secrets may have changed since the last flash. ESPHome bakes secret values into the firmware at compile time (substitution), so the device knows the credentials it was *flashed with*. Re-flash to update.

---

## OTA flash fails after rotating `secrets.yaml`

**Symptom**: After changing the `ota_password` value in `secrets.yaml`, OTA flashes via ESPHome Builder fail with an authentication error. You can still flash via serial.

**Cause**: ESPHome Builder caches the OTA password per device on first pairing. The Builder is still trying the *old* password.

**Fix** (two options):

**Option A — Flash via serial once**, which re-syncs the new password:

1. Open the device case (yes, again).
2. Wire USB-to-serial as in [`installation.md`](installation.md#first-flash-procedure-usb-serial).
3. Flash via "Plug into the computer running ESPHome Dashboard" — this prompts for the new OTA password, caches it, and from then on OTA works.

**Option B — Update HA's stored password without serial**:

1. In Home Assistant, go to **Settings → Devices & Services → ESPHome integration → your device → ⋮ menu → Reconfigure**.
2. Enter the new OTA password.
3. OTA flashes should work from this point.

Option B is preferable if it works (no case-opening). Option A is the fallback when HA's reconfigure flow doesn't expose the OTA password field.

---

## Entity stuck at "Unknown" after re-flash

**Symptom**: After successfully flashing new firmware, an entity that previously showed values now shows "Unknown" in HA. Sister entities update normally; only one (or a few) are stuck.

**Cause**: HA caches entity state per integration and may not refresh after major config changes. Particularly common when:

- Adding new entities to the YAML (HA may not register them properly until reload)
- Changing entity names (HA tries to match old names to new states and fails)
- The first publish of an entity after boot returns an invalid value (e.g., the chip emits NaN before it's warmed up), and HA treats Unknown as sticky until a non-Unknown value arrives

**Fix**:

1. **Settings → Devices & Services → ESPHome integration → ⋮ → Reload** — usually shakes loose stale state.
2. If reload doesn't help, restart Home Assistant: **Settings → System → Restart**.
3. If still stuck, verify the entity is publishing in the device's live ESPHome Builder logs. If the device isn't publishing, the issue is on the device, not HA.

---

## Voltage drift during calibration test

**Symptom**: During a calibration session, voltage readings (on both KAW and S31) drift between sets — e.g., set 1 shows 119.9 V, set 3 shows 118.4 V. Other sets within the same session don't agree closely.

**Cause**: Another load on the same circuit cycled during the test (refrigerator compressor kicked on, HVAC fan started, electric kettle elsewhere in the house). Both meters track the change, so it's not a measurement error — it's a real voltage sag.

**Fix**:

- The **calibration ratios** (V_kaw/V_s31, I_kaw/I_s31, P_kaw/P_s31) should still be consistent across sets even when absolute values drift. As long as both meters track the change identically, the ratios are valid.
- If the *ratios* themselves disagree across sets by more than ~0.5%, the load itself isn't steady. Repeat the test, ideally on a less-loaded circuit (a basement workshop outlet on its own breaker is ideal).
- Take **5 sets instead of 3** if you want to discard outliers. Drop the highest and lowest, average the middle three.

The `outlet-computer` example in [`examples/my-deployment/calibration-data.csv`](../examples/my-deployment/calibration-data.csv) demonstrates this — set 3 caught a brief voltage sag, but the V/I/P *ratios* stayed stable across all three sets, so the calibration is still trustworthy.

---

## `min_auth_mode` warning during compile

**Symptom**: ESPHome Builder log shows:
```
WARNING The minimum WiFi authentication mode (wifi -> min_auth_mode) is not set.
This controls the weakest encryption your device will accept when connecting to WiFi.
Currently defaults to WPA (less secure), but will change to WPA2 (more secure) in 2026.6.0.
```

**Cause**: ESPHome forward-compatibility warning — the default for `wifi.min_auth_mode` is changing in a future version. The template should have this set explicitly already.

**Fix**: The template already sets `min_auth_mode: WPA2`. If you're using the template unmodified and still see this warning:

1. Verify with `grep "min_auth_mode" common/s31-base.yaml` — should be `WPA2`.
2. Check that the per-device file includes the template via `packages:` — if you forgot the include, the device YAML is missing the wifi block entirely.
3. If your router *only* supports WPA (rare), set `min_auth_mode: WPA` in the per-device substitutions and override. (Routers without WPA2 support are quite old and worth replacing for general security reasons.)

---

## Compile error: "substitution X not found"

**Symptom**: ESPHome Builder compile fails with a message like `Substitution voltage_cal not found in config`.

**Cause**: The shared template references substitutions that the per-device file didn't supply. This happens when a per-device file is missing one of the eight required substitutions.

**Fix**: Verify the per-device YAML's `substitutions:` block contains all eight expected keys:

```yaml
substitutions:
  device_name: ...
  friendly_name: ...
  area: ...
  voltage_cal: ...
  current_cal: ...
  power_cal: ...
  threshold_default: ...
  restore_mode: ...
```

The `outlet-template.yaml.example` in this repo's root has the canonical list. Diff against it if you're missing one.

---

## Relay clicks audibly on every OTA flash

**Symptom**: When you OTA-flash a device, you hear the relay click during boot — sometimes once, sometimes twice. Anything plugged in briefly loses power.

**Cause**: `early_pin_init` is not set to `false`, so ESPHome drives GPIO outputs to default state during early boot before state-restore runs.

**Fix**: The template already sets `early_pin_init: false` in the `esp8266:` block. If you're hearing clicks on flashes anyway:

1. Verify with `grep "early_pin_init" common/s31-base.yaml` — should be `false`.
2. Recompile and re-flash; sometimes the setting takes a flash cycle to actually engage.
3. If clicks persist, this may be a hardware quirk of your particular S31 unit (some relays click on coil energization regardless of pin state). Compare to a different S31 to isolate.

See [`design-rationale.md` § boot and state-restore behavior](design-rationale.md#boot-and-state-restore-behavior).

---

## References

- [ESPHome Builder documentation](https://esphome.io/guides/getting_started_hassio)
- [ESPHome `cse7766:` component](https://esphome.io/components/sensor/cse7766) — power-meter integration
- [ESPHome `wifi:` component](https://esphome.io/components/wifi) — WiFi-related troubleshooting
- [ESPHome `ota:` component](https://esphome.io/components/ota) — OTA flash details
