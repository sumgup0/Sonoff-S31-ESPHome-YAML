# Calibration

The CSE7766 power-meter chip inside every Sonoff S31 is **±2% accurate** out of the box (per the chip datasheet). For most casual use, the factory calibration is fine. For energy monitoring, comparing across outlets, or anything you want to act on quantitatively, calibrating against a reference meter brings each outlet to within ~0.2% of the reference (~0.2% being the KAW's own measurement uncertainty).

This doc covers:

1. [Why per-device calibration matters](#why-per-device-calibration-matters)
2. [Reference equipment](#reference-equipment)
3. [What you can and can't calibrate](#what-you-can-and-cant-calibrate)
4. [Self-consumption sanity check](#self-consumption-sanity-check)
5. [Step-by-step procedure](#step-by-step-procedure)
6. [The math](#the-math)
7. [Worked example: outlet-home-theater](#worked-example-outlet-home-theater)
8. [Applying calibration](#applying-calibration)
9. [Verifying post-flash](#verifying-post-flash)
10. [When you can skip calibration](#when-you-can-skip-calibration)
11. [Power-factor behavior post-calibration](#power-factor-behavior-post-calibration)

---

## Why per-device calibration matters

Different chips have different errors. Here's the variance across the four S31s in `examples/my-deployment/` (all measured against the same Kill-A-Watt with the same heater on the same circuit, in immediate succession):

| Outlet | Voltage error | Current error | Power error | Direction |
|---|---|---|---|---|
| home-theater | +0.22% | +1.37% | **+2.76%** | All high |
| office | −0.59% | +0.39% | +1.38% | V low; I/P high |
| **misc** | +0.56% | +1.54% | **+3.50%** | All high (worst chip) |
| computer | −0.45% | +0.39% | +1.50% | V low; I/P high |

**The four chips are all within their ±2% spec on V and I, but the *power* channel ranges from +1.38% to +3.50% — a 2.12-percentage-point spread.** If you applied the home-theater outlet's calibration values to the misc outlet, you'd over-correct it by 0.74% — making it *less* accurate than not calibrating at all.

This is why per-device calibration is needed for accurate cross-outlet comparison: each chip has its own factory-calibration fingerprint, and there's no way to know what it is until you measure.

`★ Why the power channel is the worst:` the CSE7766 calibrates voltage, current, and power through separate signal paths internally. The voltage path is a simple resistor divider; the current path adds a shunt + amplifier; the real-power path adds an integrator on top. More analog stages = more error budget. Across the four chips here, P_cal varied roughly 2× as much as V_cal.

---

## Reference equipment

You need two things:

### 1. A reference meter

A **Kill-A-Watt P3 P4400** (or equivalent inline mains meter) is a common tool:

- Available for ~$25 at hardware stores
- Displays V, A, W, VA, PF, kWh, Hz on a small LCD
- Specced at ±0.2% on voltage, ±0.5% on current/power (well within KAW use cases)
- Plugs into the wall; the device under test plugs into the KAW

Spec page: [P3 International P4400](https://www.p3international.com/products/p4400.html).

Alternatives that work: any clamp-meter + multimeter combo if you're comfortable opening the circuit; commercial sub-meters (more accurate but $$); a bench-grade power analyzer (more than needed for hobbyist use).

### 2. A steady, known load

Calibration accuracy depends on the load being **stable** and **simple** (close to purely resistive). Ranked from best to worst:

| Load | Why | Notes |
|---|---|---|
| **Space heater on lowest setting** ⭐ | High wattage (~750–1500 W) means S31 self-consumption is below noise; resistive element is dead steady once warm | Best choice. Use heat-only mode if available, not fan-only. |
| Electric kettle while heating | ~1500 W, very steady for ~60 s | Short window — must be quick or kettle finishes |
| Soldering iron at temperature | Resistive, steady | Only ~50–100 W — self-consumption is more visible |
| Incandescent light bulb | Resistive, PF ≈ 1.00 | 60–100 W — usable but lower-power than ideal |
| **❌ USB charger** | Switch-mode supply has variable PF and changes load during charge cycle | **Don't use** — produces unreliable calibration data |
| **❌ Anything inductive** (motor, fan-only mode) | Inductive component pulls PF below 1; chip's PF readings get noisy | **Don't use** |

### Safety reminder for the heater

- Use the **lowest** heat setting (typically 750 W). The S31 is rated 15 A / 1800 W; 750 W gives a comfortable safety margin during a 5-minute test.
- Plug the heater **directly** into the S31 (S31 directly into KAW, KAW directly into wall outlet). No extension cords or power strips between.
- Use an outlet you know is healthy and ideally on its own circuit — a sagging supply gives junk readings.

---

## What you can and can't calibrate

The shared template (`common/s31-base.yaml`) exposes three calibration knobs via per-device substitutions:

| Substitution | What it scales | Filter applied |
|---|---|---|
| `voltage_cal` | The voltage sensor | `multiply: ${voltage_cal}` first in the voltage filter chain |
| `current_cal` | The current sensor | `multiply: ${current_cal}` first in the current filter chain |
| `power_cal` | The real-power sensor | `multiply: ${power_cal}` first in the power filter chain |

**What you cannot calibrate from the YAML:**

- **Apparent power** — ESPHome computes `apparent_power = V_raw × I_raw` at frame-decode time, *before* per-sensor filters apply. So calibrated V and I are what HA displays, but apparent_power tracks the raw chip values.
- **Power factor** — emitted directly by the CSE7766 chip in its serial frame. Has its own internal calibration we can't reach. See [PF behavior post-calibration](#power-factor-behavior-post-calibration) for what to expect.

If you really need accurate apparent_power or PF on a specific outlet, you can add a `multiply` filter in `common/s31-base.yaml` for those channels. For most use cases (energy monitoring, automation triggering), V/I/P calibration is sufficient.

---

## Self-consumption sanity check

When you set up the test (KAW plugged into wall, S31 plugged into KAW, **nothing plugged into the S31**), the KAW will display ~0.5 W. That's not noise — that's the S31's own electronics drawing power. The KAW measures everything downstream of it (including the S31's internals); the S31 only measures power flowing through its relay output.

**Don't subtract this offset for high-load tests** — at 750 W, the 0.5 W self-consumption is 0.07% of the signal, well below measurement noise. *Do* be aware of it for low-load calibrations: at 20 W, that 0.5 W is 2.5%, which would corrupt your calibration ratios.

This is why the recommended calibration load is high-wattage. The math is simple: S31 self-consumption ÷ test load = noise floor for KAW-vs-S31 comparison.

---

## Step-by-step procedure

### Setup (one-time per outlet)

1. KAW plugged into a healthy wall outlet.
2. S31 plugged directly into KAW.
3. Heater plugged directly into S31, set to lowest heat setting, **off**.

### Idle baseline

4. With heater off, wait **90 seconds** for the S31's 60-second voltage-throttle window to refresh.
5. Record:
   - `V_kaw_idle` — KAW voltage display
   - `V_s31_idle` — S31 voltage from HA (`sensor.<outlet>_voltage`)

(For idle, you don't record current/power/PF — they're zero or near-zero and provide no calibration info.)

### Loaded test

6. Turn the heater on.
7. Wait **2 minutes** for two things to settle:
   - The heating element to reach steady-state temperature (constant resistance)
   - The S31's filter windows to clear and refill with steady-state samples
8. Record **set 1**:
   - V_kaw, V_s31
   - I_kaw, I_s31
   - P_kaw, P_s31
   - PF_kaw, PF_s31 (record even if S31 shows "Unknown" — note that fact)
9. Wait 60 seconds. Record **set 2** (same eight fields).
10. Wait 60 seconds. Record **set 3** (same eight fields).
11. Turn the heater off.

### Why three sets?

If the three sets disagree by more than ~1% on any quantity, the load isn't steady — usually because something else on the same circuit cycled (refrigerator compressor, HVAC). Repeat the test, ideally on a less-loaded circuit.

If the three sets agree closely, you can confidently average them; that average is what feeds the calibration math.

---

## The math

For each calibrable channel:

```
calibration_ratio = mean(KAW reading) / mean(S31 reading)
```

- If S31 reads **higher** than KAW → ratio is **< 1** (multiplier dampens the S31 output)
- If S31 reads **lower** than KAW → ratio is **> 1** (multiplier amplifies the S31 output)

Compute three ratios:

```
voltage_cal = mean(V_kaw_set1, V_kaw_set2, V_kaw_set3) / mean(V_s31_set1, ...)
current_cal = mean(I_kaw_set1, ...) / mean(I_s31_set1, ...)
power_cal   = mean(P_kaw_set1, ...) / mean(P_s31_set1, ...)
```

Round to **4 decimal places** (more precision than that is below the noise floor).

### Idle voltage cross-check

The voltage ratio computed from the 3 loaded sets should be close to (within ~0.3% of) `V_kaw_idle / V_s31_idle`. If it's not, something's off — your reference voltage drifted between the idle and loaded measurements, or a meter is misbehaving. Investigate before trusting the calibration values.

---

## Worked example: outlet-home-theater

Here's the actual data from the home-theater outlet's calibration session, preserved in `examples/my-deployment/calibration-data.csv`:

| Set | V_kaw | V_s31 | I_kaw | I_s31 | P_kaw | P_s31 |
|---|---|---|---|---|---|---|
| 1 | 119.9 | 120.2 | 5.86 | 5.93 | 701 | 719.6 |
| 2 | 119.9 | 120.0 | 5.84 | 5.93 | 699 | 719.1 |
| 3 | 119.7 | 120.1 | 5.86 | 5.94 | 702 | 721.4 |
| **Mean** | **119.833** | **120.100** | **5.853** | **5.933** | **700.667** | **720.033** |

Compute ratios:

```
voltage_cal = 119.833 / 120.100 = 0.9978       ← within 0.3%, leave at 1.00
current_cal = 5.853   / 5.933   = 0.9866       ← apply (1.37% error)
power_cal   = 700.667 / 720.033 = 0.9731       ← apply (2.76% error)
```

The voltage error (0.22%) is below the noise floor of the KAW itself (±0.2% spec). Calibrating at that level means propagating the KAW's own measurement error into the S31, which doesn't make the S31 *more* accurate, just more in agreement with the KAW. Leave `voltage_cal: "1.00"` for this outlet.

The current error (1.37%) is small but non-noise. Worth correcting — also helps PF stay within the chip's "valid" range post-cal.

The power error (2.76%) is large enough to materially affect kWh totals. Correcting it brings post-cal readings within the KAW's own measurement uncertainty.

The corresponding YAML edit (already in `examples/my-deployment/outlet-home-theater.yaml`):

```yaml
substitutions:
  voltage_cal: "1.00"
  current_cal: "0.9866"
  power_cal:   "0.9731"
```

---

## Filling the CSV

The `examples/my-deployment/calibration-data.csv` file is a worked example showing the format you'd use for tracking your own calibration sessions. Columns:

```
outlet,area,test_date,heater_setting_W,set,V_kaw,V_s31,I_kaw,I_s31,P_kaw,P_s31,PF_kaw,PF_s31,notes
```

- One row per measurement
- `set` = `idle` or `1` / `2` / `3`
- For `idle` rows: only fill V_kaw, V_s31 (other measurement columns blank)
- For loaded rows: fill all 8 measurement columns
- Use `Unknown` literally if the S31's PF reads as such
- Use `notes` for anything weird (voltage sag, brief load fluctuation, etc.)

You can paste filled-in CSV data into your calibration session notes, run the math by hand or with a spreadsheet, and update YAML.

---

## Applying calibration

1. Edit the per-device YAML's `substitutions:` block:

   ```yaml
   substitutions:
     voltage_cal: "<your value>"
     current_cal: "<your value>"
     power_cal:   "<your value>"
   ```

2. Save.
3. OTA-flash the device via ESPHome Builder.
4. Wait **~90 seconds** after the device comes back online (filter windows must refill before HA sees calibrated values).
5. Verify the new readings match KAW within ~0.2% on V/I/P.

---

## Verifying post-flash

With the heater still running at the same load, expect:

| Sensor | Pre-cal | Post-cal | Should approach |
|---|---|---|---|
| Voltage | 120.10 V | 120.10 V (unchanged if cal=1.00) | KAW (119.83) |
| Current | 5.933 A | 5.852 A (5.933 × 0.9866) | KAW (5.853) |
| Power | 720.03 W | 700.66 W (720.03 × 0.9731) | KAW (700.67) |
| Apparent Power | ~712 VA | ~712 VA (unchanged — uncalibrated) | (V_raw × I_raw) |
| Power Factor | NaN/Unknown | populates ~0.997 | (varies — see below) |

If post-cal V/I/P don't match KAW to within ~0.3%, something went wrong:

- The OTA didn't actually take (verify by checking `sensor.<outlet>_uptime` reset to ~0)
- HA cached old entity values (Settings → Devices → ESPHome → Reload)
- The flash itself has a config error (check ESPHome Builder logs)

---

## When you can skip calibration

The calibration ratio for a channel is "below the noise floor" — and not worth applying — when:

- **Voltage error < 1%** — most KAWs are ±0.2%, mains varies ±0.5% naturally; sub-1% S31 errors aren't worth chasing.
- **Current error < 1%** — same logic.
- **Power error < 2%** — within the chip's stated tolerance; below this you're tuning at the same level as the KAW's own uncertainty.

For *casual* use (knowing roughly when something's running, integrating energy for HA's Energy dashboard), even uncalibrated S31s are usable. Calibration takes ~10 minutes per outlet and brings post-cal readings within the reference meter's accuracy. Worth doing if you have a reference; safe to skip if you don't and casual energy estimates are sufficient.

---

## Power-factor behavior post-calibration

Three categories of behavior across the four outlets:

| PF behavior post-cal | Why | Outlets affected (in `examples/my-deployment/`) |
|---|---|---|
| Populates cleanly at ~0.99 for resistive load | Chip's calibration mismatch happened to be in the right direction; calibrating V/I/P brings P/AP just below 1.0 | home-theater, office, computer |
| Populates but reads ~5% low (e.g., 0.95 for resistive) | Chip's apparent-power channel is significantly inflated relative to V × I | misc |
| Stays "Unknown" even at high load | Chip's apparent-power channel is significantly *deflated* relative to V × I; P / AP > 1.0 → invalid → NaN. Less common after calibration but possible. | (none in this deployment) |

For most deployments PF after calibration will be useful enough for load-type discrimination ("resistive heater" PF ≈ 1 vs "switch-mode charger" PF ≈ 0.5). For perfect PF accuracy on the affected outlet, you'd need to add an `apparent_power_cal` substitution (not currently in the template) that scales the apparent_power channel separately.

See [`design-rationale.md`](design-rationale.md) for the deeper explanation of how the chip's internal channels interact and why some calibration mismatches manifest as NaN PF while others manifest as wrong-but-numeric PF.

---

## References

- [Kill-A-Watt P3 P4400 spec page](https://www.p3international.com/products/p4400.html)
- [ESPHome `cse7766:` component](https://esphome.io/components/sensor/cse7766) — the power-meter integration that ingests CSE7766 frames
- [ESPHome `sensor:` filters reference](https://esphome.io/components/sensor/) — `multiply`, `filter_out`, `clamp`, `throttle_average`, `throttle`, `or`, `lambda`
- CSE7766 datasheet (Chipsea Technology / 朝陽微電子) — chip-level accuracy spec
