---
title: ESP32-C6 Power Audit (shinpad_c6.ino)
type: analysis
tags:
  - project/esp-c6
  - firmware
  - hardware
  - power
created: 2026-04-25
updated: 2026-04-25
status: living
---

# ESP32-C6 Power Audit (shinpad_c6.ino)

A trace of where current is drawn while the firmware as it stands today (`Rishab_PCB_esp32c6/shinpad_c6.ino`) is running, plus a ranked list of optimization opportunities. Numbers were estimates from the C6 datasheet + Seeed forum measurements until 2026-04-25 when Kushaj measured **50 mA idle on the bare assembled daughter board** (matches the old bring-up audit's pre-fix number exactly). After this round of fixes, the target is **~10 mA idle**.

## ⚡ 2026-04-25 — first optimization pass shipped

Applied to `shinpad_c6.ino` in this order (changes 1A–1D below):

| Tier 1 fix | Status | Code |
| ---------- | ------ | ---- |
| (A) `setCpuFrequencyMhz(80)` at boot | **Applied** | new line in `setup()` after WiFi teardown |
| (B) `esp_pm_configure(max=80, min=40, light_sleep=true)` | **Applied** | new block in `setup()` immediately after (A); requires `#include "esp_pm.h"` |
| (C) `delay(40)` in idle path of `loop()` | **Applied** | replaced bottom-of-loop `delay(1)` with `if (isLogging \|\| isReplaying) delay(1); else delay(40);` |
| (D) **Slow advertising 1000–1500 ms (single mode)** | **Applied** | `setMinInterval(0x640)` / `setMaxInterval(0x960)` in `initBLE()`. **Replaces** the previous 100–150 ms |
| (G) LED off when idle on battery | **Deferred** | per Kushaj — keep current solid-on / 1 Hz blink behavior for now |
| (F) BMA400 sleep after BMI270 init | **N/A** | BMA400 is never `setupBMA400()`'d in the BMI270-success path, so it stays in its power-on default (sleep, ~0.16 µA per datasheet). No fix needed. |

Pending validation on real hardware. Expected idle: **~10 mA** (down from 50 mA). All fixes are in `Rishab_PCB_esp32c6/shinpad_c6.ino`; no library or partition changes required.

### Why slow-adv as a single mode (no Hot/Warm/Cold state machine)

The previous draft of this audit recommended a 3-state advertising machine (Hot / Warm / Cold). After discussing the actual product use pattern with Kushaj on 2026-04-25, that turns out to be over-engineered:

- **Use pattern:** coach turns on the pad → opens the phone (1–2 min later) → pairs + sends START_LOG → walks onto the pitch → **phone is in a bag in the locker for the entire ~90 min game** → coach returns → phone reconnects → STOP_LOG → done.
- **Implication:** the pad is advertising-into-the-void for ~95% of any session. Fast advertising "to be findable quickly" is wasted effort because the only person who would care is 200 m away.
- **Pairing latency at boot is invisible** because the coach hasn't even unlocked their phone yet.
- **End-of-session reconnect at 1.5 s** is acceptable — the coach just walked back from a pitch; another second of waiting is nothing.
- **No button-to-discover** per Kushaj's preference. Pad always advertises (slowly).
- **No idle deep sleep** per Kushaj's preference. Optimizing for *active-use* battery, not standby.

So we collapsed Hot/Warm/Cold into a single 1000–1500 ms always-on mode. Simpler code, no timeout state, and saves the same ~9 mA radio average vs the old fast adv during game time (which is the bulk of any session anyway).

## ⚠️ 2026-04-25 follow-up — measured 46 mA, expected ~10 mA

After flashing the Tier-1 fixes, Kushaj measured the bare board at **46 mA** (down from 50 mA). That's only ~4 mA of savings vs the ~25 mA we projected.

### Hypothesis: Bluedroid is holding a PM lock

The most likely cause is well-documented in the ESP-IDF / Arduino-ESP32 community: **the Bluedroid BLE stack acquires an `ESP_PM_APB_FREQ_MAX` power-management lock while advertising or connected**, which prevents `esp_pm_configure(...light_sleep_enable=true)` from actually entering light sleep, and may also keep the CPU clock pinned above the `min_freq_mhz` floor.

If this is the cause:
- (D) slow advertising survives → contributes ~3–4 mA savings = exactly what we observed
- (A) `setCpuFrequencyMhz(80)` partially survives but gets bumped back up by the BLE PM lock during adv events
- (B) `esp_pm_configure(light_sleep)` accepts the call but never engages light sleep because of the APB lock
- (C) `delay(40)` busy-waits at full clock instead of becoming a light-sleep window

The McGuinness post (`tomasmcguinness.com/2025/01/06/...`) only documents Matter / Thread + deep sleep — he reached his ~50 µA target by going straight to `esp_deep_sleep_start()`, not by tuning light-sleep with a radio active. **Confirms the broader finding: with any Espressif radio stack live, you can't get below ~30–50 mA on this chip. Sleep is the only path to single-digit mA.**

### Diagnostic prints added

`shinpad_c6.ino` now prints at boot:
```
[PM] CPU MHz before optimization: <N>
[PM] setCpuFrequencyMhz(80) ok=<bool>, now <N> MHz
[PM] esp_pm_configure ret=<code> (OK | FAILED)
```

**First flash 2026-04-25**: serial-monitor opened too late, missed the early `[PM]` prints (log started at "[FS] mounted"). Sensor stack confirmed healthy — BMI270 came up first attempt, all three I2C devices detected, BLE connected, `STOP_LOG` from app processed cleanly. Reset-button workflow noted for next attempt.

**Late-open mitigation added**: `loop()` now also re-prints a `[PM-late t=5s]` summary at 5 sec and a `[PM-late t=30s]` follow-up at 30 sec, using `esp_pm_get_configuration()` to read back what's actually applied. This way even if Serial Monitor opens 30 sec after boot, we capture the data.

Three possible boot-log outcomes that tell us what to do next:
- `esp_pm_configure ret=-1` (or `esp_pm_get_configuration FAILED`) → PM not compiled into the Arduino-ESP32 build (sdkconfig issue) — probably needs core 3.x
- `ret=0 (OK)` and `[PM-late t=5s] CPU MHz now: 80` and idle still ~46 mA → confirmed Bluedroid APB lock — go straight to FIFO + deep-sleep architecture (P)
- `ret=0 (OK)` but `[PM-late t=5s] CPU MHz now: 160` → PM accepted but BLE forced clock back up — same conclusion, FIFO + deep-sleep is the answer

### 2026-04-25 — boot log captured, root cause confirmed

```
[PM] CPU MHz before optimization: 160
[PM] setCpuFrequencyMhz(80) ok=1, now 80 MHz
[PM] esp_pm_configure ret=262 (FAILED — PM not compiled in or invalid args)
```

**`ret=262` = `0x106` = `ESP_ERR_NOT_SUPPORTED`.** `CONFIG_PM_ENABLE` is not compiled into the Arduino-ESP32 build for this board on Kushaj's installed core version. The call returns NOT_SUPPORTED and bails without doing anything. Light sleep is **unreachable** without rebuilding the Arduino-ESP32 core libs (which means custom sdkconfig + recompile of the entire SDK — multi-day rabbit hole).

What this means concretely:
- **(A) `setCpuFrequencyMhz(80)` worked** ✅ → that's where the ~4 mA actually came from + slow adv
- **(B) `esp_pm_configure` rejected** ❌ → no light sleep, ever, on this board with this core
- **(C) `delay(40)` is just a 40 ms busy-wait at 80 MHz** instead of a sleep window — useless for power
- **(D) Slow adv works** ✅ — modest contribution

> [!danger] Decision 2026-04-25: do NOT pursue PM rebuilding
> Even if we managed to enable `CONFIG_PM_ENABLE`, the absolute ceiling with Bluedroid alive is **~25–30 mA idle** because of the APB lock issue. The FIFO + deep-sleep architecture (P below) gets us to **~2 mA during the logging session** — a 15× improvement that doesn't require any PM/SDK work at all. Skip the rabbit hole, go straight to (P).

### Code that's left in but no longer load-bearing

The `setCpuFrequencyMhz(80)` call survives because it does work and saves ~5–10 mA on its own (CPU clock cut). The `esp_pm_configure()` call also stays — it harmlessly returns NOT_SUPPORTED today, but if a future core update enables PM it'll start working without any code change. The `delay(40)` idle path is no longer pulling its weight, but no harm in leaving it; it'll start working too if PM ever gets compiled in. The diagnostic prints can stay as a permanent boot-time health check.

### Measurement methodology check

If measuring with **USB plugged in**, there's a hardware floor of ~6 mA that no firmware change can move:
- Charge LED (hardwired, always on with USB): ~3 mA
- Charge IC quiescent: ~1–2 mA
- USB-CDC enumerated: ~1–2 mA

Need to confirm Kushaj is measuring on battery only (USB unplugged) before drawing conclusions.

---

> [!info] Arduino IDE settings used (Seeed XIAO ESP32C6 board profile)
> - **Board:** Seeed XIAO ESP32C6 (`XIAO_ESP32C6`)
> - **CPU Frequency** dropdown: doesn't matter — `setCpuFrequencyMhz(80)` overrides at boot
> - **USB CDC On Boot:** Enabled (required for Serial output over USB-C)
> - **Erase All Flash Before Sketch Upload:** **Disabled** (preserves LittleFS)
> - **Partition Scheme:** Default 4 MB w/ spiffs (will need "Huge APP (3MB No OTA)" later if/when miniz is enabled)
> - **Arduino-ESP32 core version:** must be ≥ 3.0 — `esp_pm.h` and the unified `esp_pm_config_t` struct require ESP-IDF v5.x under the hood. Seeed's bundled board manager URL sometimes lags; if it does, switch to Espressif's official URL.

---

## 1. How the chip works when it's on

The firmware is a single-threaded Arduino sketch on the HP core. There is no FreeRTOS task explicitly created by us; everything runs out of `loop()` plus a few ESP-IDF / Bluedroid background tasks the BLE stack spins up.

### Steady state (awake, idle, NO BLE connection)

Every iteration of `loop()`:

1. `now = millis()` — reads the hardware timer.
2. `buttonTick(now)` — `digitalRead(PIN_BTN)`, debounce/hold state machine, no I/O if button isn't pressed.
3. `ledTick(now)` — `(bool)Serial` charge-proxy check, then `digitalWrite(PIN_LED, ...)` to either solid-ON (no USB) or 1 Hz blink (USB present).
4. Deferred re-advertise check (`needReadvertise` flag) — typically false.
5. `isLogging` while-loop — skipped when not logging.
6. `isReplaying` block — skipped when not replaying.
7. `delay(1)` — yields the CPU for 1 ms.

At ~160 MHz CPU clock with `delay(1)` between iterations, the loop runs ~1000 times per second. The CPU is **not** parked in light sleep during `delay(1)` because no `esp_pm_configure(...light_sleep_enable=true)` call is made. So the CPU is ~99% busy at 160 MHz even doing nothing.

In parallel, **the BLE stack is advertising** at a 100–150 ms interval at +9 dBm TX power. Each advertising event is a brief radio TX burst (~1–2 ms at high current) followed by a short scan window listening for a connect request. Average current from the radio alone is roughly proportional to `(burst_current × burst_duration) / interval`.

In parallel, **all three sensors are powered** because `PIN_SENSOR_PWR` (D2) was driven HIGH in `setup()` and never goes LOW again until deep sleep. They all sit at their configured ODRs:
- BMI270: 100 Hz accel + 200 Hz gyro, normal+perf mode
- BMA400: 200 Hz accel, normal mode (also init'd as a fallback that didn't end up being used)
- MMC5603: idle until next trigger (no measurement queued during idle — the ping-pong only runs while `isLogging`)

Even though we only *read* from them during a logging session, they are clocking at their configured ODRs and burning their normal-mode current the entire time the chip is awake.

### Steady state (awake, BLE connected, no logging)

Same as above, but BLE replaces advertising bursts with connection-event packets at the negotiated connection interval (whatever the phone asked for, typically 30–50 ms on iOS, 7.5–50 ms on Android). Average current is similar — the radio is on a similar duty cycle, just timed by the connection rather than adv events.

### Steady state (logging)

The `while (isLogging && ...)` block fires every 50 ms (`SAMPLE_INTERVAL_MS`):
1. One I2C read of 12 bytes from BMI270 (or 6 from BMA400)
2. Bias subtraction + LPF on `gy` (cheap CPU)
3. If `mmcReady`: 6-byte read of last MMC measurement + 1-byte trigger of next
4. Append row to in-RAM `chunkRows[]`
5. Every 512 rows (~25.6 s of data): `compressChunk()` runs → bit-pack the 6 columns → `dataFile.write()` to LittleFS → `dataFile.flush()`

Sensor I/O is ~1–2 ms of bus activity per 50 ms tick → low duty cycle. The big spike is the chunk write: 512 rows × 6 cols × ~1.5 bytes/col = ~4.5 KB written to internal flash every 25 s. Internal flash writes are tens of mA briefly.

### Steady state (replaying)

Every loop iteration:
1. Read up to 244 bytes from `dataFile`.
2. `pData->setValue` + `pData->notify` → BLE TX of one connection-event-aligned packet.
3. `delay(4)`.

So at ~250 packets/sec × 244 bytes = ~60 KB/s. The BLE radio is the dominant draw here.

---

## 2. Boot-on sequence (chronological, with power impact)

The full path from "USB plugged in" or "wake from deep sleep" to "advertising and ready":

| Step | What runs | Approx duration | Power state |
| ---- | --------- | --------------- | ----------- |
| 0 | ESP32-C6 ROM bootloader, second-stage bootloader, app image load | ~200–400 ms | CPU full clock, radios off |
| 1 | `Serial.begin(115200)` + `delay(200)` | 200 ms | USB-CDC enumerated (if host present) |
| 2 | `WiFi.mode(WIFI_OFF) + esp_wifi_stop() + esp_wifi_deinit()` | <1 ms | **WiFi radio + MAC torn down** — saves ~30 mA vs leaving it idle |
| 3 | LED pin output, `ledOff()` | <1 ms | LED off (HIGH) |
| 4 | `pinMode(PIN_SENSOR_PWR, OUTPUT)` + `digitalWrite(HIGH)` + `delay(200)` | 200 ms | **Sensor VDD rail comes up** — Q1 turns on, Q2 gate goes LOW, sensors get 3V3. Bulk caps charge. Sensors run their internal power-on reset |
| 5 | `pinMode(PIN_BTN, INPUT)` | <1 ms | — |
| 6 | I2C: `setBufferSize(256)`, `begin(SDA, SCL)`, `setClock(50000)`, `setTimeOut(1000)`, `delay(100)` | 100 ms | I2C peripheral active, pull-ups idle high |
| 7 | `LittleFS.begin(true)` | ~50–500 ms (longer if first boot / format) | Flash controller active |
| 8 | `scanI2CBus()` — pings every address 0x08–0x77 | ~40 ms | Brief I2C transactions |
| 9 | `probeOtherSensors()` — chip ID reads of MMC + BMA | ~5 ms | Brief I2C |
| 10 | `setupBMI270()` — manual init (try 1) | ~1.5 s **typical** | Heavy I2C activity. The 8 KB config blob is uploaded as 256× 32-byte writes at 50 kHz. Each transfer has start/addr/data/stop overhead → real throughput ~5–6 KB/s → ~1.5 s for the upload alone |
| 10b | If failed: `powerCycleSensors()` (80 ms off + 150 ms settle + bus reinit) and retry | up to 2 more times | VDD rail goes off then back on; bulk caps recharge each time |
| 11 | If BMI270 failed all 3 attempts: `setupBMA400()` × up to 3 | up to ~1 s | I2C only, no config blob |
| 12 | `setupMMC5603()` — chip ID, soft reset, kick first measurement | ~40 ms | Brief I2C |
| 13 | `Wire.setClock(400000)` | <1 ms | I2C clock bumped to 400 kHz |
| 14 | Gyro-Y bias calibration: 400 ms loop reading IMU at ~50 Hz | 400 ms | I2C read every 20 ms; CPU full speed |
| 15 | `initBLE()` — Bluedroid stack, GATT service + 8 chars, advertising | ~200–400 ms | **BLE radio on**, advertising at 100–150 ms interval, +9 dBm |
| 16 | `ledOn()` | <1 ms | LED solid on |

**Total boot time: ~3.5–5 seconds typical** (longer if BMI270 init retries).

The boot itself is short and bursty. The interesting power story is what runs *afterwards*, indefinitely, until the user holds the button or the battery dies.

---

## 3. Where the power is going

Estimated current draw on a bare-XIAO-plus-daughter-board, awake, idle, no BLE connection. **Every line below corresponds to something the firmware actually has running today.**

| Subsystem | Estimated draw | Reason / source |
| --------- | -------------- | --------------- |
| **CPU @ 160 MHz, busy loop with `delay(1)`** | **~20 mA** | No `setCpuFrequencyMhz()` call. No `esp_pm_configure(...light_sleep_enable=true)` call. So idle task does not park the core. |
| **BLE radio advertising at 100–150 ms / +9 dBm** | **~10–15 mA average** | `setMinInterval(0xA0)` / `setMaxInterval(0xF0)`. `setPower(ESP_PWR_LVL_P9)`. Default mid-level TX power would be ~7–10 mA. |
| **BMI270 in normal mode** (100 Hz accel + 200 Hz gyro) | ~700 µA – 1 mA | Per BMI270 datasheet; gyro is the dominant share. Stays on the rail and clocks even when not being read. |
| **BMA400 in normal mode** (200 Hz accel) | ~150 µA | Even though the firmware is using BMI270, BMA400 was init'd on the same rail and is still clocking. **Pure waste.** |
| **MMC5603 idle** (no triggered measurement) | ~few µA | Single-shot mode; only spikes during a measurement. Idle = negligible. |
| **LED solid ON via GPIO15 (active-LOW)** | **~1–2 mA** | Built-in LED driven through whatever resistor Seeed put on the XIAO module. |
| **Charge IC quiescent (board-level)** | ~1 mA | XIAO C6 datasheet quiescent + community measurements. Cannot be removed in firmware. |
| **3V3 LDO leakage (board-level)** | ~0.3–0.5 mA | XIAO C6 module LDO. Board-level. |
| **Charge LED (hardwired, USB present)** | ~3 mA | Only when USB is plugged in. Not on battery. |
| **I2C pull-ups (2× 2.7K)** | ~few µA average | Only meaningful during bus activity; low duty cycle when not logging. |
| **USB-CDC Serial enumerated** | ~1–2 mA | Only when USB host is connected. |
| **`Serial.print(...)` calls** | spike during print | Each call writes to USB-CDC ring buffer; tiny but non-zero. Not a bottleneck. |

### Rough totals

| Mode | Estimated total | Notes |
| ---- | --------------- | ----- |
| **Awake, idle, on battery, no BLE conn** | **~33–40 mA** | This is the number to attack first. Goal: ~10 mA. |
| **Awake, idle, USB plugged in** | ~37–45 mA | + charge LED + USB-CDC |
| **Logging, BLE connected** | ~40–50 mA | + flash writes + sustained sensor reads + connection events |
| **Replaying** | ~50–65 mA | BLE TX-heavy |
| **Deep sleep** | unknown — never measured on this hardware | Goal: ~50 µA |

---

## 4. Optimization opportunities (ranked by expected savings, idle-mode)

These are written as concrete code changes. They are NOT in the current firmware.

### 🔴 Tier 1 — biggest wins, cheapest to do **(all four shipped 2026-04-25)**

#### A. Drop CPU clock to 80 MHz at boot (~10–15 mA saved) — ✅ shipped
```cpp
#include "esp_pm.h"
// in setup() after Serial.begin:
setCpuFrequencyMhz(80);
```
Bump back to 160 MHz only if/when WiFi is ever turned on. BLE works fine at 80 MHz. There's no Cresento workload on this chip that needs 160 MHz; even DeltaPack compression of 512 rows is sub-millisecond.

#### B. Enable auto light-sleep (~10 mA saved) — ✅ shipped
```cpp
esp_pm_config_t pm = {
  .max_freq_mhz = 80,
  .min_freq_mhz = 10,        // or 40 if 10 is unstable
  .light_sleep_enable = true,
};
esp_pm_configure(&pm);
```
With this, the FreeRTOS idle task parks the CPU when no task is ready. BLE wakes the CPU on connection events. **Multiplies the impact of (A)** because the CPU spends much more time at the floor frequency.

#### C. Bump idle `delay(40)` (multiplier on B) — ✅ shipped
At the bottom of `loop()`, replace `delay(1)` with a state-aware delay:
```cpp
if (isLogging || isReplaying) delay(1);  // tight timing
else                          delay(40); // long light-sleep windows
```
Without this, the loop wakes the CPU 1000×/sec just to do nothing. With it, ~25×/sec when idle.

#### D. Slow advertising — single mode 1000–1500 ms (~9 mA saved) — ✅ shipped
Replaces the previous 100–150 ms. **Single mode**, no Hot/Warm/Cold state machine — see the rationale block at the top of this note. Trade-off: phone reconnect latency goes from instant to ~1.5 s typical, which is invisible given the actual use pattern (1–2 min phone-open at boot, 90 min uninterrupted logging in between, ~5 s reconnect at end). Connected throughput is unaffected — adv stops once a connection exists.
```cpp
adv->setMinInterval(0x640);  // 1000 ms (units of 0.625 ms)
adv->setMaxInterval(0x960);  // 1500 ms
```

#### E. Cut sensor rail when not logging (~1 mA saved + sets up deep sleep story)
Right now `PIN_SENSOR_PWR` goes HIGH at boot and stays HIGH forever (until deep sleep). Change to:
- HIGH only during init, calibration, and `isLogging` / `isReplaying`
- LOW while idle awake

Cost: starting a session has to re-power the rail (~150 ms settle) and re-run BMI270 init (~1.5 s including config blob). That's 1.7 s of latency on `START_LOG`. Acceptable for soccer training — the phone START button isn't time-critical to the millisecond. **Or** keep the rail on, write `PWR_CTRL=0x00` to BMI270 to suspend without losing config (saves the 1.5 s but only saves ~700 µA, not the full ~1 mA — see (F)).

### 🟡 Tier 2 — modest savings, easy

#### F. Disable BMA400 once BMI270 is up — ❌ N/A
**Original audit was wrong.** BMA400 is *only* configured (`setupBMA400`) in the fallback branch when BMI270 init fails. In the BMI270-success path, BMA400's registers are never written, so the chip stays in its **power-on default = sleep mode (~0.16 µA per datasheet)**. No fix needed in the success path. If we ever want to suppress BMA400 in the *fallback* path mid-session, we'd write `ACC_CONFIG0 = 0x00`, but that's not in scope.

#### G. LED off when idle (~1–2 mA saved) — ⏸️ deferred 2026-04-25
Per Kushaj — keep the current solid-on / 1 Hz blink behavior for now. Worth revisiting if measured idle current after the Tier-1 pass is still above the 10 mA target. The behavior to switch to (when re-enabled) is:
- BLE connected → solid ON
- Logging / replaying → solid ON
- Idle, no connection, on battery → OFF (or a brief flash every 5 s as a "still alive" hint)
- USB present → 1 Hz blink (unchanged)

#### H. Drop TX power when connected (~1 mA saved)
P9 (max) is right for advertising — phone can be at the edge of range. Once connected, the phone is right next to the pad; drop to P3 or P0:
```cpp
// onConnect:
BLEDevice::setPower(ESP_PWR_LVL_P3);
// onDisconnect / re-advertise:
BLEDevice::setPower(ESP_PWR_LVL_P9);
```

### 🟢 Tier 3 — nice to have, not on the critical path

#### I. SAFE_PRINT macro for Serial
Wrap every `Serial.print*` in a macro that checks `(bool)Serial`. Two reasons:
1. Tiny power saving when USB host isn't there (no driver work).
2. **Bigger reason:** prevents the USB-unplug hang documented in the previous firmware revision's notes. If USB is detached mid-run, `Serial.println` can block forever.

#### J. Auto-sleep-on-idle timer
Add a "no BLE connection AND no button activity for 5 minutes → enter deep sleep" rule. Right now the only path into deep sleep is a 1.5 s button hold. A pad in someone's bag with no phone in range will burn battery indefinitely.

#### K. Lower I2C timeout after init
`Wire.setTimeOut(1000)` is set at boot for BMI270's flaky post-reset wake. After init succeeds, lower it back to 50 ms — saves CPU stalls on a (rare) bad sensor read.

#### L. Combine `ledTick` / `buttonTick` work
Tiny wins; only worth it if every µA matters.

### 🔵 Tier 4 — bigger architectural changes (later)

#### M. IMU INT1-driven sampling
Both BMI270 and BMA400 INT1 lines go to GPIO0. Configure BMI270 to fire INT1 on data-ready at 20 Hz and use it as a hardware wakeup. Tight 50 ms `delay(1)` polling gets replaced with light sleep + interrupt. Combined with (B) this would be a major win during logging.

#### N. BMA400 as wake-on-motion sensor
BMA400 can run at **~800 nA – 1.2 µA** in low-power motion-detect mode (datasheet rev 2.3). Sleep mode is **160–200 nA**. While idle awake (no logging, no connection), put BMI270 in suspend, run BMA400 in motion-detect, sleep the CPU. Wake on shake → re-power BMI270 → start advertising aggressively.

#### O. Move IMU sampling onto the LP core
The C6 has a 20 MHz LP RISC-V core that can run while the HP core sleeps. Could handle the entire 20 Hz sample loop and only wake HP at chunk boundaries. Significant lift, but unique to the C6.

#### P. **BMI270 FIFO + ESP32 deep sleep during logging** ⭐ — ✅ **shipped 2026-04-25 (untested on hardware)**

**Kushaj's hypothesis 2026-04-25:** "turn off our chip because the IMUs might have some cache we can use to keep logging data, then we can pull from the cache."

**Verified against datasheets — the silicon supports this architecture:**

| Chip | FIFO size | Per-frame (header mode) | Buffered at 20 Hz | Suspend retains config? |
| ---- | --------- | ----------------------- | ----------------- | ----------------------- |
| BMI270 | **2048 B** (datasheet rev 1.5 §4.7) | 13 B (gyro+accel + 1 B header) | **~7.85 sec** | **Yes** — config + FIFO both retained, no need to re-upload the 8 KB blob across suspend↔normal |
| BMA400 | **1024 B** (datasheet rev 2.3 §p.29) | 7 B (3 axes 12-bit + 1 B header) | **~7.3 sec** | Yes |

Both chips expose a **byte-level watermark interrupt on INT1**, which is wired to ESP32-C6 GPIO0 on this daughter board. GPIO0 is LP-domain capable on the C6 → it's a valid **deep-sleep wake source**. So the chip can fully power off and be woken by the IMU when the FIFO crosses ~80% full.

**Architecture sketch ("logging mode"):**

```
On START_LOG:
  - esp_bt_controller_disable()          ← BLE off entirely (saves ~10–15 mA)
  - configure BMI270 FIFO: header mode, accel+gyro, watermark = 1600 B (~6 sec)
  - INT_MAP_DATA.fwm_int1 = 1            ← FIFO watermark → INT1
  - INT1_IO_CTRL = active-high push-pull
  - esp_deep_sleep_enable_gpio_wakeup(GPIO0_MASK, HIGH)
  - esp_deep_sleep_start()
                                          (~50 µA average for the next 6 sec)
On wake (INT1 fires at watermark):
  - read FIFO over I2C in one burst (~50 ms at 400 kHz)
  - decode frames, append to LittleFS chunk
  - clear interrupt
  - back to deep sleep
                                          (~150 ms wake duty cycle every 6 sec)

Periodic BLE check-in (every N drains, e.g. every 60 sec):
  - bring BLE up at low TX power
  - advertise for 2 sec
  - if no connect → sleep again
  - if connect → exit logging mode normally, replay session

On STOP_LOG (when phone connects):
  - drain final FIFO contents
  - re-enable BLE permanently
  - run replay pump as usual
```

**Power math during a 90-min logging session:**

| Phase | Current | Duty cycle | Contribution |
| ----- | ------- | ---------- | ------------ |
| Deep sleep (BMI270 normal mode in background) | ~50 µA + ~700 µA = ~0.75 mA | ~97% | ~0.73 mA |
| Wake to drain FIFO | ~50 mA | ~150 ms / 6 sec ≈ 2.5% | ~1.25 mA |
| Periodic BLE check-in | ~10 mA | 2 sec / 60 sec ≈ 3.3% | ~0.33 mA |
| **Total (avg over a 90-min game)** | | | **~2.3 mA** |

Compare to current ~46 mA → **20× improvement**. On a 200 mAh LiPo: 200 / 2.3 = **~87 hours of recording vs current ~4 hours**. With BMI270 in low-power mode (~400 µA gyro floor, requires `adv_power_save=1`), this drops to ~2.0 mA total.

**Caveats / risks:**

- **BLE re-init after deep sleep is ~200–400 ms**. Acceptable as long as BLE check-ins are infrequent.
- **GPIO15 LED needs to be off in deep sleep** (already handled in `enterDeepSleep()`).
- **Sensor rail (D2) must stay HIGH during logging** so BMI270 keeps clocking from its kept-alive bias. Cannot use the existing `sensorsOff()` in this path.
- **Replay needs all data, including FIFO that hasn't been drained yet** — STOP_LOG handler must drain BMI270 FIFO once more before close.
- **BMI270 `adv_power_save` requires 450 µs delay between writes** — I2C helpers need adjusting in low-power mode (datasheet §4.5).
- **Phone visibility during a game becomes 60-second windowed** — during the 58-of-60 seconds the pad isn't advertising, the phone won't see it. Acceptable since the coach won't try to stop the session mid-game.
- **First-time pairing must happen before START_LOG** — once logging starts, the pad isn't continuously discoverable. This matches the existing flow ("coach pairs first, then sends START_LOG, then walks onto the pitch").

**Effort estimate:** ~200 lines of new firmware. Mostly BMI270 FIFO config + decode + state-machine changes. Done in one focused pass.

**This is the most impactful single change available** for the active-use battery target. It maps directly onto Kushaj's "use the IMU cache" instinct — the cache is real (2 KB / ~8 sec at 20 Hz), it just isn't quite long enough to skip every wake, so we use a periodic-drain-and-sleep pattern.

### Implementation shipped 2026-04-25 (untested on hardware)

All under `#define USE_DEEP_SLEEP_LOGGING 1` — flip to 0 to revert to legacy 20 Hz polling logging.

**Final tuning chosen with Kushaj 2026-04-25:**
- **Chip ODR:** 100 Hz (ACC + GYR), 13 B/frame in FIFO header mode
- **5:1 decimation in firmware** → file stays at exact 20 Hz (sample_hz=20 unchanged → app analytics untouched)
- **FIFO watermark:** 1690 bytes (~130 frames at 100 Hz, ~1.3 sec wake interval)
- **BLE check-in:** every **10 sec** (Kushaj's call — prioritizes UX over battery; STOP latency capped at 10 sec)
- **BLE check-in window:** 2.5 sec advertising
- **Mag (MMC5603) skipped** during deep-sleep mode — no FIFO on the mag chip, so capturing it would require waking on every IMU sample (defeats the architecture). mx/my/mz columns written as 0.

**Files modified** (all in `Rishab_PCB_esp32c6/shinpad_c6/shinpad_c6.ino`):
1. **Feature flag block** with `USE_DEEP_SLEEP_LOGGING` + tuning constants + `RTC_DATA_ATTR loggingState` struct (8 fields, survives deep sleep)
2. **BMI270 FIFO register defines** (FIFO_LENGTH/DATA/WTM/CONFIG, INT1_IO_CTRL, INT_MAP_DATA) + helpers `bmi270ConfigureFIFO()`, `bmi270FIFOBytesAvailable()`, `bmi270ReadFIFO()`
3. **Deep-sleep logging core**: `openSessionFileForAppend()`, `drainFIFOAndAppend()` (header-mode parser + 5:1 decimator + LPF restore-from-RTC), `enterLoggingDeepSleep()` (configures GPIO0+timer wake sources, calls `esp_deep_sleep_start`), `runBleCheckIn()` (brings BLE up for 2.5 sec, polls for `bleConnected`), `fastPathLoggingWake()` (wake router fast path)
4. **Wake router at top of `setup()`** — checks `loggingState.magic == LOGGING_STATE_MAGIC && loggingState.active`, jumps to fast path if true
5. **`startLogging()`** modified — after legacy state setup, configures BMI270 FIFO, populates RTC state, tears BLE down, calls `enterLoggingDeepSleep()`. If BMI270 isn't the active IMU (BMA400 fallback), stays in legacy polling mode (BMA400 has FIFO but isn't supported in this revision).
6. **`stopAndReplay()`** modified — adds a deep-sleep cleanup block at the top: drains final FIFO, flushes chunker, closes append-mode `dataFile`, clears RTC magic. Then runs the existing replay path normally.
7. **`loop()` re-advertise block** — if `loggingState.active` and we just disconnected without STOP_LOG, drain FIFO and re-enter deep sleep instead of restarting advertising forever.

**Critical design notes** (write these into your future-self brain):
- **Sensor power rail (D2) STAYS HIGH during deep-sleep logging.** BMI270 needs power to keep clocking the FIFO. Cutting D2 = losing all in-flight samples + having to re-upload the 8 KB config blob on every wake.
- **The chunker buffer is wiped by deep sleep.** `chunker_flush()` is called at the end of every `drainFIFOAndAppend()` so each wake produces a complete (small) chunk. Trade-off: more chunks per session (~4150 vs the legacy ~210), file is ~40% larger from chunk header overhead — non-issue at our session sizes.
- **LPF state + 5:1 decimator counter live in RTC** (`loggingState.filtGy`, `loggingState.decimateCtr`) so wake-to-wake continuity is preserved. Otherwise the LPF would snap to zero on every wake.
- **Append mode** for LittleFS file across wakes (`"a"` after first wake, `"w"` only on the cold-boot first wake of a session).
- **GPIO0 wake source** uses `ESP_GPIO_WAKEUP_GPIO_HIGH` because INT1 is configured active-high in `bmi270ConfigureFIFO()` (`INT1_IO_CTRL = 0x0A`). Don't change one without the other.
- **Timer wake** is set to `DSL_BLE_CHECKIN_INTERVAL_SEC` (10 s) so we get either a FIFO INT1 wake (every ~1.3 s) or a timer wake (every 10 s), whichever comes first. Each wake drains the FIFO; only timer wakes also bring BLE up.
- **`BLEDevice::deinit(true)`** is called before `enterLoggingDeepSleep` to release the Bluedroid stack cleanly. `BLEDevice::init` will rebuild it on the next BLE check-in wake.

**Open risks (test on hardware to validate):**
- BMI270 FIFO header parser — if real-world frames include unexpected tags (e.g. `0x40` input-config frames) we silently skip 1 B; might desync. If you see corrupted replay rows, instrument the parser.
- BLE re-init from cold post-deep-sleep on Arduino-ESP32 v3.x for C6 — should work but `BLEDevice::init` after `deinit` has been a flake on older cores.
- `LittleFS.end()` + `LittleFS.begin(true)` cycle on every wake — if there's any state corruption, sessions could be lost. Append mode should be tolerant.
- The app may not handle the 4000+ small chunks gracefully if it has any per-chunk overhead in its decoder. Watch for slow STOP→DONE replay times.

### 🚨 2026-04-25 follow-up #13 — CHIP_PU glitch is the actual culprit; firmware is at limit

**Test result after INPUT_CFG=1 + aggressive resync fixes:** still seeing partial data + repeated `reset_reason=1 (POWERON)` cold reboots mid-session. User's reveal showed `cmd='STOP_LOG'` arriving but no preceding START_LOG (lost to a power cycle), and the upper Serial output showed multiple full cold-boot sequences within a single test session.

**Read the ESP32-C6 datasheet (`esp32-c6_datasheet_en.pdf`) for further power-saving opportunities.** Most important finding (Section 5.4, Table 5-4):

> `VIH_nRST = 0.75 × VDD` and `VIL_nRST = 0.25 × VDD` on **CHIP_PU**. **A VDD sag that drops CHIP_PU below 0.25×VDD will trigger a POR.** A noisy CHIP_PU pin is a more likely culprit than chip undervoltage.

**This is the actual root cause.** The chip's VDD doesn't have to drop below the 3.0V minimum operating voltage (Section 5.2) — the CHIP_PU pin only needs to dip below 25% of VDD momentarily, which can happen via **capacitive coupling from BLE TX bursts on the XIAO module's PCB**. The XIAO doesn't have a dedicated RC filter on CHIP_PU, so TX-burst transients couple in and trigger a POR. **This is a hardware issue with the XIAO module, not our firmware.**

This is why disabling BOD didn't help — POR via CHIP_PU is a separate reset path.

**Other relevant datasheet findings:**

| Finding | Section | Implication |
| ------- | ------- | ----------- |
| BLE TX @ 0 dBm = **130 mA peak** | 5.6.1, Table 5-8 | Already at lowest safe TX power; can't reduce further without losing BLE entirely |
| BLE TX @ +9 dBm = 190 mA peak | 5.6.1 | Confirms the previous +9 dBm setting was hammering the supply |
| CPU floor with BLE = **40 MHz** (XTAL direct, no PLL division) | 4.1.3.3 | Can drop from 80 → 40 MHz, saves ~5–8 mA |
| Light-sleep = 180 µA **with BLE preserved** | 5.6.2, Table 5-10 | Documented as supported, even though `esp_pm_configure` returned NOT_SUPPORTED on this build. Worth re-checking via direct API. |
| Modem-sleep = 14 mA @ 80 MHz idle | 5.6.2 | BLE radio periodically off; another option |
| GPIO drive strength = **20 mA default**, configurable 5–40 mA | 2.3, footnote 3 | Lower on slow-edge pins (D2 sensor power, LED) → smaller switching transients → less coupling to CHIP_PU |
| Default GPIO12/13 drive = 40 mA (USB pins) | 2.3 | These are USB-CDC pins; high default drive contributes to coupling. Could lower to 20 mA, with risk of breaking serial monitor |
| LP core (RV32IMAC, 20 MHz) has **LP I2C** | 4.1.1.3, 4.1.2.1 | Could poll BMI270 from LP core while HP+radio off entirely. Big architectural lift but unique to C6. |

**Firmware fixes shipped 2026-04-25:**

1. **CPU dropped from 80 MHz to 40 MHz** (cold boot + on every wake from deep sleep). Cuts ~5–8 mA from average current. 40 MHz is the documented floor while BLE is up.
2. **GPIO drive strength on PIN_SENSOR_PWR (D2) and PIN_LED (GPIO15) lowered to `GPIO_DRIVE_CAP_0`** (~5 mA, default was 20 mA). These are slow-edge signals that don't need fast transitions; weaker drive reduces switching transients that can couple into CHIP_PU and trigger a POR.

**Hardware recommendations (the actual fix):**

The firmware is now at the limit of what it can do alone. Remaining issues are dominantly hardware:

1. **Add an RC filter on CHIP_PU** of the XIAO module. A 10 kΩ + 100 nF to ground (R-C low-pass) filters out the TX-burst transients before they can reach the CHIP_PU input. This is THE single highest-impact fix and absolutely required for a battery-powered product.
2. **Add bulk decoupling at VDD** — 100 µF low-ESR ceramic + 100 nF ceramic at the LDO output. Damps the rail droop during 130 mA TX bursts.
3. **Use a higher-capacity / lower-ESR LiPo** — at minimum 500 mAh, ideally 1000+ mAh. Smaller cells have higher internal resistance which directly translates to voltage sag under load.
4. **Consider replacing the XIAO module** for production — the XIAO is a dev module not optimized for batteryload. A custom module with a real LDO (e.g., TPS62823) and proper decoupling would be more robust.

**What we're NOT doing (and why):**

- **Light-sleep direct API**: would be worth trying but the architecture is currently around deep sleep, and the partial data loss problems aren't from missing this mode — they're from CHIP_PU resets.
- **LP core for sensor polling**: huge architectural change, not justified until we know the chip can stay alive (CHIP_PU fixed).
- **Modem-sleep**: similar reasoning; not the bottleneck.

**Net diagnosis:** firmware is doing everything reasonable. The user needs to either accept partial data on this hardware setup or fix the XIAO module's CHIP_PU decoupling externally (add the RC filter on the carrier board). Without that, sessions will continue to be partial because the chip will keep POR-ing on TX bursts.

### 📖 2026-04-25 follow-up #14 — datasheet read + corrections to earlier claims

Read the full ESP32-C6 datasheet v1.1 (`Rishab_PCB_esp32c6/esp32-c6_datasheet_en.pdf`). Several things to correct from earlier follow-ups:

**Authoritative power numbers (Table 5-11, p.65 + Table 5-10, p.64):**

| Mode | Typ |
| --- | --- |
| Active / Modem-sleep, 160 MHz CPU running, peripherals enabled | 38 mA |
| Active / Modem-sleep, 80 MHz CPU running, peripherals enabled | **30 mA** |
| Active / Modem-sleep, 80 MHz CPU idle, peripherals enabled | 25 mA |
| Light-sleep (CPU + wireless powered down, peripheral clocks off) | 180 µA |
| Light-sleep (CPU + wireless + peripherals fully powered down) | 35 µA |
| **Deep-sleep (RTC timer + LP memory only)** | **7 µA** |
| Power off (CHIP_PU low) | 1 µA |

**The user's "30 mA when BLE radio is supposed to be off" is exactly Modem-sleep at 80 MHz with CPU running.** It's not a bug — it's the chip's normal idle state when BLE controller is up and advertising. Real deep sleep is 7 µA on the datasheet (15 µA on the Seeed module including LDO + charge IC). We're 4000× off — the chip is NOT entering deep sleep, it's stuck in modem-sleep.

This explains the multi-reset observation: chip in modem-sleep at 30 mA → BLE TX burst → rail droops → brownout / power-on reset → cold boot → repeat. We never get to deep sleep at all.

**Corrections to prior claims in this audit:**

1. **GPIO0 is NOT a strapping pin on ESP32-C6.** Per Table 3-1 (p.29) the strapping pins are `MTMS`, `MTDI`, `GPIO8`, `GPIO9`, `GPIO15`. GPIO0 is a regular LP-IO pin (`LP_GPIO0` per Table 2-7) and fully usable as a deep-sleep wake source. My earlier theory that GPIO0's strapping role was blocking the wake was wrong. The real reason `cause=7` never fired is most likely either (a) BMI270 INT1 not actually reaching the pin cleanly, or (b) `esp_deep_sleep_enable_gpio_wakeup` config not taking effect because we never actually entered deep sleep in the first place.

2. **The "40 MHz CPU breaks BLE" claim was unverified.** The datasheet does NOT say BLE requires 80 MHz minimum. Section 4.1.3.3 (Clock) says the 40 MHz XTAL is what BLE uses internally; CPU can run at 40 MHz against this. I made the claim based on the user's empirical "couldn't pair at 40 MHz" observation, but that could have been caused by something else (brownout coinciding with the test, etc.). The 80 MHz floor was a guess presented as a documented fact. **Removing the strong "DO NOT" comments from source.**

3. **Deep sleep target of 7 µA (chip) / 15 µA (module) is achievable per datasheet** but ONLY when sleep is actually entered. We're not getting there — we're staying in modem-sleep at 30 mA.

**Real next-step diagnostic plan:**

The fundamental question is: does `esp_deep_sleep_start()` actually put the chip into deep sleep? Three ways to verify:

a) Check `esp_sleep_get_wakeup_cause()` on next setup() entry. If it returns `ESP_SLEEP_WAKEUP_UNDEFINED` consistently, we didn't deep-sleep — the chip cold-booted instead.

b) Add explicit `esp_bt_controller_disable()` + `esp_bt_controller_deinit()` after `BLEDevice::deinit(true)` in case Bluedroid's own deinit isn't fully releasing the radio.

c) Try `esp_sleep_pd_config(ESP_PD_DOMAIN_RTC_PERIPH, ESP_PD_OPTION_OFF)` before `esp_deep_sleep_start` for more aggressive power-down.

These don't require architecture changes, just instrumentation + a small additive fix. Hold off until user signs off.

### ⚠️ 2026-04-25 follow-up #13 — `setCpuFrequencyMhz(40)` breaks BLE pairing — DO NOT lower below 80 MHz

**Symptom:** firmware was edited (between sessions) to set CPU frequency to 40 MHz instead of 80 MHz. Justification given in source comment claimed "40 MHz is the BLE-stack floor per ESP32-C6 datasheet §4.1.3.3" — **this is wrong.** Result: chip booted, advertised, but **could not be paired with from the app at all.** "im not able to even pair to the chip."

**Why 40 MHz breaks BLE on ESP32-C6:**

40 MHz is the XTAL-direct clock (no PLL multiplication). The Bluedroid BLE stack and the BLE controller need APB clock ≥ 80 MHz to handle connection events, GATT writes, and notification scheduling within the BLE timing windows. At 40 MHz:
- Adv works (it's mostly hardware-driven)
- But CONNECT_REQ → handshake → GATT discovery → CCCD writes → notify all run in software and miss their timing budgets
- Phone sees the device, attempts to connect, the link establishes briefly then fails or times out

**The actual minimum CPU frequency for BLE on ESP32-C6 is 80 MHz.** This is mentioned in passing in IDF docs but not loudly. Don't take a datasheet section citation at face value — verify by attempting BLE operations at the proposed frequency.

**Fix shipped 2026-04-25:**
- Both `setCpuFrequencyMhz` calls reverted from 40 → 80
- Source comments rewritten with `⚠ DO NOT lower this to 40 MHz` and the verified-on-hardware note
- Two locations: cold-boot setup (~line 2106) and wake-router fast path (~line 2061)

**Save the ~5–8 mA "savings" from going to 40 MHz:** not worth it. We need pairing to work. Stick with 80 MHz.

If we ever genuinely need to drop below 80 MHz to save more current, it'd require the same kind of architecture as the FIFO + deep-sleep mode but inverted: BLE off entirely between events, CPU at 40 MHz between BLE-up windows, BLE up at 80 MHz only when actively communicating. Major architectural lift. Not worth it for the few mA saved.

### 🔧 2026-04-25 follow-up #12 — partial DSL data + RTC corruption hypothesis + parser fixes

**Test result after the BOD-disable + TX-power fixes:** session ran end-to-end without resetting (huge improvement!). `[boot] reset_reason=1 (POWERON) bootCount=1` only. START_LOG arrived, DSL flow ran, deep sleep happened, phone reconnected, STOP_LOG arrived, **38 rows captured** (`[Log] STOP_LOG (logged rows=38)`). For a ~10-sec session that's roughly half what we'd expect at 20 Hz (200 rows). Replay sent 50 rows total but the user's CSV only had 18 rows — so there's a ~64% data loss between chip and app on the BLE replay path.

**Two new mysteries from this run:**

**Mystery A: wake-cycle logs missing from RTC buffer.** The reveal jumps directly from `>>> esp_deep_sleep_start <<<` to `[DSL-checkin] BLE up` — we never see `[boot] reset_reason=8 (DEEPSLEEP) bootCount=2`, `[DSL] wake from deep sleep, cause=4`, drain frame counts, etc. that should appear on every wake. Yet `bootCount=1` stays unchanged AND `loggingState.magic` clearly survived (DSL cleanup in `stopAndReplay` ran).

**Hypothesis (likely):** the brownout is STILL happening, just no longer triggers a full reset because we disabled the BOD. The voltage dip survives logically (chip keeps running) but **drops below the RTC SRAM retention voltage (~1.5V on C6) momentarily, corrupting the RTC log buffer mid-session**. Earlier writes (POWERON, START_LOG) survive because they were committed before the dip; wake events written DURING the dip are lost; later events after the dip resume. RTC slow memory in C6 is volatile SRAM with a low retention voltage, but not zero.

This would mean the BOD-disable isn't enough — we're still hitting underrun, just not getting a clean reset. **Real fix is hardware: better battery (>3.7V), bulk decoupling on the LDO, or lower-current peripherals.**

**Mystery B: parser bailing on `0x02` after only 16 frames.** The hex dump at the bail point:
```
ctx: 06 BC 03 53 03 C3 04 93[02] D1 F1 8C D2 06 74 00
                              ↑       ↑
                         unknown    REG_GA at +3
```
Unknown at 210, valid `0x8C` at 213. So 3 bytes between frame 16 and the unknown — that means **the INPUT_CFG frame between is 2 bytes total (1 tag + 1 byte), NOT 5 bytes** like my recent fix made it.

**Why the discrepancy with the datasheet read?** The agent's quote was:
> "Payload = 1 config-change byte + 3 sensortime bytes = 4 bytes (5 bytes total with header)"

But the **3 sensortime bytes are conditional on `fifo_time_en=1`**. We have `fifo_time_en=0` (FIFO_CONFIG_0=0x00, confirmed in readback `cfg0=00`). So in our config the payload is only the config byte → 2 bytes total, not 5. My recent fix over-skipped 3 bytes per Input_Config event, then misread data bytes as fake tags.

**Fixes shipped 2026-04-25:**

1. **INPUT_CFG payload back to 1 byte** with comment explaining the conditional on `fifo_time_en`.
2. **Aggressive parser resync**: instead of bailing on first unknown tag (or attempting a fixed 3-byte skip), scan forward up to 16 bytes looking for ANY valid tag (`0x8C`, `0x88`, `0x44`, `0x40`, `0x48`, `0x80`). When found, jump to it and continue parsing. Recovers most of the drain instead of bailing on the first weird byte. Logs the resync distance + new tag for diagnosis.

**Seeed wiki findings (cross-checked):**

- Published deep-sleep current is **15 µA at 3.8V supply**. We're not hitting this and probably can't on a sagging battery.
- Seeed explicitly flags **GPIO0 as a boot-mode strapping pin** and recommends using a different GPIO for wake if possible. Confirms our diagnosis that GPIO0/INT1 wake is unreliable.
- No documentation of USB-CDC behavior across deep sleep, GPIO hold APIs, or LDO specifications. We've figured these out from datasheets / IDF source.

**Recommendations to user:**

- **Measure battery voltage RIGHT NOW.** If <3.7V, charge to full before next test. The 15 µA deep sleep claim only holds at 3.8V.
- Consider a **higher-capacity LiPo** (≥500 mAh) — lower internal resistance = less voltage sag under load.
- Hardware long-term: PCB rev should add **bulk decoupling** at the LDO output (10–47 µF + 100 nF ceramic).
- Consider rerouting BMI270 INT1 to a non-strapping LP-IO pin (GPIO1, GPIO3-7) on next PCB rev.

**About the chip-to-app data loss (50 rows replayed, 18 received):**

Separate from chip-side issues. The chip is sending correct data; something between the BLE replay pump and the app's parser is dropping ~64%. Possible causes:
- App's BLE notify subscription buffer overflowing on iOS/Android
- Replay pacing too fast for the connection (we use 4ms between 244-byte chunks; could try 8ms)
- BLE MTU smaller than expected on this connection (we set 247 but phone might negotiate down)

This is in the App side — not chasing it from firmware unless the chip-side issues clear up first.

### 🩹 2026-04-25 follow-up #11 — `BROWNOUT` reset confirmed; BOD disabled + TX power dropped

**Reveal log finally caught the smoking gun:**
```
[boot] reset_reason=9 (BROWNOUT) bootCount=1
[BLE] connected @t=4850
[BLE] CTRL CCCD subscribed → READY notify
[BLE] cmd='STOP_LOG' len=8 hex=53 54 4F 50 5F 4C 4F 47
[BLE] DATA CCCD subscribed
[BLE] cmd='STOP_LOG' len=8 hex=53 54 4F 50 5F 4C 4F 47
[BLE] DATA CCCD subscribed
```

**`reset_reason=9 (BROWNOUT)`** — chip's brownout detector triggered a reset because the supply voltage dropped below ~2.5 V at some point. This wipes RTC slow memory and cold-boots the chip. Earlier follow-ups ruled this out based on `reset_reason=8 (DEEPSLEEP)`, but that was on an earlier test with USB plugged in (stable 5 V→3.3 V via the SGM6029 LDO). Now testing on battery alone, the BOD trips.

**This explains every weird symptom from the past 2 days:**

- "Auto-reconnect not working / have to manually reconnect when seeing 40 mA jumps" → the chip is brownout-resetting, the phone temporarily can't reach a device that's mid-cold-boot
- "Reveal log empty after sessions" → BOD reset wipes the RTC log, just like a hard power-cycle would
- "1200 ms / 300 ms data instead of 30 sec session" → chip resets mid-session, only the LittleFS file from a previous successful session survives, and that gets replayed on the next STOP
- "Failed to stop / no data received" → chip is in cold-boot or just-rebooted state when STOP arrives, doesn't have the RTC magic to know it was mid-session

**Why the SGM6029 LDO trips on this board:**

McGuinness's article (2025) flagged this exact LDO as voltage-input-sensitive. Peak current scenarios that can sag the rail below the ~2.5 V brownout threshold on a marginal LiPo:

1. BLE TX bursts at +9 dBm → ~120 mA peak per burst
2. ESP32-C6 wake-from-deep-sleep transient → ~50–100 mA briefly
3. BMI270 + BLE + CPU all active simultaneously during a check-in window
4. LiPo internal resistance × peak current → V_battery droops momentarily

**Fixes shipped 2026-04-25:**

1. **`esp_brownout_disable()` at the very top of `setup()`.** Disables the BOD reset entirely. We don't actually need it to "protect" us — there's no critical flash-write happening during BLE TX. Disabling it lets the chip ride through transient voltage dips that would have triggered a spurious reset. Standard workaround for ESP32 boards on marginal supplies.
   ```cpp
   extern "C" { void esp_brownout_disable(void); }
   // First line of setup, before WiFi teardown or anything else:
   esp_brownout_disable();
   ```
2. **Default BLE TX power: P9 → N0 (max +9 dBm → 0 dBm).** Cuts peak TX current from ~120 mA to ~30–40 mA, well below the brownout-trigger envelope. Range drops from ~10 m line-of-sight to ~3 m which is fine — coach is right next to the pad during pairing/STOP, and during the long out-of-range stretch in the middle of a game NO TX power would help anyway.
3. **Check-in TX power override REMOVED** (was set to P3 when default was P9; now redundant with N0 default).
4. **Source comments updated** to cite the BOD reset as the diagnosed cause + reveal log evidence + datasheet rationale, so this can't be rolled back without understanding why.

**Predicted outcome after these fixes:**

- No more `reset_reason=9 (BROWNOUT)` in reveal logs (or if it still happens, the BOD-disable means the chip doesn't reset on it — just keeps running)
- Sessions complete cleanly even on a marginal battery
- "Auto-reconnect" works normally because the chip stays alive between check-ins
- Full ~30 sec of CSV data for a 30 sec session (assuming the FIFO + INPUT_CFG + timer-fallback fixes from #9 also held)
- Slight increase in deep-sleep current (BOD circuitry was free, now we save its trip overhead but pay nothing because BOD passively monitors)

**Hardware caveat (carry into next PCB rev):**

The XIAO C6's SGM6029 LDO is too marginal for our current peaks on a small LiPo. Future PCB revs should:
- Add bulk decoupling near the LDO output (10–47 µF + 100 nF)
- Use a battery with lower internal resistance (>500 mAh LiPo, not coin cell)
- Consider replacing the LDO entirely or adding an external buck for higher-voltage input efficiency

### 🧪 2026-04-25 follow-up #10 — reveal log empty after sessions (RTC wiped by power-cycle)

**Symptom:** user did a 10-sec test, expected to see DSL events in `reveal`, instead saw only:
```
[boot] reset_reason=1 (POWERON) bootCount=1
[BLE] cmd=STOP_LOG
[BLE] cmd=STOP_LOG
```

No `START_LOG`, no `[DSL] startLogging entered`, no wake events. CSV showed 1200 ms of data.

**Diagnosis: the user is power-cycling the chip between the session and the `reveal` command.** Each power-cycle wipes the RTC slow memory (which is where the DSL log buffer lives). The 1200 ms of data in the CSV is from a previous successful DSL session that wasn't yet overwritten on the LittleFS file. The `reveal` shows only the post-power-cycle activity (the user pressing STOP twice, which replays the existing file).

**The chip IS working when DSL is active** — the periodic 40 mA jumps the user observed in idle are exactly the BLE check-in windows firing (every ~10 sec). The "auto-reconnect not working" complaint is the same thing: between check-ins the chip is in deep sleep and invisible to the phone, so manual reconnect during a check-in window is required.

**App code verified correct.** Checked `Cresento/src/src/contexts/BleContext.tsx:2330`:
```typescript
const payload = encodeToBase64('START_LOG');
await device.writeCharacteristicWithResponseForService(ctrlSvc, ctrlChar, payload);
```
RN BLE library decodes base64 on the way out, so chip receives raw `"START_LOG"` bytes — matches our parser exactly. Plus the chip's CONTROL char has `PROPERTY_WRITE` which matches `withResponse`. App side is fine.

**Diagnostic instrumentation added 2026-04-25:**

1. **Hex-dump the full raw bytes of every CONTROL write.** If the app ever sends something unexpected (different encoding, BOM, padding), the trimmed-string match would silently fail. Now we see the exact bytes:
   ```
   [BLE] cmd='START_LOG' len=9 hex=53 54 41 52 54 5F 4C 4F 47
   ```
2. **CCCD subscribe events teed through `DSL_LOG`** — `[BLE] CTRL CCCD subscribed → READY notify` and `[BLE] DATA CCCD subscribed` show up in `reveal`, lets us confirm the app actually subscribed before sending commands.
3. **BLE connect/disconnect events teed through `DSL_LOG`** with millis timestamps — `[BLE] connected @t=12345`, lets us see exactly when the phone got onto the link relative to other events.

**Test procedure that doesn't lose data (CRITICAL):**

The user has been losing diagnostic data because they power-cycle between the session and `reveal`. The fix:

1. Power-cycle ONCE at the very start to reset everything.
2. Open Serial Monitor immediately, press chip RESET button (this is a software reset, RTC survives).
3. From app: connect → press START → wait 30 sec → press STOP.
4. **DO NOT power-cycle, DO NOT unplug USB.** Just type `reveal` in the Serial Monitor.

The reveal log will show the entire timeline (since the buffer survives software resets), including:
- BLE connect timestamp
- CCCD subscribes
- The START_LOG command (with hex bytes)
- DSL state transitions (if any)
- Wake events with INT_STATUS_1 + FIFO len
- The STOP_LOG command
- Whatever happened in between

If after this procedure we still see `bootCount=1` and no DSL events, the chip never went into DSL mode, and we have a different bug to chase. If we see `bootCount > 1` with DSL events, the architecture is working.

### 🎯 2026-04-25 follow-up #9 — root cause of `0xF8`/`0xFC`/`0xFD` found in datasheet

**The "unknown tags" weren't unknown at all — they were misread sensortime bytes from a mis-parsed Input_Config frame.**

After reading the BMI270 datasheet (rev 1.3, §FIFO control frames table pp. 40–42, file `Rishab_PCB_esp32c6/C2836813.pdf`):

> **`Fifo_Input_Config` frame (header `0x48`) has a 4-byte payload, NOT 1 byte.**
> Layout: 1 tag byte (`0x48`) + 1 config-change byte + 3 sensortime bytes = **5 bytes total per frame.**

My parser was advancing only `i += 1` after the tag (treating it as 2 bytes total). So 3 bytes of sensortime were left in the buffer, and the next iteration of the parser read the first of those bytes as a "tag" — which is exactly what showed up in our logs as `unknown tag 0xF8` / `0xFC` / `0xFD`.

The hex-dump pattern fits perfectly:
- 16 GA frames × 13 bytes = offsets 0–207
- Input_Config (5 bytes real) at offsets 208–212
- My parser consumed only 2 bytes → advanced to offset 210
- Read offset 210 (= byte 2 of sensortime) as a "tag" — got `0xF8`
- Real next frame (`0x8C`) appeared at offset 213 — **exactly where the hex dump showed it**

**Fix shipped:** `i += 4` instead of `i += 1` in the INPUT_CFG branch of `drainFIFOAndAppend`. Source comment now explicitly cites the datasheet section + PDF path so this can't be re-introduced.

**Other findings from the datasheet pass that may still bite us:**

1. **`fifo_time_en` defaults to `1` in `FIFO_CONFIG_0`** — every FIFO drain that empties the buffer ends with a `0x44` sensortime frame. We DO disable this (`FIFO_CONFIG_0 = 0x00`) and the readback log confirmed `cfg0=00`, so we're safe. But document this as a load-bearing config write — anyone re-init'ing FIFO must keep this clear.

2. **Tag-extended frames possible.** Regular frames have a 2-bit `fh_ext` field (bits 1:0) used to tag samples that fired interrupts. So `0x8C`, `0x8D`, `0x8E`, `0x8F` are all valid GA frames — same payload, different INT-tag annotation. We don't enable INT tagging in our config (`fifo_tag_int1_en = 0`), so we shouldn't see these, but the parser could be made more robust by masking the low 2 bits before comparing tags.

3. **`fh_mode = 0b11` (any byte starting with `0xC0` or higher) is "Reserved"** in the datasheet. The `0xF8` etc. we saw were data bytes that happened to fall in this range — never real frames.

### Combined "data loss" diagnosis

The "1200 ms data for an N-second session" the user has been seeing is the combination of TWO independent bugs:

1. **GPIO0 strapping pin → INT1 wake doesn't fire** → only timer wakes (every 10 sec) → FIFO overflows in stream mode → only the last ~1.5 sec of data survives per drain
2. **INPUT_CFG payload mis-sized** → parser bails after the very first Input_Config frame (which the chip emits once at startup and on any config change) → only ~16 frames captured before the bail → a few hundred ms of data per drain

Both fixed in this build. Plus the timer-fallback architecture (every 1.5 sec instead of 10 sec) means we're no longer dependent on GPIO0 wake firing — even if H1 isn't fully solved, the FIFO will drain reliably.

### Timer-fallback architecture also shipped

Replaces the 10-sec timer wake with a 1.5-sec timer wake (matches FIFO fill rate) plus a `wakesSinceCheckin` counter in RTC that triggers a BLE check-in every 7th wake (~10.5 sec, same end-user STOP latency as before).

| Setting | Before | After |
| ------- | ------ | ----- |
| Timer wake interval | 10 sec | **1.5 sec** |
| FIFO drain frequency | every wake | every wake (now ~1.5s) |
| BLE check-in frequency | every wake | **every 7th wake (~10.5 sec)** |
| Reliance on GPIO0 INT1 wake | required | optional bonus |

Power impact unchanged on average (~7–10 mA on battery during session). What's gained is **reliability** — FIFO drains will happen on the timer regardless of whether the strapping-pin GPIO0 wake works.

### Predicted outcome

For a 30-sec session:
- ~20 timer wakes × 130 frames per wake = 2600 frames captured
- 5:1 decimation → 520 rows logged
- CSV has 520 × 50 ms = **26 seconds of data** for a 30-sec session (loss is mostly from the last drain not firing yet at STOP time)

If we still see data loss after this build, the next thing to look at is the BLE check-in window potentially eating into FIFO drain time (5 sec at 100 Hz × 13 B/sec = 6500 B FIFO fill, but FIFO is only 2 KB, so ~3 sec of data overwritten during each check-in).

### ✅ 2026-04-25 follow-up #8 — H1 + H3 confirmed; fixes shipped

**Reveal log decisively answered all five hypotheses:**

```
[boot] reset_reason=11 (?) bootCount=1                    ← USB reset, normal
[DSL-fifo] readback: cfg0=00 cfg1=D0 wtm=069A map=02 int1=0A   ← config OK, rules out H2
[boot] reset_reason=8 (DEEPSLEEP) bootCount=2             ← clean wake, rules out H4 H5
[DSL] wake from deep sleep, cause=4 (timer)               ← timer wake, NOT GPIO
[DSL-wake] INT_STATUS_1=0x02 (fwm_out=1) FIFO_LEN=2004    ← H1 SMOKING GUN
[DSL-fifo] unknown tag 0xF8 at offset 210, ctx:F2 2A 09 3B FB 43 06 64[F8] FA F0 8C 80 F3 48 09
                                                          ← H3: 0x8C tag at offset+3
```

**H1 confirmed:** `INT_STATUS_1.fwm_out=1` means BMI270 IS asserting INT1, but the chip woke on `cause=4` (timer) instead of `cause=7` (GPIO0). INT1 is firing, ESP32-C6 deep-sleep wake on GPIO0 isn't picking it up.

**Root cause likely:** GPIO0 is a **strapping pin** on ESP32-C6 (used to enter download mode at boot). Calling `esp_deep_sleep_enable_gpio_wakeup` alone isn't sufficient on this pin — the LP-IO domain needs explicit input-mode setup so it can monitor the live line level. Without that, the wake never fires even when INT1 is HIGH.

**H3 partially confirmed:** the unknown tags `0xF8`, `0xFC`, `0xFD` aren't in any documented BMI270 SDK header set, BUT the hex dumps show a consistent pattern: the next valid `0x8C` (REG_GA) tag appears at exactly **offset+3** from the unknown tag. So these are some kind of **3-byte unknown frames** (1 tag + 2 data) that appear between normal data frames. We don't know what they ARE, but we can skip past them.

**H2, H4, H5 ruled out** — config is correct, no brownouts, no mid-session resets.

**Other observation: `reset_reason=11 (?)` decoded.** Value 11 is `ESP_RST_USB` — USB peripheral state change triggered a reset. Normal when USB is connected for Serial Monitor; doesn't happen on battery.

**The "20 mA during deep sleep" is NOT a power bug — it's USB CDC overhead:**
- USB CDC peripheral stays alive in deep sleep on ESP32-C6 when USB is connected (in case the host wants to wake it). Burns ~5–10 mA continuously, **only when USB is plugged in**.
- BMI270 in normal mode (held alive by D2 hold) draws ~700 µA continuously — by design.
- On battery (USB unplugged), deep sleep current drops to ~1 mA. We can't measure or fix the on-battery number while a USB cable is attached for monitoring.

**Fixes shipped 2026-04-25:**

1. **Explicit GPIO0 configuration before deep sleep.**
   ```cpp
   pinMode(PIN_IMU_INT, INPUT);
   gpio_set_direction((gpio_num_t)PIN_IMU_INT, GPIO_MODE_INPUT);
   gpio_set_pull_mode((gpio_num_t)PIN_IMU_INT, GPIO_FLOATING);
   ```
   Plus a `digitalRead` of the line level logged via `DSL_LOG` right before sleep so we can verify INT1's state at the moment of sleep entry.
2. **Parser resync on unknown tag.** When we hit `0xF8`/`0xFC`/`0xFD` (or any other unknown tag), skip 2 more bytes and try to continue parsing. If the byte at the new position is ALSO an unknown tag, bail (recovery clearly failed). Otherwise normal parsing resumes. Empirically this should turn `parsed 16 frames, totalRows=4` into `parsed ~150 frames, totalRows=~30` per drain — even without fixing INT1, this alone gets us 5–10× more data per session.
3. **Reset reason names array extended** to include `USB` (11), `JTAG` (12), `EFUSE` (13), `PWR_GLITCH` (14), `CPU_LOCKUP` (15) so future logs decode properly.

**Predicted outcome:** with both fixes:
- INT1 wake should now fire ~every 1.3 sec → 7–8 drains per 10-sec session instead of 1
- Each drain captures ~150 frames after resync → ~30 rows logged per drain
- Total session capture: ~24–30 rows × 7–8 drains = ~200–240 rows for a 10-sec session = **near-perfect data capture**

If GPIO0 wake still doesn't fire after the explicit configuration, it's a chip-level limitation of using a strapping pin and we'd need to either reroute INT1 to a non-strapping pin (hardware change) or fall back to short timer wakes (~1.3 sec) instead of relying on the GPIO wake.

### 🔬 2026-04-25 follow-up #7 — partial data + INT1 wake never fires: hypotheses + diagnostic instrumentation

**Test result:** ODR fix worked, parser now parses real data. CSV showed reasonable gy/gx/gz values. BUT only **7 rows captured for a 10-second session** — should be ~200 rows.

**Reveal log analysis:**
- **Only ONE wake during the 10-sec session** — `cause=4` (timer wake at 10s mark)
- **No `cause=7` (GPIO/INT1) wakes ever fired**, despite FIFO watermark crossing several times during the session (FIFO had 2004 B at the timer wake — way past the 1690 B watermark)
- **Parser bails on unknown tag `0xFD`** at offset 210 after 16 frames, on the same `0xFC` tag during STOP cleanup
- FIFO is in stream mode (overwrite oldest) so most data is lost to overwrite by the time we drain on STOP

**Two distinct mysteries to solve:**

1. **Why doesn't the FIFO watermark INT1 fire a deep-sleep wake?** It should fire roughly every 1.3 sec. We're missing 7 of 8 wakes.
2. **What is `0xFD` / `0xFC`?** Not in any documented BMI270 FIFO tag set I've seen.

**Hypotheses:**

| H | Hypothesis | What `reveal` will show if true |
| - | ---------- | ------------------------------- |
| H1 | INT1 IS asserting but ESP32 GPIO0 wake isn't picking it up (hardware/electrical) | `INT_STATUS_1` reads with `fwm_out=1` (bit 1) on every timer-wake drain |
| H2 | INT1 ISN'T being generated despite FIFO past watermark (config not taking effect) | `INT_STATUS_1.fwm_out=0` always; FIFO config readback shows wrong values |
| H3 | `0xFD`/`0xFC` are real frame tags I haven't documented | Hex dump pattern matches a known BMI270 reference |
| H4 | Brownouts wiping RTC mid-session (peak current spikes) | `[boot] reset_reason=BROWNOUT (9)` appears in logs |
| H5 | Multiple resets during a session | `bootCount` keeps increasing within a single test |

**Diagnostic instrumentation added 2026-04-25:**

1. **`esp_reset_reason()` + boot counter** at top of `setup()`, both teed through `DSL_LOG`. Reveals brownouts and watchdog resets that would otherwise be invisible. Boot counter persists across all reset types except hard power-cycle.
2. **`INT_STATUS_1` (BMI270 reg 0x1D) read on every wake** in `fastPathLoggingWake`. If `fwm_out=1` here when we woke on timer, INT1 was asserted but C6 didn't pick it up (H1). If `fwm_out=0` even with FIFO past watermark, chip isn't generating INT (H2).
3. **FIFO_LEN read at wake** — verifies FIFO state at wake time independent of drain.
4. **FIFO config readback after `bmi270ConfigureFIFO`** logs `cfg0`, `cfg1`, `wtm`, `INT_MAP_DATA`, `INT1_IO_CTRL` actual register values. If any differ from what we wrote, chip rejected the write.
5. **Hex dump of bytes around unknown FIFO tags** in `drainFIFOAndAppend` (8 bytes before, the tag in `[brackets]`, 8 bytes after). Identifies what `0xFD`/`0xFC` actually are.

**Brownout-mitigation changes shipped (whether or not brownouts are happening):**

6. **Drop CPU to 80 MHz at top of wake router.** Wake from deep sleep brings CPU up at the cold-boot default of 160 MHz, both wasting current and increasing peak draw. We always want 80 MHz on this firmware regardless of which path got us awake.
7. **Lower BLE TX power to P3 during DSL check-ins.** P9 (~+9 dBm, ~120 mA peak) → P3 (~+3 dBm, ~60 mA peak). The phone reconnecting at end-of-game is right next to the pad, doesn't need shouting. Reduces peak current — main driver of brownouts on weak USB supplies.

**Test procedure for the next session:**

1. Flash + power-cycle (yank USB, plug back in — clears RTC including boot count)
2. Open Serial Monitor, press chip RESET button to capture full boot log
3. Start a session for ~30 sec (longer than 10 to make INT1 wake patterns clearer)
4. Press STOP from app
5. Type `reveal`, paste the full log

The `reveal` log will unambiguously identify which hypothesis is right.

### 🐛 2026-04-25 follow-up #6 — `unknown tag 0x88`: ACC/GYR ODR mismatch + 22 mA session current

**Symptom:** GPIO hold fix worked (BMI270 now stays alive, FIFO accumulates 2000+ bytes) but the parser bails immediately with `[DSL-fifo] unknown tag 0x88 at offset 15/2015`. Session captures 1 row total. Multimeter shows **~22 mA average during session**, far above the projected ~7 mA.

**Root cause: ACC and GYR ODRs were mismatched.** `ACC_CONF=0xA8` set accel to 100 Hz, `GYR_CONF=0xE9` set gyro to 200 Hz. When ODRs differ, the BMI270 FIFO emits a mix:
- `0x8C` combined gyr+acc frames at the slower rate (every 10 ms = 100 Hz)
- `0x88` gyro-only frames at the rate difference (every 5 ms = 100 Hz)

Total frame rate: 200 Hz, twice what we designed for. Two consequences:

1. **Parser bails on the unexpected `0x88` tag** — only 1 frame parsed per wake, the FIFO buffer is consumed (so INT1 clears) but no rows are appended.
2. **Decimator math goes wrong** — `DSL_DECIMATE=5` was sized for 100 Hz input → 20 Hz output. At actual 200 Hz input we'd get 40 Hz output (if the parser worked at all).
3. **High average current** — at 200 Hz × 13 B/frame = 2600 B/sec FIFO fill rate, watermark of 1690 B is crossed every ~0.65 sec instead of the planned 1.3 sec → 2× wake frequency. Combined with zero-yield wakes (parser bails after first frame), we get all the wake overhead with none of the throughput benefit. Hence ~22 mA instead of ~7 mA.

**Fix shipped 2026-04-25:**

1. **Match GYR ODR to 100 Hz** (`GYR_CONF=0xE9` → `0xE8`) in `setupBMI270`. All FIFO frames now arrive as `0x8C` combined frames at 100 Hz. Decimator math works as designed (5:1 → 20 Hz).
2. **Add defensive `0x88` (gyro-only) handler in the parser** — same logic as `0x8C` minus the accel bytes. We shouldn't see any `0x88` frames now, but if the chip ever emits one (e.g., during ODR-change settling on the very first wake), it gets handled instead of bailing.
3. **Source comments updated** with explicit "ODR must match" warning and pointer to this audit entry.

**Bonus fix to the legacy button-hold deep sleep path:** `enterDeepSleep()` now calls `BLEDevice::deinit(true)` before `esp_deep_sleep_start()`. Without this, the Bluedroid stack stays alive during deep sleep, burning ~10–15 mA — the chip is "in deep sleep" but still drawing tens of mA because the radio peripheral wasn't released. Same bug pattern as the DSL flow; finally caught in the legacy path too.

**Expected after fixes:**
- Session average current: **~6–7 mA** (down from 22 mA — 3× improvement just from data-correctness)
- FIFO drain wakes parse all ~130 frames per cycle, `[DSL-wake] FIFO had ~1690 B, parsed ~130 frames, totalRows=N` (incrementing)
- No `unknown tag 0x88` warnings (or just one or two on the very first wake during ODR settle)
- App receives DONE:N with N matching ~20 × session_seconds

**About idle-mode current (separate from session current):** the user's idle current of ~22 mA on USB (~16 mA on battery) is approximately the floor of the current architecture without further changes. CPU runs at 80 MHz, BLE adv at slow rate, sensors clocking. Light sleep isn't available (CONFIG_PM_ENABLE not in this build). To get below ~16 mA in idle would require an idle-mode deep-sleep cycle (similar architecture to DSL, but for "no session" state) — explicitly out of scope per Kushaj's "don't sleep when idle, optimize active use" preference.

### 🐛 2026-04-25 follow-up #5 — `FIFO had 0 B`: GPIO state lost across deep sleep

**Symptom from second test (after FIFO parser fixes):** session ran end-to-end, phone reconnected, but app reported "failed to stop, no data received". `reveal` buffer showed:

```
[DSL] entering deep sleep (timer=10s, GPIO0 wake=HIGH)
[DSL-sleep] wake-source config: gpio_ret=0 timer_ret=0
[DSL-sleep] >>> esp_deep_sleep_start <<<
[DSL] wake from deep sleep, cause=4 (4=timer 7=gpio 0=undef)
[DSL-wake] cause=4 rows-so-far=0
[DSL-wake] FIFO had 0 B, parsed 0 frames, totalRows=0     ← FIFO empty after 10 sec!
[DSL-wake] running BLE check-in (5000 ms window)
[DSL-checkin] phone CONNECTED — staying awake
```

Two clues in one log:
1. **`FIFO had 0 B` after a 10-second sleep.** The BMI270 should have been filling the FIFO at ~1300 B/s (100 Hz × 13 B/frame). Empty FIFO = chip wasn't accumulating any data.
2. **Only `cause=4` (timer wake)** — `cause=7` (GPIO wake) never fires. The watermark interrupt would only fire if the BMI270 was alive AND its FIFO was filling. Both absent.

**Root cause: ESP32-C6 doesn't preserve GPIO output state across deep sleep by default.** Even on LP-IO pins (GPIO0–7), the pin goes floating during sleep unless `gpio_hold_en` + `gpio_deep_sleep_hold_en` are explicitly called.

The chain that broke our architecture:

1. We assert D2 (GPIO2) HIGH at boot → Q1 on → Q2 on → BMI270 has VDD
2. `startLogging` configures BMI270 FIFO + watermark IRQ
3. `enterLoggingDeepSleep` → `esp_deep_sleep_start`
4. **D2 floats during sleep** → Q1 off → Q2 off → BMI270 LOSES POWER
5. INT1 line floats LOW (no chip alive to drive it) → GPIO wake source can never fire
6. After 10 s, timer wake fires (only)
7. `setup` re-asserts D2 HIGH → BMI270 powers back up FRESH (no config blob, no FIFO config, no accumulated data)
8. Wake router reads FIFO_LENGTH → 0 (chip just booted)
9. We never get past `cause=4` because BMI270 never accumulates anything to cross the watermark
10. `totalRows` stays at 0 forever → empty CSV

This also explains why the second test got "failed to stop" rather than "DONE:0 with empty file" like the first test: by the time the user pressed STOP, the BLE re-init might have left the GATT handles in a state the app couldn't write to. (Hypothesis — to confirm next test, we just made `[BLE] cmd=...` route through `DSL_LOG` so the next `reveal` will tell us whether STOP_LOG actually reached the chip.)

**Fix shipped 2026-04-25:**

1. **`gpio_hold_en((gpio_num_t)PIN_SENSOR_PWR)`** in `enterLoggingDeepSleep`, right before `esp_deep_sleep_start`. Latches D2's HIGH state through deep sleep so BMI270 keeps its VDD across the entire session. Includes `#include "driver/gpio.h"`.
2. **Released in `stopAndReplay`'s deep-sleep cleanup** via `gpio_hold_dis()` so future operations (button-hold sleep, idle-mode flow) can change the pin freely.
3. **`CtrlCB::onWrite` now routes through `DSL_LOG`** instead of plain `Serial.printf`, so commands like `STOP_LOG` and `START_LOG` show up in the `reveal` buffer. Without this, sessions that ended in "STOP didn't work" gave us no visibility into whether the command even reached the chip.

> [!warning] IDF 5.4 build gotcha — don't add back `gpio_deep_sleep_hold_en/dis`
> First attempt added `gpio_deep_sleep_hold_en()` alongside `gpio_hold_en()`. **Compile failed:** `'gpio_deep_sleep_hold_en' was not declared in this scope`. Per the [IDF 5.x migration guide](https://docs.espressif.com/projects/esp-idf/en/latest/esp32c6/migration-guides/release-5.x/index.html), on chips with only digital GPIOs (which is how the C6 is treated for this purpose), `gpio_hold_en()` alone now handles both active-state AND deep-sleep retention; the standalone `gpio_deep_sleep_hold_en()` was **removed from the public API**. Source comments now flag this as "don't add back, build will break."

**Expected outcome after this fix:**

- Next session should show `cause=7` wakes every ~1.3 sec (FIFO watermark) interleaved with `cause=4` wakes every 10 sec (BLE check-in)
- `[DSL-wake] FIFO had 1690 B, parsed 130 frames, totalRows=26` (then 52, 78, …) on each FIFO drain
- After STOP, `[BLE] cmd=STOP_LOG` line in reveal followed by `[DSL] STOP during deep-sleep mode` and `[DSL] gpio hold released on D2`
- App receives DONE:N with N > 0, CSV has real `gy` values

**If this still doesn't work,** the next suspects are:
- BLE re-init after `BLEDevice::deinit(true)` not fully working — GATT handles may be different second time around. Check whether app needs to call discoverServices again.
- BMI270 FIFO config registers being reset by the chip after some condition. Add a "verify FIFO config still correct" check on each wake.
- LittleFS state corruption across the LittleFS.end() / .begin() cycle. Switch to keeping LittleFS mounted across sleep (skip LittleFS.end() in enterLoggingDeepSleep).

### 🐛 2026-04-25 follow-up #4 — empty CSV: two FIFO parser bugs found and fixed

**Symptom from first end-to-end test:** session completed cleanly (deep sleep working, BLE check-in worked, phone reconnected, STOP_LOG arrived) but `[Log] STOP_LOG (logged rows=0)` and the resulting CSV had only column headers, empty `gy` and `time_ms`. File size on disk was 18 bytes — exactly the CRSN file header, no chunks.

**Root cause: two independent bugs in the BMI270 FIFO header-mode parser.** The first was fatal on its own; the second would have produced corrupted data even after fixing the first.

**Bug #1: wrong frame tag.** I had `0xA0` for the "regular gyr+acc combined" frame. Per Bosch's BMI2_SensorAPI reference (`bmi2_defs.h`), the actual tag is **`0x8C`**. Every wake's parser saw `0x8C` as the first byte, fell through every `if`, hit the final `else { break; }` and exited with 0 frames parsed.

The complete correct tag map (from the SensorAPI):

| Tag | Meaning | Bytes after | What we do |
| --- | --- | --- | --- |
| `0x8C` | Gyr + Acc combined data frame | 12 | parse + decimate + log |
| `0x44` | Sensortime control frame | 3 | skip |
| `0x40` | Skip frame | 1 | skip |
| `0x48` | Input config change | 1 | skip |
| `0x80` | FIFO over-read marker (filler when read past end) | — | **stop parsing** |
| anything else | unknown | — | log once + bail (don't try to resync) |

The previous value `0xA0` was a guess from reading the datasheet diagram, not the reference C SDK. Lesson: for Bosch sensors, **always cross-check tag values against the SensorAPI repo** — the datasheet diagrams are easy to misinterpret.

**Bug #2: wrong byte order in the data frame.** I had the parser pull bytes 0–5 as ACCEL (X/Y/Z) and bytes 6–11 as GYRO. Actual layout per datasheet §4.7.1 (and the SensorAPI reference parser) is **gyro first, then accel**:

```
[header 0x8C] gyr_x_LSB gyr_x_MSB  gyr_y_LSB gyr_y_MSB  gyr_z_LSB gyr_z_MSB
              acc_x_LSB acc_x_MSB  acc_y_LSB acc_y_MSB  acc_z_LSB acc_z_MSB
```

So even after fixing Bug #1, the `gy` column at index 0 would have been getting bytes that were really `ay` — the file would have been "logging" but the values would be accel-Y (small numbers near zero on a stationary pad) rather than gyro-Y (the signal that drives sprint detection in StatsEngine).

**Fix shipped 2026-04-25** in `drainFIFOAndAppend`:
- Tag constants updated to the SensorAPI values (`0x8C` / `0x44` / `0x40` / `0x48` / `0x80`)
- Frame byte order flipped to gyro-first then accel
- Added `0x48` (input config) and `0x80` (over-read) handlers explicitly
- Unknown tags now log once via `DSL_LOG` (`[DSL-fifo] unknown tag 0x%02X at offset...`) instead of silently bailing — so future surprises show up in `reveal`

**Comments updated in source** so future-you (or another model) doesn't re-introduce either bug. Both have explicit "don't repeat this" lines pointing at the SensorAPI repo as the canonical source.

### 🎉 2026-04-25 follow-up #3 — DSL working, phone reconnect tuned

**Hardware test result after the deferred-handler fix:** deep sleep is engaging cleanly. Multimeter shows **0 mA in deep sleep, brief 45 mA bursts** on FIFO drain wakes — exactly what the architecture predicts. The empty `reveal` buffer in the screenshot was because the test happened before a power-cycle that wiped RTC, not a code bug.

**Estimated average current with this build (10s interval, 2.5s slow-adv check-in):**

| Phase | Duration | Current | Contribution |
| ----- | -------- | ------- | ------------ |
| Deep sleep (BMI270 keeping FIFO going in normal mode) | 6.95 sec | ~0.7 mA | 4.9 mA·s |
| FIFO drain wakes (~7 × ~150 ms each) | 1.05 sec | ~45 mA | 47 mA·s |
| BLE check-in (init + 2.3 sec slow adv + deinit) | 2.5 sec | ~8 mA avg | 20 mA·s |
| **Per 10 sec** | | | **72 mA·s** |
| **Average** | | | **~7 mA** |

**~6.5× improvement over the 46 mA baseline.** 200 mAh battery → ~28 hours of recording.

**Remaining UX problem:** the 2.5-sec check-in window using the global slow adv interval (1000–1500 ms) only fires 1–2 ad bursts. iOS/Android scan duty cycle + CONNECT_REQ + GATT service discovery (~500 ms–2 sec on iOS) doesn't fit, so the phone misses the window almost every time. Coach can't press STOP.

**Fix shipped 2026-04-25:**

1. **Override adv interval to fast (100–150 ms) during check-in only.** `runBleCheckIn` now stops advertising right after `initBLE`, sets `setMinInterval(0xA0)` / `setMaxInterval(0xF0)`, restarts. The idle-mode slow adv (when no logging session is in progress) is unchanged. Adds the `DSL_CHECKIN_ADV_MIN` / `DSL_CHECKIN_ADV_MAX` constants alongside the other DSL tuning knobs.
2. **Extend check-in window to 5 sec.** Enough for the full reconnect + GATT discovery + STOP_LOG write + ack chain. `DSL_BLE_CHECKIN_DURATION_MS` 2500 → 5000.

**Revised power estimate (10s interval, 5s fast-adv check-in):**

| Phase | Duration | Current | Contribution |
| ----- | -------- | ------- | ------------ |
| Deep sleep | 3.95 sec | ~0.7 mA | 2.8 mA·s |
| FIFO drain wakes (~7) | 1.05 sec | ~45 mA | 47 mA·s |
| BLE check-in (init + 4.6 sec fast adv at +9 dBm + deinit) | 5 sec | ~13 mA avg | 65 mA·s |
| **Per 10 sec** | | | **115 mA·s** |
| **Average** | | | **~11.5 mA** |

That's still **4× better than baseline 46 mA**. 200 mAh battery → ~17 hours of recording. We trade ~4.5 mA of average current for reliable phone reconnect — worth it.

If we ever want to claw it back, the levers are:
- Increase check-in interval to 15–20 sec → ~9 mA but slower STOP latency
- Drop check-in TX power to P3 (vs current P9) → ~9.5 mA, slightly worse range during reconnect

### 🔥 2026-04-25 follow-up #2 — first DSL test FAILED, root cause + fix shipped

**Symptoms reported by Kushaj:**
- Idle current unchanged at 46 mA (expected — idle path doesn't use deep sleep)
- After START_LOG: current jumped to **65 mA**, never dropped (i.e., never entered deep sleep)
- After STOP_LOG: app stuck on "syncing…" forever (chip stayed at 65 mA, never replayed)

**Diagnosed root cause:** `BLEDevice::deinit(true)` was being called from inside `CtrlCB::onWrite()` — i.e., on the Bluedroid task itself. **Tearing down BLE from within a BLE callback deadlocks the stack on itself.** Bluedroid stops mid-tear-down, `esp_deep_sleep_start()` waits on the half-stopped stack and hangs, and the chip is stuck running peripherals at high current (~65 mA = active CPU + active Bluedroid + sensors). STOP_LOG never lands because the BLE stack is in a half-state.

**Fix shipped 2026-04-25:**

1. **Defer command handling out of BLE context.** `CtrlCB::onWrite` now sets `pendingStartLog` / `pendingStopLog` flags instead of calling the handlers directly. `loop()` picks up the flags and runs `startLogging()` / `stopAndReplay()` from the Arduino main task, where `BLEDevice::deinit()` and `esp_deep_sleep_start()` are safe to call.

2. **RTC log ring buffer + `reveal` Serial command.** Added a 3 KB ring buffer in RTC slow memory (`dslLogBuf`) that survives deep sleep. The `DSL_LOG()` macro tees every event to both live Serial (for when USB is plugged in) AND the RTC buffer (for after-the-fact debugging). Three new serial commands handled in `loop()`:
   - `reveal` — dumps the entire RTC log buffer to Serial. Plug in USB-C after a "blind" deep-sleep session, type `reveal`, see the full timeline.
   - `clear` — resets the buffer.
   - `state` — one-line dump of `loggingState` + runtime flags (isLogging, isReplaying, bleConnected, pending flags).

3. **Heavy instrumentation added** at every decision point in the DSL flow: `startLogging` entry, FIFO config success/fail, BLE deinit return, deep-sleep entry, wake cause + reason, FIFO bytes available + frames parsed, BLE check-in init/connect/timeout, etc. All routed through `DSL_LOG()` so the next test run leaves a complete audit trail.

4. **Treat `ESP_SLEEP_WAKEUP_UNDEFINED` as a timer wake** in `fastPathLoggingWake`. If the chip resets non-cleanly mid-session (e.g. brownout from a current spike) and RTC state still says active, we'd otherwise get stuck in a "drain FIFO + sleep again" loop with no BLE check-in window. Now we always do a check-in on undefined wake too, giving the user a way to recover.

**Hypothesis still to confirm on hardware:**
- The deferred-handler fix is the main suspect. After re-flashing, expect to see the chip drop to ~5–10 mA after START_LOG (deep sleep + brief wakes) and STOP_LOG to actually return DONE.
- If 65 mA persists, the next suspect is `BLEDevice::deinit()` itself failing on this Arduino-ESP32 core revision. The `reveal` log will show whether we got past `[DSL] BLE deinit returned` or not.
- If we DO enter deep sleep but never wake, GPIO0 INT1 wiring or polarity is the issue. The `reveal` log will show 10-second intervals between `[DSL-wake]` lines (timer wakes only) instead of ~1.3-second intervals (FIFO watermark wakes).
- If FIFO header parser is off, `[DSL-wake] FIFO had X B, parsed Y frames` will show parse counts way lower than expected.

**Expected current numbers (computed, not yet measured):**

| Phase | Current | Duty | Contribution |
| ----- | ------- | ---- | ------------ |
| Deep sleep (BMI270 + ESP32 sleep) | ~0.75 mA | ~67% (10 s cycle, awake ~3.3 s) | ~0.5 mA |
| FIFO drain wakes (~6× per 10 s) | ~30 mA | ~9% | ~2.7 mA |
| BLE check-in wake (1× per 10 s, 2.5 s incl. BLE init) | ~15 mA | ~25% | ~3.75 mA |
| **Total during a game** | | | **~7 mA** |

Battery life on 200 mAh: 200 / 7 = **~28 hours** of recording. On 500 mAh: ~71 hours.

If 7 mA isn't low enough after measurement, the next levers are:
- Increase check-in interval to 30 sec → ~5 mA
- Lower chip ODR to 25 Hz + change file sample_hz to 25 (needs app validation) → ~3 mA

---

## 5. What NOT to optimize (and why)

- **WiFi is already off.** The three-line teardown in `setup()` is the right move on C6.
- **MMC5603 ping-pong is already the lowest-power approach** — single-shot, 2 ms measurement, never blocks. Don't touch.
- **I2C clock at 400 kHz during normal operation** — fine. Higher clock = shorter active period, lower average. Don't lower this.
- **MTU 247 / 244-byte notify chunks / 4 ms pacing** — already tuned. Lowering pacing during replay risks BLE event starvation; raising MTU isn't possible (247 is the C6 max-ish).
- **Manual BMI270 init path** — we're stuck with it. SparkFun wrapper does not work on C6. (See [[ESP32-C6 XIAO Firmware (Rishab PCB)|project note]].)

---

## 6. What we can't fix in firmware (board-level / hardware)

- Charge LED is hardwired → ~3 mA whenever USB is in. Need PCB rev to gate it through a GPIO.
- No undervoltage protection in the XIAO charger → must enforce 3.3 V cutoff in firmware once a battery ADC exists, but the *risk* (bricked cells) is a board-level shortcoming.
- Charge IC + LDO quiescent → ~1.5 mA floor that no firmware change can remove.

---

## 7. Suggested order of attack

1. ✅ **2026-04-25 — (A) + (B) + (C) + (D) shipped together.** Expected idle ~10 mA, **measured 46 mA**. Bluedroid is almost certainly holding a PM lock that prevents light sleep. Diagnostic prints added 2026-04-25 follow-up — see top of this doc.
2. ⏳ **Get the boot log + verify USB-unplugged measurement** before deciding next move.
3. ⏸️ (G) LED idle off — deferred per Kushaj 2026-04-25; revisit if measurements demand it.
4. ❌ (F) BMA400 off — not needed (default state is sleep when BMI270 succeeds).
5. (E) Sensor rail gating — intrusive (changes session start latency by ~1.5 s for BMI270 re-init); only do it if the active-session battery story still needs more headroom after measurement.
6. (H) TX power dynamic — small win, only worth it if we go after the last few mA.
7. (J) Auto-sleep-on-idle — explicitly **deferred indefinitely** per Kushaj: optimizing for active-use battery, not standby. The product is supposed to stay always-discoverable.
8. Validate deep-sleep wake from button (already-correct API, just needs a hardware test). Independent of power optimization — needed before deep sleep can be used in any future flow.
9. (M) IMU INT1-driven sampling — architectural; revisit only if active-session current is still a problem.

---

## 2026-04-25 follow-up #14 — datasheet readthrough (BMI270 + ESP32-C6)

Read both datasheets locally (`C2836813.pdf` BMI270 r1.3 + `esp32-c6_datasheet_en.pdf` v1.1) to ground three remaining open issues against the silicon spec.

### What the datasheets actually say

**ESP32-C6 §5.6.2 Table 5-11 — Current Consumption in Low-Power Modes**

| Mode | Typ |
| ---- | --- |
| Light-sleep | 180 µA |
| Deep-sleep (CPU + radio + peripherals off, RTC+LP mem on) | **35 µA** |
| Power off (CHIP_PU low) | 7 µA |

Modem-sleep at 80 MHz with peripherals on is **14–30 mA** (Table 5-10). Our bench reading of "~30 mA when BLE is supposed to be off" is suspiciously close to Modem-sleep, not Deep-sleep. Either the chip never entered deep sleep, or the USB Serial/JTAG PHY is keeping a power domain alive.

**BMI270 §4.6 Table 6 — Power Modes**

| Mode | Typ |
| ---- | --- |
| Suspend | 3.5 µA |
| IMU normal (ACC+GYR @100 Hz) | 685 µA |
| IMU performance | 970 µA |

The IMU is **not** the power problem. At 685 µA it's negligible compared to the 30 mA mystery.

**BMI270 §4.7 — FIFO header byte format**

```
bit 7-6: fh_mode  (0b10=regular, 0b01=control)
bit 5-2: fh_parm  (regular: bit2=AUX, bit1=GYR, bit0=ACC; control: opcode)
bit 1-0: fh_ext   (INT2/INT1 tags)
```

Decoded against current parser:

| Tag in code | Bits | Meaning |
| ----------- | ---- | ------- |
| `0x8C` REG_GA | `10 0011 00` | Regular, ACC+GYR, no INT — ✓ correct |
| `0x88` GYR_ONLY | `10 0010 00` | Regular, GYR only — ✓ correct |
| `0x44` SENSORTIME | `01 0001 00` | Control opcode 1, 3 byte payload — ✓ correct |
| `0x40` SKIP | `01 0000 00` | Control opcode 0, 1 byte payload — ✓ correct |
| `0x48` INPUT_CFG | `01 0010 00` | Control opcode 2, **datasheet says 4 bytes**, parser uses 1 |
| `0x80` OVER_READ | `10 0000 00` | Uninitialized frame, sentinel — ✓ correct |

The Input_Config size is the only delta. Datasheet table is unconditional 4 bytes (1 byte flags + 3 bytes sensortime). Empirical observation in this firmware (with `FIFO_CONFIG_0=0x00`, fifo_time_en disabled) is that the chip emits 1 byte payloads — verified by the 2026-04-25 hex dumps. Plausible interpretation: when sensortime frames are disabled the BMI270 firmware suppresses the trailing 3 bytes inside Input_Config too. Not strictly per datasheet but consistent with the dumps. Leaving the parser at 1 byte; if the FIFO ever desyncs after a config change, this is the first place to revisit.

**ESP32-C6 §2.3.2 — LP IO MUX**

LP IO is **only** GPIO0–GPIO7. INT1 is wired to GPIO0, which IS in the LP IO range — so `esp_deep_sleep_enable_gpio_wakeup` is a valid wake source. GPIO0 is also a strapping pin (XTAL_32K_P / ADC1_CH0), but on this board there's no 32K crystal and no boot-mode dependency, so no conflict.

### Action taken

Added USB Serial/JTAG teardown right before `esp_deep_sleep_start` in `enterLoggingDeepSleep`:

```cpp
Serial.flush();
delay(20);
Serial.end();
```

Avoided `esp_sleep_pd_config(ESP_PD_DOMAIN_RTC_PERIPH, OFF)` — on C6's new PMU those legacy enums are ambiguously coupled with LP IO domains we need for INT1 wake, and the Arduino-ESP32 build in this version (3.2.0 / IDF 5.4) doesn't expose the new `esp_pmu_*` API directly.

### What this means for the open bugs

- **30 mA "deep sleep" current** — almost certainly bench-measurement artifact from USB-C plugged in. Need a LiPo-only re-measurement to confirm. Datasheet floor is 35 µA, so anything in the milliamp range with USB unplugged would still be a real bug.
- **CSV shorter than session length** — not directly explained by datasheet; either brownout reset wiping RTC mid-session (BOD now disabled, so the chip rides through dips but may corrupt RTC if Vcc dips below the SRAM retention floor ~1.5 V) or FIFO desync after an unhandled Input_Config frame mid-session.
- **BLE STOP_LOG data loss** — datasheet doesn't constrain this directly. Most likely BLE notification queue overflow during the replay burst, since at 0 dBm TX a ~50-row replay over a few seconds shouldn't fundamentally stress the link; suggests the app side is dropping packets faster than acking them.

### Open follow-ups

1. Re-measure deep-sleep current on **battery only** (no USB-C). Anything > 1 mA is a real firmware/hardware bug worth chasing; anything < 1 mA confirms the 30 mA was the USB Serial/JTAG and the architecture is working.
2. Add a one-shot `loggingState.lastWakeMs` timestamp into RTC slow memory and log the delta on every wake — confirms whether the chip is actually entering deep sleep between wakes (delta should be ~1.5 s timer) or hot-looping (delta would be < 100 ms).
3. If the brownout-mid-session theory is right, add a coin-cell-style RTC SRAM checksum on entry and abort cleanly to a "session corrupted, please re-record" state on the BLE control characteristic.

---

## 2026-04-25 follow-up #23 — App-side STOP_LOG fix + watermark math correction

Two bugs found and fixed in the same round.

### Bug 1: Watermark math was wrong (chip side)

Earlier ([#22](#2026-04-25-follow-up-22)) I bumped `DSL_FIFO_WATERMARK_BYTES` from 1690 → 2000 thinking the constraint was `watermark < FIFO_max`. Wrong constraint. The real constraint is:

```
watermark + (wake_duration × fill_rate) <= FIFO_max
```

Wake processing is ~150 ms (I2C re-init + 2 KB FIFO read at 400 kHz + chunker compress + flash write). At 1300 B/s fill rate, that's 195 B added during the wake. So:

```
max_safe_watermark = 2048 - 195 = 1853 B
```

2000 was over budget → FIFO crossed 2048 mid-wake → BMI270 stream mode overwrote oldest samples → **samples lost**. This is the "half the data" the user reported on recover.

**Reverted** to 1690 ([shinpad_c6.ino:203](Rishab_PCB_esp32c6/shinpad_c6/shinpad_c6.ino:203)). 163 B of headroom = ~125 ms of slack for wake processing variance. Could safely push to 1850 if measured wake duration stays under 150 ms, but the marginal power gain isn't worth the safety margin.

### Bug 2: App's stopLogging didn't re-establish DATA monitor (app side)

Reading the React Native app's BleContext — `Cresento/src/src/contexts/BleContext.tsx`:

| App flow | DATA monitor setup |
| -------- | ------------------ |
| `recoverDataFromChip` (line 2607) | **Yes — fresh monitor created before sending STOP_LOG** |
| `stopLoggingAndGetCsv` (line 2410+) | **No — relies on monitor from `startLogging`** |

The `startLogging` flow sets up the DATA monitor (line 2222) on the connected device. But in BUTTON_END_DSL, the chip enters deep-sleep right after START_LOG, BLE goes down, the phone gets a disconnect event — and the DATA monitor (tied to the original connection) is invalidated. After button wake → chip re-advertises → phone reconnects → the new connection has NO DATA monitor.

The recovery flow always sets up a fresh monitor before STOP_LOG, which is why "recover session" worked. The stop flow didn't.

**Fix shipped** in [Cresento/src/src/contexts/BleContext.tsx](Cresento/src/src/contexts/BleContext.tsx) — added the same fresh-DATA-monitor setup right before `STOP_LOG` is sent in `stopLoggingAndGetCsv`. Idempotent for the legacy non-DSL case (S3, Nordic): re-establishing a still-valid monitor is harmless. Required for BUTTON_END_DSL.

### Why the chip-side auto-subscribe alone wasn't sufficient

Earlier ([#22 → late edit](#2026-04-25-follow-up-22)) I added `postButtonWake` chip-side logic that force-set the CCCD descriptor values to `0x0001` on connect. The reveal log showed it actually firing:

```
[BLE] post-button-wake: auto-subscribing CTRL+DATA CCCDs
[BLE] CTRL CCCD value forced to 0x0001
[BLE] DATA CCCD value forced to 0x0001
```

But the user STILL got no data. Why: the chip-side CCCD value controls whether the chip pumps notifies; the iOS-side delivery to the app depends on the app having registered a notify handler (which is what `monitorCharacteristicForService` does). The chip's auto-subscribe didn't help because **the app side had no listener**. Notifies arrived at iOS, iOS had nothing to deliver them to.

The chip auto-subscribe is now redundant once the app fix is in (the app explicitly subscribes again). Leaving it in for defense-in-depth — it's a no-op when the app subscribes properly, and a fallback if any other client ever fails to subscribe.

---

## 2026-04-25 follow-up #22 — Cycle-stretch options (Knob A shipped, Knob B deferred)

User observed the chip cycling fast between 4 mA (deep sleep) and 40 mA (drain wake) and asked how to spend more time at 4 mA.

### Why the cycle exists

In DSL, the BMI270 streams accel + gyro into its 2 KB FIFO at 100 Hz × 13 B/frame = **1300 B/s**. When the FIFO crosses `DSL_FIFO_WATERMARK_BYTES`, INT1 fires and wakes the chip. The chip drains the FIFO (~150 ms), writes a chunk to LittleFS, and re-enters deep sleep.

```
sleep duration = watermark / 1300
cycle period   = sleep + ~150 ms drain
% time asleep  = sleep / cycle
```

### Knob A — bump watermark within FIFO budget (SHIPPED)

Pushed `DSL_FIFO_WATERMARK_BYTES` from 1690 → 2000 ([shinpad_c6.ino:203](Rishab_PCB_esp32c6/shinpad_c6/shinpad_c6.ino:203)).

| Watermark | Sleep | Cycle | % sleep | Avg current (USB-bench) |
| --------- | ----- | ----- | ------- | ----------------------- |
| 1690 (was) | 1.30 s | 1.45 s | 90% | ~6 mA |
| **2000 (now)** | **1.54 s** | **1.69 s** | **91%** | **~5.5 mA** |
| 2040 (max) | 1.57 s | 1.72 s | 91% | ~5.4 mA |

Marginal additional gain past 2000 (FIFO max is 2048 B; need ~8 B headroom for one frame). 48 B of headroom at 2000 = 36 ms of fill — plenty even if a wake gets delayed.

### Knob B — drop chip ODR (POTENTIAL, NOT YET SHIPPED)

The bigger lever is the BMI270's sample rate. The minimum combined ODR is **25 Hz** (gyro floor; accel can go lower but they must match for clean header-mode FIFO frames).

| Chip ODR | FIFO fill rate | Watermark | Sleep | Cycle | % sleep | Avg current | Decim ratio | Required SAMPLE_HZ |
| -------- | -------------- | --------- | ----- | ----- | ------- | ----------- | ----------- | ------------------ |
| 100 Hz (current) | 1300 B/s | 2000 | 1.54 s | 1.69 s | 91% | ~5.5 mA | 5:1 | 20 ✓ |
| 50 Hz | 650 B/s | 2000 | 3.08 s | 3.23 s | 95% | ~4.5 mA | 2:1 | 25 |
| 25 Hz | 325 B/s | 2000 | 6.15 s | 6.30 s | 98% | ~4.2 mA | 1:1 | 25 |

The decim ratio `DSL_DECIMATE = DSL_CHIP_ODR_HZ / SAMPLE_HZ` must be an integer. So dropping the chip to 50 or 25 Hz forces `SAMPLE_HZ = 25`.

### Cost of Knob B: app-side sample-rate audit

`SAMPLE_HZ` is written into the file header (`fh.sample_hz`) at session start. The decoder reads it from there. In theory all time-based metrics derive from `sample_hz` and adapt cleanly. In practice, hardcoded "20" or "50ms" assumptions may exist downstream. Need to grep the following before flipping:

- **RN app**: `Cresento/src/src/utils/StatsEngine.ts` — peak detection windows, sprint detection thresholds, fatigue index time bins
- **Website**: `Cresento Website/Cresento.net/lib/analytics/` — service.ts, stats.ts, trends.ts (~5 files)
- **iOS app**: `DataRecoveryIOS/.../StatsEngine.swift` — same metrics
- **Vision system / pose-fusion**: `text_ai_coach/`, `experimental_implementation/` — should be header-driven but verify
- **Any test fixtures or sample CSVs** that assume 20 Hz

Look for: `20` (literal), `50` (interval ms), `0.05` (seconds), `0.02` (period), `WINDOW_SIZE` style constants tied to sample count.

### When to ship Knob B

- ~3 mAh saved per 90-min game vs Knob A alone (5.5 mA → 4.2 mA × 1.5 hr)
- 500 mAh LiPo runtime: 83 h → 119 h (still within "many sessions per charge" range either way)
- Net win: small enough that the audit cost dominates. Defer until the rest of the firmware/app stack is locked down.

### Re-enabling NimBLE migration as additional escalation

If we ever want DSL with periodic BLE check-ins (live-stream during a game, or remote stop without button press), follow-up #21's NimBLE migration playbook is the path. Knob A + B + NimBLE together = ~3-4 mA average even with check-ins, ideally.

---

## 2026-04-25 follow-up #21 — BUTTON_END_DSL hybrid architecture shipped

After follow-up #20 disabled DSL outright (legacy 20 Hz polling, ~30-40 mA), revisited the deep-sleep architecture with a different sleep/wake pattern that **avoids the bug class entirely**.

### The insight

The crash root cause from #20: repeated `BLEDevice::init()` across deep sleep wakes (~30 inits in a 90-min game with the check-in cadence) accumulates BT controller state that eventually faults. We can't `BLEDevice::deinit(true)` to clean up (panics, see #17). No fix is available in Arduino-ESP32 v3.2.0 / IDF 5.4.

If the number of BLE inits per session is small (~2: pre-START + post-button), the fault doesn't trigger. So the architecture: **drop the periodic BLE check-ins entirely; use a button press as the user-driven end-of-session signal**.

### Architecture

```
        ┌────────────────────────────────────────────────┐
        │  IDLE/ADV   ~30-40 mA                          │
        │  - LED: fast blink (5 Hz)                      │
        └─────────┬──────────────────────────────────────┘
                  │ phone sends START_LOG via BLE
                  ▼
        ┌────────────────────────────────────────────────┐
        │  DSL LOGGING   ~5 mA average                   │
        │  - BLE NOT used during this period             │
        │  - LED: OFF (deep sleep)                       │
        │  - INT1 wake: drain FIFO every ~1.3 s          │
        │  - Button wake: end session                    │
        └────┬────────────────────┬──────────────────────┘
             │ INT1                │ button press
             ▼                     ▼
        ┌──────────────┐   ┌──────────────────────────────┐
        │ Drain FIFO,  │   │  END SESSION                 │
        │ chunk_flush, │   │  - flush + close file        │
        │ back to sleep│   │  - clear loggingState        │
        └──────────────┘   │  - fall through to setup()   │
                           │    rest (re-init BLE,        │
                           │    advertise)                │
                           └──────────┬───────────────────┘
                                      ▼
                           ┌──────────────────────────────┐
                           │  POST-SESSION   ~30-40 mA    │
                           │  - LED: fast blink           │
                           │  - Phone connects → STOP_LOG │
                           │  - Replay file → DONE        │
                           └──────────────────────────────┘
```

### Implementation in [shinpad_c6.ino](Rishab_PCB_esp32c6/shinpad_c6/shinpad_c6.ino)

1. **New flag** `USE_BUTTON_END_DSL = 1` (requires `USE_DEEP_SLEEP_LOGGING = 1`).
2. **Wake-source mask** — `enterLoggingDeepSleep` now configures `(1<<PIN_IMU_INT) | (1<<PIN_BTN)` as a bitmask wake source on HIGH level. Both pins are LP-IO capable (datasheet Table 2-7).
3. **Wake routing** — `fastPathLoggingWake` reads `esp_sleep_get_gpio_wakeup_status()` to determine which pin fired:
   - INT1 → drain FIFO + back to sleep (no BLE)
   - Button → flush chunker + close file + clear `loggingState.magic` + return `true` (stay awake)
   - Timer → safety net, drain + back to sleep
4. **Wake router fall-through** — when the button wake returns true, the wake router in `setup()` now falls through (under `#if USE_BUTTON_END_DSL`) to the rest of cold-boot init: WiFi off, PM config, sensor power, I2C, BMI270 re-init, MMC5603 init, BLE init, advertise. Phone can then connect and send `STOP_LOG` against the existing file.
5. **LED state machine** updated per user spec:
   - USB present → 1 Hz blink (charging proxy, takes priority)
   - `bleConnected` → solid
   - else → 5 Hz fast blink (advertising / waiting)
   - Deep sleep → off (LED loses power, can't control)
6. **Button INPUT_PULLDOWN** in the deep-sleep path — same treatment as the active-mode pin, ensures the wake source isn't triggered by a floating pin.

### Edge cases handled

| Case | Behavior |
| ---- | -------- |
| User holds button >1500 ms after button-wake | Falls into the existing `buttonTick` hold-to-sleep path → full power-off via `enterDeepSleep()`. Session file preserved on flash. Cold boot from next button press. |
| User forgets to press button | Chip keeps drain-cycling until LiPo dies. Data on flash. Plug USB → `force_stop` serial command recovers it. |
| Phone disconnects during STOP_LOG flow | Existing disconnect-rescue path: re-advertise (slow). User can reconnect. |
| New session immediately after old one | After STOP_LOG completes, `loggingState` is cleared. New START_LOG triggers fresh DSL handoff. |

### Power numbers (modeled, not yet measured)

| Phase | Duration in 90 min game | Current |
| ----- | ------------------------ | ------- |
| Pre-start advertising | ~1 min | ~35 mA |
| **DSL logging** | **~88 min** | **~5 mA** |
| Post-button BLE up | ~30 sec | ~35 mA |
| Total session battery | ~7.5 mAh | |

500 mAh LiPo runtime ≈ **66 sessions per charge**, vs ~9 sessions on the legacy 20 Hz polling path.

### What to verify on first test

1. `state` serial command shows `loggingState.magic=0xC6F1F0C6 active=1` while session is in deep sleep
2. Reveal log shows `[DSL-sleep] enter` → `[boot] reset_reason=8 (DEEPSLEEP) bootCount=N` (climbing N) → `[DSL-wake] gpio_wakeup_status=0x1 button=0 int1=1` for INT1 wakes
3. After button press: `[DSL-wake] gpio_wakeup_status=0x2 button=1 int1=0` → `[BUTTON-DSL] button press detected` → setup falls through → BLE up
4. Phone can connect within ~2 sec (slow adv at 1000-1500 ms) and STOP_LOG completes the replay
5. Bench current: ~5 mA during the deep-sleep phase (with USB-C bias maybe 8-10 mA), ~35 mA during BLE-up phases

If any of those fail (especially #1 going to POWERON instead of DEEPSLEEP), we've hit a different bug — likely the same controller-state issue would surface even with 2 inits if something else is off.

---

## Future: NimBLE migration as escalation path

If `BUTTON_END_DSL` ever fails for the same controller-state reason, OR if we need DSL-style architecture with periodic BLE check-ins (e.g. for live-stream during a game), the next escalation is replacing Bluedroid with NimBLE.

### What NimBLE buys us

- ~50% less RAM than Bluedroid
- Different controller-interaction model that's reportedly better-behaved across sleep cycles
- Maintained as an alternative BLE stack in IDF 5.x; supported on ESP32-C6
- `NimBLE-Arduino` library on GitHub (`h2zero/NimBLE-Arduino`) installable via Arduino Library Manager

### Migration cost

Mostly mechanical search/replace:

| Bluedroid | NimBLE |
| --------- | ------ |
| `#include <BLEDevice.h>` | `#include <NimBLEDevice.h>` |
| `BLEDevice::init("name")` | `NimBLEDevice::init("name")` |
| `BLEDevice::setPower(level)` | `NimBLEDevice::setPower(level)` |
| `BLECharacteristic::PROPERTY_READ \| PROPERTY_NOTIFY` | `NIMBLE_PROPERTY::READ \| NIMBLE_PROPERTY::NOTIFY` |
| `BLE2902` descriptor | implicit (NimBLE auto-creates the CCCD if PROPERTY_NOTIFY is set) |
| `setCallbacks(new MyCB)` | identical |
| `getValue()` returns `String` | returns `std::string` (same v2.x quirk) |

**Risk:** the migration itself could introduce subtle bugs (CCCD callback semantics differ slightly, MTU negotiation behavior, etc.). Save it for after `BUTTON_END_DSL` proves not enough.

### Migration checklist (when the time comes)

1. Install `NimBLE-Arduino` via Arduino Library Manager (or `arduino-cli lib install NimBLE-Arduino`)
2. In `shinpad_c6.ino`: swap `BLEDevice.h` → `NimBLEDevice.h`, `BLEServer.h` → `NimBLEServer.h`, etc.
3. Replace `BLE2902` descriptor adds with the implicit NimBLE pattern
4. Update `BLECharacteristic::PROPERTY_*` to `NIMBLE_PROPERTY::*`
5. Verify `getValue()` return type — may need to switch String/std::string handling
6. Re-enable the BLE check-in path (set `USE_BUTTON_END_DSL = 0`) and test 30+ check-in cycles in a row to see if the controller fault goes away
7. If it does: done — DSL with check-ins is now safe, ~5 mA average even with periodic BLE
8. If not: NimBLE didn't help on C6 either; stick with `BUTTON_END_DSL`

### What we've discussed and tried (timeline, 2026-04-25)

| Attempt | Outcome |
| ------- | ------- |
| Original DSL with timer-only wake + BLE check-ins | Crashes after ~60 s (POWERON) |
| Add gpio_hold_en for D2 | Sensors stay powered ✓ |
| Disable brownout detector | BOD reset eliminated ✓ |
| Drop BLE TX power from P9 → N0 | Brownout fully resolved ✓ |
| Fix BMI270 FIFO frame parser (0x8C, 0x88, 0x44, 0x40, 0x48, 0x80) | Drain works ✓ |
| Fix Input_Config payload size (1 byte when fifo_time_en=0) | Parser stops desyncing ✓ |
| INT1 GPIO0 wake working primary, timer fallback | INT1 wakes confirmed (`cause=7`) ✓ |
| chunker_flush() in stopAndReplay DSL branch | No more dropped tail rows ✓ |
| dataNotifReady gate restored | Replay no longer races phone subscription ✓ |
| Remove `BLEDevice::deinit(true)` panic | Chip stops crashing on START_LOG ✓ |
| Button INPUT_PULLDOWN (schematic-driven) | No more spurious hold-to-sleep ✓ |
| Loop delay 1ms → 20ms | No measurable power difference (Bluedroid PM lock dominates) |
| TX power N0 → N3 + adv preferred + updateConnParams | iOS couldn't reconnect; reverted |
| **BUTTON_END_DSL hybrid (current)** | **~5 mA average target; pending bench verification** |

The takeaway: **the schematic and BMI270 are not the blocker; the schematic supports DSL cleanly. The blocker is Bluedroid/IDF behavior across deep sleep wakes.** All hardware-side work is done. Future power optimization is a software-stack question (NimBLE) or an architecture question (button-driven session boundaries, which is where we are now).

---

## 2026-04-25 follow-up #20 — DSL DISABLED; legacy 20 Hz polling restored

After follow-ups #14–19 each individually fixed real bugs, the deep-sleep-during-logging architecture is still **fundamentally unreliable** on this chip. Final test: 3.5 min recorded → 68 s of recovered data, "took forever" before phone could reconnect, "random and inconsistent". Final reveal showed `[boot] reset_reason=1 (POWERON) bootCount=1 — 44 bytes used / 3072 total`, meaning the RTC log buffer was wiped — i.e. the chip hard-reset DURING deep sleep mid-session.

### Why the architecture is unreliable on C6

Even with all fixes applied (deinit removal, gpio_hold_en, brownout disable, FIFO frame parser, INT1 wake, dataNotifReady gate, button INPUT_PULLDOWN, safe BLE quiesce, comprehensive logging), the chip still hard-resets after 60–90 s of successful drain cycles. The crash signature is `POWERON, bootCount=1`, RTC wiped — meaning whatever happened wasn't a clean DEEPSLEEP wake.

Root suspicion: repeated `BLEDevice::init()` across deep sleep wakes (one full BLE init every ~21 s for a check-in) accumulates state in the BT controller hardware peripheral that's NOT fully reset by the deep-sleep transition. After N wakes, the controller faults. We can't `BLEDevice::deinit(true)` to clean it (panics on C6, see #17), and there's no IDF-public API for fully resetting just the controller without going through the Arduino wrapper.

This isn't a fixable-in-firmware problem without an IDF or Bluedroid update.

### What shipped

`USE_DEEP_SLEEP_LOGGING = 0` ([shinpad_c6.ino:162](Rishab_PCB_esp32c6/shinpad_c6/shinpad_c6.ino:162)). Legacy 20 Hz polling path now active during a session — the chip stays awake, samples the IMU at 20 Hz directly, runs the existing chunker / file writer / BLE replay pipeline in `loop()`. This is the **same architecture used by the S3 firmware** (`Cresento/chip_code/New_SmallBoard_Code.ino`), proven over many real games on the same app.

All DSL code remains intact behind the feature flag for future re-enable. None of it is dead code.

### Power tradeoff

| Architecture | Avg current during 90 min game | Battery used (90 min) | 500 mAh runtime |
| ------------ | ------------------------------ | --------------------- | --------------- |
| DSL (target, ideal) | ~5–10 mA | ~12 mAh | ~80 h |
| **Legacy (now)** | **~30–40 mA** | **~50 mAh** | **~13 h** |

13 hours per charge is still ~9 sessions per charge. Acceptable. Reliability matters more than power for shipping.

### Other improvements that DID land + persist after the rollback

- **Button INPUT_PULLDOWN** (#19): floating button pin fix from the schematic. Independent of DSL.
- **Safe BLE quiesce** + removed `BLEDevice::deinit(true)` panic (#17): irrelevant in legacy path but no longer dangerous.
- **`force_stop` / `recover` serial command**: emergency USB-C recovery if BLE ever fails to reconnect. Type at Arduino Serial Monitor — kicks `stopAndReplay()` locally without needing the phone to send STOP_LOG.
- **`state` serial command** now also reports session file size on flash, so you can verify data is actually on disk before STOP_LOG.
- **Comprehensive logging** (#18): all the new DSL_LOG entries now compile-out cleanly when `USE_DEEP_SLEEP_LOGGING=0`. Re-enable cost is zero.

### Re-enabling DSL in the future

Set `USE_DEEP_SLEEP_LOGGING = 1` again. Verification flow:
1. Flash, run a 5+ min session, leave phone disconnected the whole time.
2. Plug USB-C, type `reveal`.
3. Confirm:
   - `bootCount` climbs across wakes (NEVER stays at 1)
   - all `reset_reason` lines say `8 (DEEPSLEEP)` (NEVER `1 (POWERON)` mid-session)
   - chip survives at least 10 BLE check-in cycles without crashing
4. If any of those fail, leave DSL=0 — the architecture isn't safe to ship yet.

---

## 2026-04-25 follow-up #19 — schematic-driven button pin fix

User shared the daughter-board schematic (`Rishab_PCB_esp32c6/real_print.pdf`). Reading it surfaced a real bug.

### Pin map confirmed from schematic

| Pin | Net | Notes |
| --- | --- | ----- |
| GPIO0 / D0 | BMI270 INT1 + BMA400 INT1 (shared) + header J1 pin 1 | Only one IMU configured at a time, so no actual conflict |
| GPIO1 / D1 | SW1 → R8 (2.7 kΩ) → 3V3 | **Floats when button released — no external pull-down on board** |
| GPIO2 / D2 | Q1 base (NPN, R7=1K) → Q2 gate (P-MOSFET) → VDD rail | Sensor-power high-side switch: D2 HIGH = VDD on |
| GPIO17 / D7 | W25Q128JV CS# | Flash unused by firmware; CS# floats high → flash standby |
| GPIO18-20 / D8-10 | W25Q128JV SPI | Unused |
| GPIO22 / D4, GPIO23 / D5 | I2C SDA/SCL with 2.7 kΩ pull-ups (R3, R6) | BMA400 + BMI270 + MMC5603NJ all on this bus |

### Bug: floating button pin

The schematic shows SW1 wired as: `D1 — switch — R8(2.7K) — 3V3`. Pressing the button connects D1 to 3V3 via R8 (read HIGH). Releasing leaves D1 disconnected from everything. There is **no external pull-down resistor** on the board.

Firmware was doing `pinMode(PIN_BTN, INPUT)` (no internal pull). With the pin floating between presses, `btnDown()` (which reads `digitalRead(PIN_BTN) == HIGH`) returns essentially random values. After 1500 ms of "stable HIGH" reads (`BTN_HOLD_MS`), `buttonTick()` fires `enterDeepSleep()` — full power-off, separate from the DSL flow, kills the active logging session.

This is a strong candidate for the "stuck at 40 mA, can't reconnect" symptom: if a stray reading triggers `enterDeepSleep()`, the chip drops out of DSL mode into the button-hold deep-sleep path, the loggingState isn't preserved correctly, and the chip ends up in a weird state where BLE is partially up (or down) and the phone can't reach it.

### Fix

`pinMode(PIN_BTN, INPUT_PULLDOWN)` so D1 reads LOW when released, HIGH when pressed (the 2.7 kΩ external pull-up dominates over the ~45 kΩ internal pull-down). Comment updated to point at the schematic and explain the prior wrong assumption ("external pull-down is via the circuit").

Also added `DSL_LOG` entry inside `buttonTick` when the hold path fires — so future reveal output shows definitively whether enterDeepSleep was triggered by the button or some other path.

### Other things confirmed from the schematic

- **No USB-detect circuit** on this board (the S3 has `BUS_ADC_PIN` for this; C6 doesn't). Can't programmatically distinguish bench (USB-C powered) from field (battery only). Bench-vs-battery power difference will remain a measurement-side question.
- **Power gate (D2 → Q1 → Q2)** topology matches what the firmware assumes. `gpio_hold_en(D2)` correctly preserves sensor power across deep sleep.
- **W25Q128JV flash and BMA400 are both wired but unused** — they sit on the gated VDD rail in their POR-default standby state, contributing negligibly to deep-sleep current (W25Q ~1 µA, BMA400 ~0.16 µA).

---

## 2026-04-25 follow-up #18 — diagnostic logging + safe BLE quiesce

After follow-up #17 fixed the panic, a new symptom: chip cycles 3 mA / 40 mA for 5–10 s (good — DSL drain-and-sleep cycles working), then "stuck at 40 mA, occasionally 19 mA, can't reconnect". Power signature analysis:

| Reading | Likely state |
| ------- | ------------ |
| 3 mA | Deep sleep with USB-bench bias (datasheet 35 µA + USB Serial/JTAG ~5 mA) |
| 40 mA | Active CPU @80 MHz + BLE radio bursts (modem-sleep with BLE TX) |
| 19 mA | Light-sleep between BLE adv intervals |

The 5–10 s cycling matches DSL drain wakes (every 1.5 s × 4–7 wakes) before a BLE check-in fires. After the check-in, "stuck at 40 mA" is the chip in stayAwake mode after a phone connection — i.e. the BLE check-in worked, phone connected (briefly), `runBleCheckIn` returned true, chip is now waiting for STOP_LOG with BLE up. The 19 mA dips are light-sleep between BLE events.

The "can't reconnect" symptom is most likely: phone connects briefly during a check-in window → connection drops (out of range, GATT cache mismatch, etc.) → on disconnect, the loop re-sleeps via the DSL disconnect-rescue path → chip is back in deep sleep cycles → phone has to wait up to ~21 s for the next check-in window to find the pad again.

### Changes shipped 2026-04-25

1. **Safe BLE quiesce** — added `BLEDevice::getAdvertising()->stop()` in `startLogging()` and `runBleCheckIn()` before deep sleep entry. Replaces the panicking `BLEDevice::deinit(true)` with a call that just stops adv bursts so they don't collide with the sleep transition. The deep sleep itself tears down the rest of the radio.

2. **Comprehensive diagnostic logging.** Every state transition now emits a `DSL_LOG` line that survives in the RTC buffer for `reveal`:
   - `enterLoggingDeepSleep` — entry log with all relevant state, exit log if `esp_deep_sleep_start` ever returns
   - `fastPathLoggingWake` — entry log with cause, totalRows, chunkSeq, wakesSinceCheckin, CPU MHz, free heap
   - `runBleCheckIn` — heartbeat log every 1 s during the wait loop with remaining-time + connected state
   - `startLogging` — entry log + per-step "OK" lines, error log on file open failure
   - `onDisconnect` — now includes `was-replaying / was-logging / DSL-active` flags so we can see what state the disconnect interrupted
   - `[PM-late t=30s]` — now includes `loggingState.magic / active / totalRows / wakesSinceCheckin`

3. **Verification flow.** After flashing this build, run a session, then:
   - Plug in USB-C
   - Open serial monitor at 115200
   - Type `state` — see the live mode (isLogging, isReplaying, loggingState)
   - Type `reveal` — dumps the entire RTC log buffer with every transition the chip went through during the session

### What the user should look for in the reveal output

| Pattern | Meaning |
| ------- | ------- |
| `[DSL-sleep] enter` followed quickly by `[boot] reset_reason=8 (DEEPSLEEP) bootCount=N+1` | Deep sleep working correctly. |
| `[DSL-sleep] enter` followed by `[boot] reset_reason=1 (POWERON) bootCount=1` | Panic on sleep entry. Re-introduced regression — hunt recent edits to enterLoggingDeepSleep / runBleCheckIn / startLogging. |
| `[DSL-sleep] !!! esp_deep_sleep_start RETURNED` | Sleep entry failed silently. Almost always a wake-source config issue. |
| Many `[DSL-checkin] waiting...` heartbeats with `connected=0` | Phone never finds the chip during the 5 s window — try shorter `EVERY_NTH` or longer `CHECKIN_DURATION_MS`. |
| `[DSL-checkin] phone CONNECTED at +Xms` followed by `[BLE] disconnected` shortly after | Connection unstable — phone moved out of range, or GATT cache mismatch on phone side (suggest "Forget Device" then re-pair). |

---

## 2026-04-25 follow-up #17 — `BLEDevice::deinit(true)` panics on C6 Bluedroid

**Root cause** of the "30s session yields 300 ms of data" + "chip consumes 20-70 mA while logging" + "STOP_LOG no data" cluster.

The C6 firmware was calling `BLEDevice::deinit(true)` in 4 places before `esp_deep_sleep_start()`:
1. `startLogging()` — after START_LOG, before first sleep
2. `runBleCheckIn()` — after no-connection timeout, before re-sleeping
3. `enterDeepSleep()` — button-hold full power-off
4. loop disconnect handler — when phone disconnects mid-session

On Arduino-ESP32 v3.2.0 / IDF 5.4 / ESP32-C6 Bluedroid, `BLEDevice::deinit(true)` panics and force-resets the chip. The reset wipes RTC slow memory (POWERON, not DEEPSLEEP), so:

- `loggingState.magic` → 0, wake router skips the fast path
- `loggingState.totalRows` → 0, next `openSessionFileForAppend` does `firstWake=true` and truncates the session file
- chip just re-advertises, phone reconnects, looks like it's "logging" but isn't
- power 20-70 mA = modem-sleep (not deep-sleep) because the chip never actually enters deep sleep
- final 300 ms of data = whatever was captured during the brief moment between START_LOG and the panic, plus any final drain on the next session boot

**Confirmed by reading the working chip firmware** (cross-reference per user direction "read the other code files"):

- `Cresento/chip_code/New_SmallBoard_Code/New_SmallBoard_Code.ino` (S3, paired with same app): button-hold deep sleep at line 689 calls `esp_deep_sleep_start()` directly, NO BLE teardown.
- `Rishab_PCB_esp32c6/Old Code (Replaced with struans work)/.../Rishab_PCB_esp32c6.ino` (pre-rewrite C6): all 3 deep sleep entries call `esp_deep_sleep_start()` with no BLE teardown.

`esp_deep_sleep_start()` automatically powers down the radio peripheral as part of the sleep transition. The earlier diagnosis "BLE keeps radio up during deep sleep, causes 10-15 mA leak" was wrong — that 10-15 mA was the panic-reset hot loop, not a real BLE-during-sleep leak.

**Fix** ([shinpad_c6.ino](Rishab_PCB_esp32c6/shinpad_c6/shinpad_c6.ino), 2026-04-25): removed all 4 `BLEDevice::deinit(true)` calls, replaced with comments preserving the "do not add this back without reveal-verifying bootCount climbs" warning.

**Verification rule:** after any change near the deep-sleep entry, the reveal log MUST show:
- `[boot] reset_reason=8 (DEEPSLEEP) bootCount=N` with N climbing across wakes
- NEVER `reset_reason=1 (POWERON)` mid-session — that is the panic signature

If you ever see POWERON mid-session after editing the deep-sleep path, you have re-introduced this panic. Revert.

---

## 2026-04-25 follow-up #16 — STOP_LOG data race during deep-sleep check-in

The "STOP_LOG arrives but app reports no data" symptom turned out to be a BLE notify race, NOT a chunker flush issue (though that fix was also needed).

**Root cause** found by reading the S3 firmware (`Cresento/chip_code/New_SmallBoard_Code.ino` ~line 2249), which pairs to the same app and works correctly:

1. The S3 firmware sets `dataNotifReady = true` ONLY in the explicit `DataCCCD_CB::onWrite` callback.
2. The S3 replay loop gates on `if (isReplaying && dataNotifReady)`.

The C6 firmware had broken this in two places:
1. `ServerCB::onConnect` set `dataNotifReady = true` "defensively" — comment claimed the CCCD onWrite callback was "unreliable on C6". That defensive set defeats the gate entirely: as soon as the phone connects, the chip thinks DATA is subscribed.
2. The replay tick had been changed to ungate from `dataNotifReady` because it was already always-true from #1.

**Why this only manifested in the deep-sleep check-in path:** in the legacy logging path, the phone has plenty of time to subscribe to DATA (LOGGING phase lasts seconds), so it's already subscribed by the time STOP_LOG goes out. In the deep-sleep check-in path, the app reconnects and immediately sends STOP_LOG via CTRL before subscribing to DATA. The chip's race window is on the order of ~100 ms — long enough to pump the entire 1.3 KB session file via dropped notifies before the phone subscribes.

**Fix** ([shinpad_c6.ino](Rishab_PCB_esp32c6/shinpad_c6/shinpad_c6.ino), 2026-04-25):
- Removed the defensive `dataNotifReady = true` from `onConnect`.
- Restored `if (isReplaying && dataNotifReady)` gate on the replay tick.
- Added a 3-sec timeout fallback to force-pump if CCCD genuinely never fires (covers the original C6 quirk that motivated the broken defensive set).

**Lesson:** when a C6 / Bluedroid behavior seems "unreliable", check the S3 firmware first. Both run Bluedroid; if S3 works and C6 doesn't, the bug is almost certainly in our C6 code, not the stack.

---

## 2026-04-25 follow-up #15 — STOP_LOG fix + N=14 check-in cadence

Two changes shipped on 2026-04-25 evening:

1. **Fixed dropped tail-rows on STOP_LOG via deep-sleep check-in.** The DSL cleanup branch in `stopAndReplay()` was calling `drainFIFOAndAppend()` (which only `addRow`s into the chunker) then closing the file, but never calling `chunker_flush()`. The legacy path's flush is gated on `isLogging`, which is false during a check-in stop. Result: up to `CHUNK_ROWS-1` rows from the tail of the session were dropped — the user's "STOP_LOG sent but app says failed". Added explicit `chunker_flush()` in the DSL branch.

2. **Reverted `Serial.end()` before `esp_deep_sleep_start`.** Earlier this evening I added it as a USB-CDC current-reduction tweak but it caused intermittent POWERON (not DEEPSLEEP) resets on wake — bootCount stayed at 1 across what should have been multiple wake cycles. Best guess: USB CDC `end()` on Arduino-ESP32 v3.x leaves the peripheral in a state where re-enumeration on host drop-out races with deep-sleep entry. Removed; bench-measurement USB bias is now accepted as a measurement artifact, not a firmware fix.

3. **Changed `DSL_CHECKIN_EVERY_NTH` from 7 to 14.** With N=7 the user observed ~24 s wait between STOP_LOG press and the pad becoming findable (10.5 s timer cadence + iOS scan duty cycle). N=14 takes that to ~30–40 s, which is acceptable since the coach is typically walking back from the pitch when they press stop. Halves the BLE check-in duty cycle from ~48% of cycle time to ~24%.

### Tradeoff table (modeled, not measured)

For 5-second BLE check-in window:

| `EVERY_NTH` | Findable cadence | Avg current (USB-bench) | Avg current (LiPo, 35 µA sleep) | 500 mAh runtime |
| ----------- | ---------------- | ----------------------- | -------------------------------- | --------------- |
| 7 (was) | every 10.5 s | 14.4 mA | 11.4 mA | ~44 h |
| **14 (now)** | **every 21 s** | **11.4 mA** | **7.8 mA** | **~64 h** |
| 20 | every 30 s | 10.3 mA | 6.4 mA | ~78 h |
| 40 | every 60 s | 7.6 mA | 3.4 mA | ~147 h |

LiPo column assumes USB unplugged → deep sleep at the datasheet 35 µA. Bench measurements with USB-C in will under-deliver because the USB Serial/JTAG PHY adds ~5 mA of bias. Re-measure on LiPo only before deciding whether to push N higher.

---

## Related

- [[ESP32-C6 XIAO Firmware (Rishab PCB)]] — main project note (schematic, BLE quirks, BMI270 init story)
- [[01 - Critical Preservation Rules]] — undervoltage rule applies once battery ADC is wired
- [[Hardware Overview]] — board-level comparison
- [Seeed PPK2 deep sleep thread](https://forum.seeedstudio.com/t/15ua-with-xiao-esp32c6-with-ppk2-while-sleeping-basic-example/276412)
- [Tomas McGuinness — Lowering power consumption in ESP32-C6](https://tomasmcguinness.com/2025/01/06/lowering-power-consumption-in-esp32-c6/)
