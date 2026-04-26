---
title: ESP32-C6 XIAO Firmware (Rishab PCB)
type: project
tags:
  - project/esp-c6
  - firmware
  - hardware
  - critical
created: 2026-04-14
updated: 2026-04-25
status: active
location: "Rishab_PCB_esp32c6/shinpad_c6.ino"
---

# ESP32-C6 XIAO Firmware (Rishab PCB)

A third ShinPad firmware track targeting the **Seeed XIAO ESP32-C6** mounted on a custom **daughter board designed by Rishab** (schematic: `real_print.pdf`). Same BLE GATT contract as the [[Nordic BLE Firmware (nRF52811)|Nordic firmware]] and [[ESP32-S3 SmallBoard Firmware|ESP32-S3 firmware]] so all three look identical to the React Native / iOS apps.

> [!info] Three parallel firmware tracks
> Cresento now has THREE chip designs. This one uses a COTS XIAO ESP32-C6 module on a custom sensor carrier board, making it cheaper and easier to assemble than the full-custom ESP32-S3 PCB.

> [!success] State as of 2026-04-25
> Full board is built. Firmware in `Rishab_PCB_esp32c6/shinpad_c6.ino` has BMI270 (gyro+accel), MMC5603NJ magnetometer, and BLE pairing to the app **all working end-to-end**. WiFi upload, SPI flash, battery ADC, and deep-sleep wake validation are intentionally deferred.

> [!info] 2026-04-25 — first power optimization pass shipped
> Measured 50 mA idle on the bare assembled board. Applied four Tier-1 fixes targeting ~10 mA idle: `setCpuFrequencyMhz(80)`, `esp_pm_configure(...light_sleep_enable=true)`, idle-aware `delay(40)` in `loop()`, and **single-mode slow advertising at 1000–1500 ms** (replaces previous 100–150 ms). LED-idle-off was deferred. No library or partition changes — but **Arduino-ESP32 core ≥ 3.0** is now required (for `esp_pm.h`). See [[ESP32-C6 Power Audit]] for the full reasoning, the use-pattern discussion that motivated single-mode adv, and the Arduino IDE settings table.

> [!warning] 2026-04-25 — first pass disappointed (50 mA → 46 mA), root cause found
> Diagnostic boot prints showed `esp_pm_configure ret=262 (ESP_ERR_NOT_SUPPORTED)`. **`CONFIG_PM_ENABLE` is not compiled into the Arduino-ESP32 build for this board on the installed core version.** Light sleep silently never engaged. Only `setCpuFrequencyMhz(80)` and slow adv survived → ~4 mA savings. Decision: don't pursue PM rebuilding; even with light sleep working, Bluedroid APB lock would cap us at ~25–30 mA. See [[ESP32-C6 Power Audit#⚠️ 2026-04-25 follow-up — measured 46 mA, expected ~10 mA]] for the full diagnosis.

> [!success] 2026-04-25 — second pass shipped: BMI270 FIFO + ESP32 deep-sleep architecture
> While **logging during a game**, the firmware now: (1) configures BMI270's onboard 2 KB FIFO with a watermark interrupt routed to INT1 (GPIO0), (2) tears BLE down entirely, (3) `esp_deep_sleep_start()` until either the FIFO INT1 fires (~every 1.3 sec to drain) or a 10-sec timer wake triggers a brief BLE check-in window. Wake router at the top of `setup()` detects "we are mid-session" via an `RTC_DATA_ATTR` state struct and jumps to a fast path. **Idle/discovery mode is unchanged** — slow advertising at 1000–1500 ms — only the active logging session uses deep sleep. Computed average current during a 90-min game: **~7 mA** (vs current ~46 mA — a 6.5× win). Behind `#define USE_DEEP_SLEEP_LOGGING 1`; flip to 0 to revert. **Untested on hardware as of 2026-04-25.** Full design + risks + open questions in [[ESP32-C6 Power Audit#P. **BMI270 FIFO + ESP32 deep sleep during logging** ⭐ — ✅ **shipped 2026-04-25 (untested on hardware)**]].

> [!danger] 2026-04-25 — CHIP_PU coupling glitch is the actual root cause; firmware is at limit
> Read ESP32-C6 datasheet (`esp32-c6_datasheet_en.pdf`). Section 5.4 Table 5-4 reveals: **`VIL_nRST = 0.25 × VDD` on CHIP_PU — a VDD sag dropping CHIP_PU below 25% of VDD triggers a POR**, independent of BOD. This is what's causing the repeated `reset_reason=1 (POWERON)` events: BLE TX bursts (130 mA peak even at 0 dBm per Table 5-8) couple capacitively into CHIP_PU on the XIAO module's PCB, the pin glitches below 0.25×VDD, the chip POR-resets. **Disabling BOD didn't help because POR via CHIP_PU is a separate reset path.** Hardware fix needed: RC filter (10 kΩ + 100 nF) on CHIP_PU + bulk decoupling at VDD. **Firmware fixes shipped:** CPU dropped 80→40 MHz (datasheet floor with BLE up per §4.1.3.3), GPIO drive strength on PIN_SENSOR_PWR + PIN_LED lowered to GPIO_DRIVE_CAP_0 (~5 mA from 20 mA default) to reduce coupling. Net: firmware is now at its limit; remaining issues are hardware. See [[ESP32-C6 Power Audit#🚨 2026-04-25 follow-up #13 — CHIP_PU glitch is the actual culprit; firmware is at limit]].

> [!danger] 2026-04-25 — `setCpuFrequencyMhz(40)` BREAKS BLE PAIRING — do not lower below 80 MHz
> Firmware was edited (between sessions) to drop CPU to 40 MHz with a comment claiming that's the "BLE-stack floor per datasheet §4.1.3.3". **This is wrong and breaks the product.** At 40 MHz the chip boots and advertises but the Bluedroid stack can't service connection events fast enough → phone can't pair. Verified on hardware: "im not able to even pair to the chip". Reverted both `setCpuFrequencyMhz` calls to 80 (cold-boot setup AND wake-router fast path) with explicit ⚠ DO NOT comments. Marginal current savings of dropping to 40 MHz aren't worth losing pairing entirely. See [[ESP32-C6 Power Audit#⚠️ 2026-04-25 follow-up #13 — `setCpuFrequencyMhz(40)` breaks BLE pairing — DO NOT lower below 80 MHz]].

> [!info] 2026-04-25 — Session ran end-to-end; partial data loss + INPUT_CFG re-fixed to 1 byte
> After BOD-disable + TX-power fix: chip stayed alive through the session, captured 38 rows, no resets. **Two new findings:** (1) INPUT_CFG payload is **1 byte not 4** in our config — the "+3 sensortime bytes" the datasheet mentions is conditional on `fifo_time_en=1` which we explicitly disable. Hex-dump math: 16 GA × 13 = 208 + INPUT_CFG (2 bytes total) at 208–209 + unknown at 210. Payload reverted to 1 byte. (2) Parser resync now scans forward up to 16 bytes for ANY valid tag (was: skip 2 bytes blindly, fail). Should recover most of each drain. Also: wake-cycle logs missing from RTC buffer suggests **brownouts STILL happening** — BOD-disable stops the reset, but voltage dip drops below RTC SRAM retention (~1.5V), corrupting the log buffer mid-session. Real fix is hardware (battery >3.7V, bulk decoupling). Seeed wiki confirms 15 µA deep sleep claim is at **3.8V supply only**. See [[ESP32-C6 Power Audit#🔧 2026-04-25 follow-up #12 — partial DSL data + RTC corruption hypothesis + parser fixes]].

> [!danger] 2026-04-25 — BROWNOUT confirmed as root cause; BOD disabled + TX power dropped
> Reveal finally caught `reset_reason=9 (BROWNOUT) bootCount=1` after testing on battery (USB unplugged). The XIAO C6's SGM6029 LDO sags during BLE TX bursts on a marginal LiPo, dropping the rail below ~2.5 V which trips the brownout detector → chip resets → RTC wiped → session lost. **This was the actual cause of every weird symptom over the past 2 days** (auto-reconnect failing, reveal log empty, partial CSVs, "failed to stop"). **Two fixes shipped:** (1) `esp_brownout_disable()` at very top of `setup()` to ride through transient voltage dips without resetting; (2) default BLE TX power dropped from P9 (+9 dBm, ~120 mA peak) to N0 (0 dBm, ~30 mA peak) — eliminates the worst peak-current scenario; range drops from ~10 m to ~3 m which is fine for actual use pattern. Hardware caveat for next PCB rev: add bulk decoupling near LDO + use ≥500 mAh LiPo. See [[ESP32-C6 Power Audit#🩹 2026-04-25 follow-up #11 — `BROWNOUT` reset confirmed; BOD disabled + TX power dropped]].

> [!success] 2026-04-25 — `0xF8`/`0xFC`/`0xFD` solved: Input_Config payload is 4 bytes, not 1
> Read the BMI270 datasheet (rev 1.3, §FIFO control frames pp. 40–42, `Rishab_PCB_esp32c6/C2836813.pdf`). The `Fifo_Input_Config` frame (tag `0x48`) has a **4-byte payload** (1 config byte + 3 sensortime bytes), not 1 byte as my parser assumed. So my parser was leaving 3 bytes of sensortime in the buffer, then reading the first of those bytes as a "tag" — which is what `0xF8`/`0xFC`/`0xFD` actually were: data bytes from a mis-parsed Input_Config. Fix: `i += 4` instead of `i += 1` in INPUT_CFG branch. Source comment now cites the datasheet section + PDF path so this can't regress. Also shipped a **timer-fallback architecture** (1.5 sec timer wake + every-7th-wake BLE check-in) so we don't depend on the unreliable GPIO0 strapping-pin wake. Predicted outcome: full ~520 rows for a 30-sec session. See [[ESP32-C6 Power Audit#🎯 2026-04-25 follow-up #9 — root cause of `0xF8`/`0xFC`/`0xFD` found in datasheet]].

> [!success] 2026-04-25 — H1 + H3 confirmed: GPIO0 strapping-pin issue + 3-byte unknown frames
> Reveal showed `INT_STATUS_1.fwm_out=1` AND `cause=4 (timer)` simultaneously — BMI270 IS asserting INT1 but ESP32 isn't waking on GPIO0. **GPIO0 is a strapping pin on C6** (boot-mode select); just calling `esp_deep_sleep_enable_gpio_wakeup` isn't sufficient — the LP-IO domain needs explicit input-mode setup. Hex dumps also revealed `0xF8`/`0xFC`/`0xFD` are 3-byte unknown frames (next valid `0x8C` always appears at offset+3). Ruled out: brownouts, multi-resets, config-not-taking-effect. The 20 mA in "deep sleep" is **USB CDC peripheral overhead** (only present when USB is connected for monitoring; goes away on battery). **Fixes shipped:** explicit `pinMode/gpio_set_direction/gpio_set_pull_mode` for GPIO0 + live INT1 level log before sleep; parser now skips 2 bytes on unknown tag and continues instead of bailing. Reset reason name array extended (11=`USB`, 12=`JTAG`, etc). Expected: full ~200 rows for a 10s session. See [[ESP32-C6 Power Audit#✅ 2026-04-25 follow-up #8 — H1 + H3 confirmed; fixes shipped]].

> [!info] 2026-04-25 — partial data + missing INT1 wakes: diagnostic instrumentation shipped
> Fourth test: parser captured **real gyro values** for the first time (small win!) but only **7 rows for a 10-sec session**. Two mysteries: (1) `cause=7` GPIO INT1 wake never fires — only timer wakes, so we lose ~7 of 8 expected drain cycles; FIFO is in stream mode and overwrites most data before STOP. (2) Parser bails on `0xFD` / `0xFC` tags that aren't in any documented BMI270 FIFO tag set. Five hypotheses (H1: INT1 asserting but GPIO0 not picking it up; H2: INT1 not generated; H3: 0xFD is real undocumented tag; H4: brownouts; H5: multiple resets) — instrumentation added to test all five via `reveal`: `esp_reset_reason` + boot counter, `INT_STATUS_1` read on each wake, `FIFO_LEN` at wake, FIFO config readback, hex dump around unknown tags. Also shipped two brownout-mitigation tweaks (CPU to 80 MHz on wake — was 160; BLE TX P9 → P3 during check-ins). See [[ESP32-C6 Power Audit#🔬 2026-04-25 follow-up #7 — partial data + INT1 wake never fires: hypotheses + diagnostic instrumentation]].

> [!bug] 2026-04-25 — `unknown tag 0x88` + 22 mA session: ACC/GYR ODR mismatch
> Third test: GPIO hold worked (FIFO had 2015 B, BMI270 stayed alive), but parser bailed at `[DSL-fifo] unknown tag 0x88 at offset 15/2015` after 1 frame. Session captured 1 row total. Average session current was **22 mA** instead of the projected 7 mA. Root cause: ACC was at 100 Hz but GYR was at 200 Hz, so the BMI270 emitted a MIX of `0x8C` (combined, 100 Hz) and `0x88` (gyro-only, 100 Hz) frames — total 200 Hz frame rate. Parser bailed on `0x88`, decimator math would have been wrong anyway, and 2× watermark crossings made every wake near-empty. **Fix:** match GYR ODR to 100 Hz (`GYR_CONF` 0xE9 → 0xE8) so all frames are `0x8C`, plus a defensive `0x88` handler in case any leak through during ODR-change settling. **Bonus:** legacy `enterDeepSleep()` (button-hold path) now calls `BLEDevice::deinit(true)` before sleep — without that the Bluedroid stack stays alive during "deep sleep" burning ~10–15 mA. See [[ESP32-C6 Power Audit#🐛 2026-04-25 follow-up #6 — `unknown tag 0x88`: ACC/GYR ODR mismatch + 22 mA session current]].

> [!bug] 2026-04-25 — `FIFO had 0 B` after parser fixes: GPIO state lost across deep sleep
> Second test: `reveal` showed `cause=4` only (no GPIO INT1 wakes ever) and `FIFO had 0 B` after a 10-sec sleep. Diagnosis: ESP32-C6 doesn't preserve GPIO output state across deep sleep by default — D2 (sensor power gate) was floating during sleep, BMI270 was losing VDD entirely, FIFO contents lost, INT1 floats LOW so GPIO wake never fires. Fix: `gpio_hold_en((gpio_num_t)PIN_SENSOR_PWR)` in `enterLoggingDeepSleep`; released in `stopAndReplay` DSL cleanup. Also: `CtrlCB::onWrite` now tees through `DSL_LOG` so BLE commands (`START_LOG` / `STOP_LOG`) show in `reveal`. Required `#include "driver/gpio.h"`. **First build attempt also added `gpio_deep_sleep_hold_en()` — that function was removed in IDF 5.x; `gpio_hold_en` alone now handles both active and deep-sleep retention. Don't add back.** See [[ESP32-C6 Power Audit#🐛 2026-04-25 follow-up #5 — `FIFO had 0 B`: GPIO state lost across deep sleep]].

> [!bug] 2026-04-25 — first session got rows=0: two FIFO parser bugs fixed
> Session completed end-to-end (sleep, wake, check-in, reconnect, STOP all working) but the file had only the 18-byte CRSN header and no data chunks. Two bugs in the BMI270 FIFO header-mode parser: (1) **wrong frame tag** — I had `0xA0`, the canonical SensorAPI value is `0x8C` for gyr+acc combined frames, every drain bailed on the first byte; (2) **wrong byte order** — I had accel-first then gyro, datasheet says gyro-first then accel, so even after fixing the tag the `gy` column would have been logging accel-Y values. Both fixed. Source comments updated to cite the SensorAPI repo as canonical. Unknown tags now logged via `DSL_LOG` so future surprises surface in `reveal`. See [[ESP32-C6 Power Audit#🐛 2026-04-25 follow-up #4 — empty CSV: two FIFO parser bugs found and fixed]].

> [!success] 2026-04-25 — DSL working on hardware, phone reconnect tuned
> Deep sleep architecture confirmed working: multimeter shows 0 mA in sleep, ~45 mA bursts on FIFO drain. Estimated average **~7 mA during a logging session** with the tuned-down 2.5 sec check-in (was 46 mA — 6.5× improvement). Remaining issue was that the 2.5 sec check-in used the global slow adv interval (1000–1500 ms), giving the phone too few burst chances to reconnect for STOP_LOG. Fix: `runBleCheckIn` now overrides to **fast adv (100–150 ms)** during check-in only, and **window extended to 5 sec** for full GATT discovery time. New estimated average: **~11.5 mA** — still 4× better than baseline. See [[ESP32-C6 Power Audit#🎉 2026-04-25 follow-up #3 — DSL working, phone reconnect tuned]].

> [!bug] 2026-04-25 — first DSL test failed, deadlock fix + visibility tools shipped
> First on-board test: 65 mA after START_LOG (no deep sleep), STOP_LOG hung "syncing forever". Root cause: `BLEDevice::deinit()` called from inside `CtrlCB::onWrite` (Bluedroid task), which deadlocks the stack on itself. Fix: command handling deferred via `pendingStartLog` / `pendingStopLog` flags consumed in `loop()`, so heavy ops run from the Arduino task instead. Also added: **RTC-resident log ring buffer + `reveal` Serial command** so debug logs survive deep sleep — type `reveal` in Serial Monitor after a session to see everything the firmware did while disconnected. Also `clear` (reset buffer) and `state` (one-line state dump). Heavy `DSL_LOG()` instrumentation added throughout the DSL flow. See [[ESP32-C6 Power Audit#🔥 2026-04-25 follow-up #2 — first DSL test FAILED, root cause + fix shipped]].

---

## Stack

- **MCU module:** Seeed XIAO ESP32-C6 (RISC-V dual-core: 160 MHz HP + 20 MHz LP)
- **Daughter board:** Custom PCB (KiCad) with IMU + accelerometer + magnetometer + SPI flash + power gate
- **BLE only:** Bluetooth 5 (LE) via the Arduino BLE stack (Bluedroid). Wi-Fi is **explicitly disabled** at boot — see "Why WiFi is force-off" below.
- **Filesystem:** LittleFS on internal flash (the W25Q128JV on the daughter board is wired but not used)
- **Compression:** DeltaPack second-difference predictor + bit-packing. Gzip disabled (`USE_GZIP=0`).
- **Library dependency:** [SparkFun BMI270 Arduino Library](https://github.com/sparkfun/SparkFun_BMI270_Arduino_Library) — but **only for its 8 KB `bmi270_config_file[]` blob**. The wrapper's init path doesn't work on ESP32-C6 (see below); we do the init manually.

---

## Daughter Board Schematic (real_print.pdf)

### Components

| Ref | Part           | Purpose                              | Bus    | I2C Addr |
| --- | -------------- | ------------------------------------ | ------ | -------- |
| U1  | BMA400         | Low-power accelerometer              | I2C    | 0x14     |
| U4  | BMI270         | 6-axis IMU (gyro + accel)            | I2C    | 0x68     |
| U3  | MMC5603NJ      | 3-axis magnetometer                  | I2C    | 0x30     |
| U2  | W25Q128JV      | 16 MB SPI NOR flash                  | SPI    | —        |
| Q2  | NTS2101PT1G    | P-MOSFET power gate (sensor VDD rail)| —      | —        |
| Q1  | MMST3904       | NPN gate driver for Q2               | —      | —        |
| SW1 | Push button    | User button (active HIGH)            | —      | —        |
| R6  | 2.7K           | I2C pull-up on SCL                   | —      | —        |
| R3  | 2.7K           | I2C pull-up on SDA                   | —      | —        |
| R4  | 30K            | Q2 gate pull-up to 3V3               | —      | —        |
| R7  | 1K             | Q1 base resistor                     | —      | —        |
| R8  | 2.7K           | Button pull-up to 3V3                | —      | —        |

### Power Architecture

```
XIAO 3V3 ──► Q2 (NTS2101PT1G P-MOSFET) Source ──► Drain = sensor VDD rail
                ▲ Gate controlled by Q1 (MMST3904 NPN)
                │   Q1 Base ◄── D2_GPIO2 via R7 (1K)
                │   Q1 Collector ──► Q2 Gate
                │   R4 (30K) pulls Q2 gate to 3V3 when Q1 off
                │
   D2 HIGH → Q1 on  → Q2 gate LOW  → VDD ON  (sensors powered)
   D2 LOW  → Q1 off → Q2 gate HIGH → VDD OFF (zero current draw)
```

This lets the firmware fully power-cycle every sensor on the daughter board with one GPIO. The current firmware uses this to recover from a stuck BMI270 init (see retry path below); a future revision will reuse it for deep-sleep zero-drain mode.

### Pin Mapping (XIAO → daughter board)

| XIAO Pin     | GPIO | Connects To                                      | Used by firmware |
| ------------ | ---- | ------------------------------------------------ | ---------------- |
| D0 / A0      | 0    | BMA400 INT1 + BMI270 INT1 (shared)              | No (polling)     |
| D1           | 1    | SW1 push button (active HIGH)                    | Yes (hold-to-sleep + wake source) |
| D2 / A2      | 2    | Sensor power gate (Q1 base)                      | Yes (HIGH at boot, LOW for power-cycle retries) |
| D4 / SDA     | 22   | I2C SDA (BMA400, BMI270, MMC5603)                | Yes |
| D5 / SCL     | 23   | I2C SCL (BMA400, BMI270, MMC5603)                | Yes |
| D7 / RX      | 17   | W25Q128 CS#                                      | No (LittleFS on internal flash) |
| D8 / SCK     | 19   | W25Q128 SCLK                                     | No |
| D9 / MISO    | 20   | W25Q128 SO/SIO1                                  | No |
| D10 / MOSI   | 18   | W25Q128 SI/SIO0                                  | No |
| —            | 15   | Built-in user LED (active LOW)                   | Yes |
| 3V3 / GND    | —    | Q2 source + decoupling caps / common ground      | — |

### J1 Daughter Board Header (14-pin)

| Pin | Signal          | Pin | Signal             |
| --- | --------------- | --- | ------------------ |
| 1   | D0_GPIO0 (INT)  | 9   | D8_GPIO19 (SCK)    |
| 2   | D1_GPIO1 (BTN)  | 10  | D9_GPIO20 (MISO)   |
| 3   | D2_GPIO2 (PWR)  | 11  | D10_GPIO18 (MOSI)  |
| 4   | NC              | 12  | 3V3                |
| 5   | D4_GPIO22 (SDA) | 13  | GND                |
| 6   | D5_GPIO23 (SCL) | 14  | NC                 |
| 7   | NC              |     |                    |
| 8   | D7_GPIO17 (CS#) |     |                    |

---

## Firmware overview (shinpad_c6.ino)

### What runs end-to-end

1. **`setup()`** — kills WiFi, powers the sensor rail (D2 HIGH), brings up I2C at 50 kHz, mounts LittleFS, scans the I2C bus, probes MMC + BMA, runs the BMI270 manual init (with up to 3 power-cycle retries), falls back to BMA400 if BMI270 won't come up, inits MMC5603, bumps I2C to 400 kHz, calibrates a gyro-Y bias over ~400 ms, then starts BLE advertising as `ShinPad_<SHINPAD_UNIT_NUMBER>`.
2. **`loop()`** — services the button (1.5 s hold → deep sleep), updates the LED, handles deferred BLE re-advertise, runs the 20 Hz logging tick when `isLogging`, and the BLE replay pump when `isReplaying`.
3. **BLE control** — phone writes `START_LOG` / `STOP_LOG` to the CONTROL characteristic; firmware notifies `READY` / `LOGGING` / `FLUSHING` / `DONE:<rows>` back. Session bytes stream over the DATA characteristic in 244-byte notify packets at ~4 ms pacing.
4. **Per-unit naming** — `#define SHINPAD_UNIT_NUMBER 1` at the top of the .ino. Change per board so every physical pad has a unique advertised name. The DEVICE_ID characteristic returns the same string for app-side auto-heal.

### Sample format (logged columns)

The firmware now writes **6 columns per sample at 20 Hz**, in this exact order:

| Idx | Field | Source                                    | Filter / scaling |
| --- | ----- | ----------------------------------------- | ---------------- |
| 0   | gy    | BMI270 gyro-Y (or BMA400 accel-Y fallback)| **bias-corrected + 1st-order LPF (75% prev / 25% new)** — drives the app's sprint/peak detection |
| 1   | gx    | BMI270 gyro-X                             | raw int16        |
| 2   | gz    | BMI270 gyro-Z                             | raw int16        |
| 3   | mx    | MMC5603NJ X                               | zero-centered (raw is uint16 around 0x8000) |
| 4   | my    | MMC5603NJ Y                               | zero-centered |
| 5   | mz    | MMC5603NJ Z                               | zero-centered |

> [!important] gy stays at column 0
> The app reads `cols` from the CRSN file header and maps column index → field name via `getCsvHeader()`. Keeping `gy` at index 0 means `parseCSVDetailed` finds it via `headers.indexOf('gy')` unchanged, so [[StatsEngine Cross-Platform|StatsEngine]] keeps working without any edits. The session file still ends in `DONE:<rows>` and the binary container ([[03 - Data Pipeline]]) is byte-for-byte compatible with the Nordic / ESP32-S3 firmware.

The CRSN file header at the start of `/session.bin` records `cols=6` so the decoder picks up all six channels. ChunkHeader is unchanged — `predictor=2` (second-difference), CRC32 over the bit-packed payload, optional gzip stage gated by `USE_GZIP`.

---

## Why-it-works: BMI270 manual init on ESP32-C6

> [!danger] Don't switch to the SparkFun wrapper's `imu.beginI2C()` path
> The SparkFun BMI270 Arduino Library does work on most ESP32 cores, but **fails consistently on ESP32-C6** with `BMI2_E_COM_FAIL` (rc=-2) or `BMI2_E_SET_APS_FAIL` (rc=-13). The C6 I2C peripheral mishandles the wrapper's STOP+START access pattern after a soft reset, and there is no error visibility through the wrapper. Three things had to be true at once for init to succeed:

1. **`Wire.setBufferSize(256)` BEFORE `Wire.begin()`** — the BMI270 init requires uploading an 8 KB config file in 32-byte chunks. The Arduino I2C TX buffer defaults to 128 bytes; any chunk that exceeds it is silently truncated and the init verification fails every time.
2. **`Wire.setTimeOut(1000)` (default 50 ms)** — after soft reset the BMI270 wakes from APS mode and clock-stretches longer than the default I2C timeout allows. This was the actual root cause of `BMI2_E_COM_FAIL` returns from the library on C6.
3. **50 kHz during init, 400 kHz after** — slower I2C is more forgiving when the chip is clock-stretching during post-reset wake-up. The C6 I2C driver is picky about this. Bumped to 400 kHz right after `setupBMI270()` returns and before BLE init.

### The manual init sequence (`setupBMI270()`)

We bypass the wrapper entirely and use known-working `i2cReadByte` / `i2cWriteByteRetry` helpers. We still link against the SparkFun library purely for its `bmi270_config_file[]` blob (8192 bytes), accessed via `extern "C" const uint8_t bmi270_config_file[]`.

| Step | Action | Why |
| ---- | ------ | --- |
| 1 | Read CHIP_ID (0x00), expect 0x24 | Sanity check before going further |
| 2 | Soft reset: write 0xB6 to CMD (0x7E) | `i2cWriteByteRetry(8 tries)` because in post-reset APS mode the chip sometimes NACKs the first writes while its internal clock is still ramping |
| 3 | Wait 20 ms (>> datasheet 450 µs) | First post-reset access is flaky on some parts; generous delay |
| 4 | Disable advance power save: write 0 to PWR_CONF (0x7C) | **Retry loop is the key fix.** SparkFun does a single attempt and dies with `BMI2_E_SET_APS_FAIL` when the APS clock isn't fully up |
| 5 | Enter config-upload mode: clear bit 0 of INIT_CTRL (0x59) | — |
| 6 | Upload 8 KB blob in 32-byte chunks | INIT_ADDR_0/_1 take the *half-word* index (i/2) — low nibble in _0, upper 8 bits in _1. Burst-write data to INIT_DATA (0x5E) |
| 7 | Exit config mode (set bit 0 of INIT_CTRL), wait 25 ms | Datasheet says 20 ms |
| 8 | Verify INTERNAL_STATUS (0x21) low nibble == 1 | Anything else = config didn't load |
| 9 | Configure accel/gyro/power | ACC_CONF=0xA8 (±8 g, 100 Hz, normal+perf), ACC_RANGE=0x02; GYR_CONF=0xE9 (±2000 dps, 200 Hz, normal+perf+noise_perf), GYR_RANGE=0x00; PWR_CTRL=0x06 (gyro+accel on) |

### Retry strategy

`setup()` tries `setupBMI270()` up to 3 times. Between attempts it calls `powerCycleSensors()` which:

- Ends `Wire` cleanly (releases the driver / preserves the buffer-size config)
- Drives D2 LOW (Q2 gate HIGH → VDD OFF), waits 80 ms
- **Floats SDA and SCL while powered down** so the chip isn't back-fed through the 2.7 K pull-ups (otherwise the VDD cycle doesn't actually reset the chip's internal state)
- Drives D2 HIGH again, waits 150 ms for the rail + bulk caps to settle
- Re-runs `Wire.setBufferSize(256)` then `Wire.begin()`, 50 kHz, 1 s timeout

If all three BMI270 attempts fail, the firmware falls back to BMA400 (with a fresh power-cycle in between, because failed BMI270 attempts leave the I2C bus in an unknown state — we observed BMA400 returning `0x24` instead of its real `0x90` chip ID until VDD was cycled).

---

## Why-it-works: BMA400 fallback (accel-only)

When BMI270 won't come up, `setupBMA400()` initializes the BMA400 at ±8 g / 200 Hz / normal mode (`ACC_CONFIG1=0x9A`, `ACC_CONFIG0=0x02`). Reads return 12-bit signed values across two bytes per axis with the sign in the low 4 bits of the MSB.

To keep the app's peak-detection thresholds tuned (they were calibrated against the old ICM-42688 firmware at 4096 counts/g), the BMA400's 12-bit values are multiplied by 16 to rescale to the same int16 range. Since BMA400 has no gyro, **accel-Y is mapped into the `gy` slot** in `readIMURaw()` so kicks/movements still produce visible peaks at column 0.

This is graceful degradation, not a primary path — `imuKind` records which sensor came up so future firmware can adjust analytics if needed.

---

## Why-it-works: MMC5603NJ ping-pong reads

The magnetometer is now fully functional (was a TODO in the previous firmware revision). Driver pattern:

- **Init** (`setupMMC5603`): verify PRODUCT_ID at 0x39 == 0x10, software reset via IC1 bit 7, wait 25 ms, kick off the first measurement (IC0 bit 0 = TM_M).
- **Per-sample flow**: at sample N we **read** whatever measurement was triggered during sample N-1, then **immediately trigger** the next one. A 16-bit MMC5603 measurement completes in ~2 ms — far less than the 50 ms sample interval — so we never have to block waiting on the chip.
- **Output format**: raw chip output is unsigned 16-bit centered at 0x8000. We subtract the bias to hand the app values symmetric around zero (signed int16). Big-endian byte order on the wire (xH xL yH yL zH zL).

If MMC init fails, `mmcReady=false` and the mag columns are written as zero — the app decodes them either way, so this is a non-fatal degradation.

---

## Why-it-works: ESP32-C6 BLE quirks

These are workarounds for bugs / tight spots in the C6 Arduino BLE stack (Bluedroid):

| Quirk | Workaround | Why |
| ----- | ---------- | --- |
| WiFi shares antenna with BLE on C6 | `WiFi.mode(WIFI_OFF); esp_wifi_stop(); esp_wifi_deinit();` at start of `setup()` | Even silent WiFi (no `WiFi.begin()`) can be powered on by the Arduino core and steals airtime → produces the "connects, drops, reconnects" symptom |
| CCCD write callback fires unreliably | Set `notifReady=true` and `dataNotifReady=true` on `onConnect()` regardless of whether the descriptor write actually came through | `pCtrl->notify()` and `pData->notify()` are no-ops if no central is actually subscribed, so this is safe — and it prevents the "phone subscribed but firmware doesn't believe it" deadlock |
| Synchronous adv-restart inside `onDisconnect()` is silently dropped | `needReadvertise = true; readvertiseAtMs = millis() + 120` then call `BLEDevice::startAdvertising()` from `loop()` after the settle | Was the cause of "board randomly stops advertising after a few disconnects". GAP needs time to clean up the old link first |
| Adv interval is a power vs. reconnect-latency knob | **Single-mode 1000–1500 ms** (`setMinInterval(0x640)` / `setMaxInterval(0x960)`, units 0.625 ms). **Not** the C6 default (~1.28 s) and **not** the previous 100–150 ms tuning | Use pattern is "phone is far away for ~95% of every session" → fast adv just burns radio time for nobody. Slow adv saves ~9 mA average. ~1.5 s reconnect at end of game is invisible to the coach. See [[ESP32-C6 Power Audit#⚡ 2026-04-25 — first optimization pass shipped]] |
| Default TX power is mid-level | `BLEDevice::setPower(ESP_PWR_LVL_P9)` (max) | Reduces dropouts when phone is at edge of range (sock pocket, bag, etc.) |
| Replay used to gate on `dataNotifReady` | Always run the replay pump once `isReplaying` is set | `pData->notify()` is a no-op if nothing's subscribed; gating on the unreliable CCCD flag was causing "STOP shows DONE:0" timeouts |
| In-flight session can outlive the connection | `onDisconnect` closes `dataFile` and clears `isReplaying` | Otherwise the loop would spam `notify()` forever against a dead link |

Other BLE choices baked in:

- `BLEDevice::setMTU(247)` → 244-byte chunks fit within MTU - ATT overhead. Directly drives the `txBuf[244]` size in the replay pump.
- 4 ms `delay()` between notify chunks: enough back-pressure that the BLE stack still services connection events promptly, but doesn't choke throughput.
- Scan response payload re-sets the device name explicitly so iOS always sees the full `ShinPad_<n>` string regardless of adv-packet name truncation.
- No bonding / no encryption — pairing is managed by the app via Firestore. Skipping `BLESecurity` also avoids Bluedroid-only constants that aren't all exposed on the C6 Arduino core.

---

## LED behavior

The XIAO ESP32-C6 has only a single-color built-in LED on GPIO15 (active LOW). The current firmware uses a deliberately simple scheme:

| State                        | LED                              |
| ---------------------------- | -------------------------------- |
| Awake, NOT on USB            | Solid ON                         |
| Awake AND on USB (charging proxy) | 1 Hz blink (500 ms on / 500 ms off) |
| Logging / replaying          | (unchanged from base awake state) |
| Deep sleep                   | OFF                              |

"Charging" is approximated by USB host presence: `(bool)Serial` is truthy when the host has opened the CDC port. There's no charge-status pin exposed to the ESP32 on this XIAO module.

> [!info] Future: NeoPixel option
> If a WS2812 is added to a future daughter-board revision, the LED code can be swapped to use `neopixelWrite()` for the full RGB battery / state scheme used by the [[ESP32-S3 SmallBoard Firmware]].

---

## Deep sleep

- **Trigger**: hold SW1 for 1.5 s (`BTN_HOLD_MS`) while awake → `enterDeepSleep()`.
- **Guards**: never sleeps mid-session (`if (isLogging || isReplaying) return`). Closes any open file, ends LittleFS, turns LED off, waits for button release (so we don't immediately re-wake), debounces, drives D2 LOW (cuts sensor rail), floats SDA/SCL.
- **Wake source**: `esp_deep_sleep_enable_gpio_wakeup(1ULL << PIN_BTN, ESP_GPIO_WAKEUP_GPIO_HIGH)` — this is the **deep-sleep** API (D1/GPIO1 is LP-IO capable on C6). The light-sleep API used to be wired here by mistake, which left the chip unable to wake at all once it slept.
- **Auto-sleep on idle is not currently wired**. The button hold is the only path into deep sleep.

---

## Firmware status

| Feature                          | Status         | Notes                                             |
| -------------------------------- | -------------- | ------------------------------------------------- |
| BLE GATT service (8 chars)       | **Working**    | All UUIDs match, RN + iOS apps connect           |
| BLE commands (START_LOG/STOP_LOG)| **Working**    | Full session lifecycle                            |
| Session state machine            | **Working**    | idle → logging → flushing → idle                  |
| BMI270 6-axis IMU (gyro + accel) | **Working**    | Manual init, 3 retries with sensor power-cycle   |
| BMA400 fallback                  | **Working**    | Accel-only mode if BMI270 won't init              |
| MMC5603NJ magnetometer           | **Working**    | Ping-pong single-shot reads, 6-channel logging    |
| LittleFS session storage         | **Working**    | Internal flash; W25Q128JV unused                  |
| DeltaPack compression codec      | **Working**    | `predictor=2`, identical to ESP32-S3              |
| Data replay over BLE             | **Working**    | 244-byte chunks, 4 ms pacing                      |
| Device ID characteristic         | **Working**    | `ShinPad_<SHINPAD_UNIT_NUMBER>` from compile-time const |
| Per-unit advertised name         | **Working**    | Set via `#define SHINPAD_UNIT_NUMBER`             |
| Deep sleep + button wake         | **Working**    | 1.5 s hold to enter; D1 GPIO HIGH wakes           |
| LED status indicator             | **Working**    | Solid awake / 1 Hz blink on USB                   |
| Sensor power gate (Q2)           | **Working**    | D2 HIGH = ON, used during BMI270 retries          |
| I2C bus scan + per-sensor probe  | **Working**    | Diagnostic at boot                                |
| WiFi (any mode)                  | **Disabled**   | Force-off at boot to free up the BLE radio        |
| CPU frequency scaling            | **Applied**    | `setCpuFrequencyMhz(80)` at boot — ~10–15 mA saved |
| Auto light-sleep                 | **Applied**    | `esp_pm_configure(max=80, min=40, light_sleep=true)` — ~10 mA saved (multiplier on idle delay) |
| Idle-aware loop pacing           | **Applied**    | `delay(40)` when not logging/replaying, `delay(1)` otherwise — multiplies the light-sleep win |
| Slow advertising (1000–1500 ms)  | **Applied**    | Single-mode. ~9 mA saved. ~1.5 s reconnect latency is acceptable per use pattern |
| Battery ADC                      | **Stubbed**    | Returns 100 % / 4.0 V — no BAT pin on XIAO C6     |
| W25Q128JV external SPI flash     | **Unused**     | Wired but firmware uses internal flash            |
| Gzip compression                 | **Disabled**   | `USE_GZIP=0`. miniz files stashed in `Rishab_PCB_esp32c6_libs/miniz/`; copy in + flip flag to enable. Needs Huge APP partition. |
| Auto-sleep on idle               | **Not wired**  | Only manual hold-to-sleep                         |
| Low-battery cutoff               | **Not wired**  | Needs battery ADC first                           |

---

## TODOs

1. **Battery ADC** — XIAO ESP32-C6 has no BAT pin in `pins_arduino.h`. To enable real battery readings, add a voltage divider on the daughter board to a spare ADC pin (e.g. D3/GPIO21) and wire `readBatteryMillivolts()`. Required before the [[01 - Critical Preservation Rules|3.3 V undervoltage rule]] can be enforced.
2. **Optional W25Q128JV session storage** — driver from previous firmware revision can be ported in. Would lift the session size cap from ~2 MB (internal LittleFS) to 16 MB.
3. **IMU interrupt-driven sampling** — both BMI270 and BMA400 INT1 are wired to GPIO0. Could drive a hardware interrupt for more precise 20 Hz timing instead of polling.
4. **BMA400 as low-power wake sensor** — BMA400 can run at ~5 µA in motion-detect mode while BMI270 is powered down. Useful for an idle wake-on-shake.
5. **Enable gzip compression** — copy `miniz*.{c,h}` into the sketch folder, set `#define USE_GZIP 1`, switch to **Huge APP (3MB No OTA)** partition. App decoders already accept both codecs.
6. **Power optimization pass** — the previous firmware revision had `setCpuFrequencyMhz(80)`, `esp_pm_configure(...light_sleep_enable=true...)`, longer idle `delay()`, and longer adv intervals. Not wired in `shinpad_c6.ino` yet — re-add once the core feature set is locked. See [[ESP32-C6 Power Audit]] for the full ranked list.
7. **Validate deep-sleep wake on hardware** — API has been corrected (`esp_deep_sleep_enable_gpio_wakeup`), needs a real PPK2 / multimeter pass to confirm the chip actually wakes from button press and that deep-sleep current is in the expected µA range.

---

## Known board-level gotchas (carry forward to next PCB rev)

> [!danger] No low-voltage protection on XIAO C6
> The XIAO charging IC does NOT include LiPo undervoltage cutoff. A deep-discharged cell can be permanently damaged. Firmware must enforce a 3.3 V minimum once a battery ADC is wired ([[01 - Critical Preservation Rules]]).

> [!warning] Charge LED drains battery 24/7
> The on-board charge LED is hardwired and draws a few mA whenever USB is present. Route it through a GPIO on the next PCB revision.

> [!warning] Reports of XIAO C6 boards damaged during charging
> Multiple Seeed forum reports of power-path issues during USB charging. Consider adding external TVS / soft-start on next PCB rev.

### Sources

- [Seeed PPK2 deep sleep thread (15 µA baseline)](https://forum.seeedstudio.com/t/15ua-with-xiao-esp32c6-with-ppk2-while-sleeping-basic-example/276412)
- [Seeed forum — no undervoltage protection](https://forum.seeedstudio.com/t/does-xiao-esp32-c6-have-low-voltage-protection-for-li-ion-battery/292538)
- [Seeed forum — boards frying during charge](https://forum.seeedstudio.com/t/xiao-esp32c6-keeps-frying-when-charging/283403)

---

## Key differences from ESP32-S3 SmallBoard

| Aspect            | ESP32-S3 SmallBoard              | XIAO ESP32-C6 + Rishab daughter board |
| ----------------- | -------------------------------- | -------------------------------------- |
| CPU               | Xtensa LX7 (dual-core, 240 MHz) | RISC-V (160 MHz HP + 20 MHz LP)       |
| RAM               | 8 MB PSRAM                       | 512 KB SRAM (no PSRAM)                |
| Wi-Fi             | Wi-Fi 4 (802.11n) — used for upload | Capable of Wi-Fi 6, but **disabled** in firmware (antenna shared with BLE) |
| Extra radios      | None                             | 802.15.4 (Thread/Zigbee/Matter)       |
| IMU               | ICM-42688 / QMI8658 / LSM6DS    | BMI270 (6-axis) + BMA400 fallback      |
| Magnetometer      | None                             | MMC5603NJ (in pipeline now)            |
| Flash storage     | W25Q128 16 MB SPI (used)         | W25Q128JV 16 MB SPI (wired but unused — internal LittleFS instead) |
| Fuel gauge        | MAX17048                         | None — needs daughter-board divider    |
| Charger IC        | BQ24075 (GPIO-controlled)        | XIAO built-in LiPo charge circuit (no GPIO control, no UVLO) |
| Power gating      | SYSOFF pin                       | P-MOSFET + NPN gate driver (D2 GPIO2)  |
| LED               | NeoPixel (RGB)                   | Single-color built-in (GPIO15, active-LOW) |
| Form factor       | Full custom PCB                  | COTS XIAO + custom daughter board      |
| BMI270 init path  | (not used)                       | Manual — SparkFun wrapper fails on C6 |

---

## Future possibilities (ESP32-C6 unique)

- **Thread / Zigbee / Matter:** mesh networking between multiple ShinPads on a pitch
- **Wi-Fi 6:** lower-latency uploads (would need an antenna-share strategy with BLE)
- **LP core (20 MHz RISC-V):** could handle IMU sampling while main core sleeps
- **BMI270 + BMA400 fusion:** BMI270 for high-accuracy 6-axis, BMA400 as always-on wake sensor
- **MMC5603NJ heading:** add compass heading for player orientation on the pitch (mag now logged, just needs analytics support)

---

## Related

- [[ESP32-C6 Power Audit]] — where current is going on this firmware, ranked optimization plan
- [[Proposal - ESP-NOW Mesh Gateway]] — pad-to-pad ESP-NOW mesh proposal (exploring)
- [[BLE Protocol]] — frozen GATT contract
- [[Session State Machine]] — idle → logging → flushing
- [[ESP32-S3 SmallBoard Firmware]] — sibling firmware (ICM-42688 based)
- [[Nordic BLE Firmware (nRF52811)]] — original firmware
- [[Hardware Overview]] — cross-board comparison
- [[StatsEngine Cross-Platform]] — gy at column 0 keeps this unchanged
- [[03 - Data Pipeline]] — how sensor data flows to Firebase
