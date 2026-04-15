---
title: ESP32-C6 XIAO Firmware (Rishab PCB)
type: project
tags:
  - project/esp-c6
  - firmware
  - hardware
  - critical
created: 2026-04-14
status: active
location: "Rishab_PCB_esp32c6/"
---

# ESP32-C6 XIAO Firmware (Rishab PCB)

A third ShinPad firmware track targeting the **Seeed XIAO ESP32-C6** mounted on a custom **daughter board designed by Rishab** (KiCad: `daughter_kicad.sch`). Same BLE GATT contract as the [[Nordic BLE Firmware (nRF52811)|Nordic firmware]] and [[ESP32-S3 SmallBoard Firmware|ESP32-S3 firmware]].

> [!info] Three parallel firmware tracks
> Cresento now has THREE chip designs. This one uses a COTS XIAO ESP32-C6 module on a custom sensor carrier board, making it cheaper and easier to assemble than the full-custom ESP32-S3 PCB.

---

## Stack

- **MCU module:** Seeed XIAO ESP32-C6 (RISC-V dual-core: 160 MHz HP + 20 MHz LP)
- **Daughter board:** Custom PCB (KiCad) with IMU, accelerometer, magnetometer, SPI flash, power gate
- **BLE:** Bluetooth 5 (LE) via Arduino BLE
- **Wi-Fi:** Wi-Fi 6 (802.11ax) for fast-upload mode
- **Filesystem:** LittleFS (on internal flash)
- **Compression:** DeltaPack bit-packing (gzip disabled by default, less RAM than ESP32-S3)

---

## Daughter Board Schematic (real_print.pdf)

### Components

| Ref | Part           | Purpose                              | Bus    | I2C Addr |
| --- | -------------- | ------------------------------------ | ------ | -------- |
| U1  | BMA400         | Low-power accelerometer              | I2C    | 0x14     |
| U4  | BMI270         | 6-axis IMU (gyro + accel)            | I2C    | 0x68     |
| U3  | MMC5603NJ      | 3-axis magnetometer                  | I2C    | 0x30     |
| U2  | W25Q128JV      | 16 MB SPI NOR flash                  | SPI    | —        |
| Q2  | NTS2101PT1G    | P-MOSFET power gate (VDD rail)       | —      | —        |
| Q1  | MMST3904       | NPN gate driver for Q2               | —      | —        |
| SW1 | Push button    | User button (active HIGH)            | —      | —        |
| R6  | 2.7K           | I2C pull-up on SCL                   | —      | —        |
| R3  | 2.7K           | I2C pull-up on SDA                   | —      | —        |
| R4  | 30K            | Q2 gate pull-up to 3V3               | —      | —        |
| R7  | 1K             | Q1 base resistor                     | —      | —        |
| R8  | 2.7K           | Button pull-up to 3V3                | —      | —        |

### Power Architecture

```
XIAO 3V3 ──► Q2 (NTS2101PT1G P-MOSFET) Source ──► Drain = VDD rail
                ▲ Gate controlled by Q1 (MMST3904 NPN)
                │   Q1 Base ◄── D2_GPIO2 via R7 (1K)
                │   Q1 Collector ──► Q2 Gate
                │   R4 (30K) pulls Q2 gate to 3V3 when Q1 off
                │
   D2 HIGH → Q1 on → Q2 gate LOW → VDD ON (sensors powered)
   D2 LOW  → Q1 off → Q2 gate HIGH → VDD OFF (zero current draw)
```

This allows the ESP32-C6 to completely cut power to ALL sensors for deep sleep.

### Pin Mapping

| XIAO Pin        | GPIO | Connects To                    |
| --------------- | ---- | ------------------------------ |
| D0 / A0         | 0    | BMA400 INT1 + BMI270 INT1     |
| D1              | 1    | SW1 push button (active HIGH)  |
| D2 / A2         | 2    | Sensor power gate (Q1 base)    |
| D4 / SDA        | 22   | I2C SDA (BMA400, BMI270, MMC5603) |
| D5 / SCL        | 23   | I2C SCL (BMA400, BMI270, MMC5603) |
| D7 / RX         | 17   | W25Q128 CS#                    |
| D8 / SCK        | 19   | W25Q128 SCLK                   |
| D9 / MISO       | 20   | W25Q128 SO/SIO1                |
| D10 / MOSI      | 18   | W25Q128 SI/SIO0                |
| —               | 15   | Built-in LED (active LOW)      |
| 3V3             | —    | Q2 source + decoupling caps     |
| GND             | —    | Common ground                   |

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

## Firmware Status

| Feature                          | Status       | Notes                                             |
| -------------------------------- | ------------ | ------------------------------------------------- |
| BLE GATT service (8 chars)       | **Working**  | All UUIDs match, apps connect                     |
| BLE commands (START/STOP/etc.)   | **Working**  | Full command set incl WiFi upload                  |
| Session state machine            | **Working**  | idle -> logging -> flushing -> idle               |
| DeltaPack compression codec      | **Working**  | Identical bit-packing to ESP32-S3                  |
| Data replay over BLE             | **Working**  | 244-byte chunks, 8ms pacing                        |
| WiFi fast-upload (Wi-Fi 6)       | **Working**  | HTTP POST to gateway:8080/upload                   |
| Device ID characteristic         | **Working**  | ShinPad_XXXXXX from MAC                            |
| Deep sleep + GPIO wake           | **Working**  | GPIO1 button wake, sensors powered off via Q2      |
| LED status indicators            | **Working**  | PWM brightness mirrors a_b_c.ino NeoPixel scheme (see below) |
| Sensor power gate (Q2)           | **Working**  | D2 HIGH = VDD on, LOW = zero drain                 |
| I2C bus scan                     | **Working**  | Probes all 3 sensors on connect                    |
| BMI270 6-axis IMU                | **Partial**  | Accel works, gyro needs config blob upload          |
| BMA400 accelerometer             | **Init OK**  | Driver written, not used in main pipeline           |
| MMC5603NJ magnetometer           | **Init OK**  | Init written, read function TODO                    |
| W25Q128JV SPI flash              | **Init OK**  | JEDEC ID read works, full R/W driver written        |
| Battery ADC                      | **Stubbed**  | TODO: read XIAO internal BAT pin                   |
| Gzip compression                 | **Disabled** | `USE_GZIP=0` — miniz stashed in `../Rishab_PCB_esp32c6_libs/miniz/`, copy into sketch + set flag to enable. Needs Huge APP partition. |

---

## LED behavior (maps a_b_c.ino NeoPixel to single-color PWM)

The XIAO ESP32-C6 has only a single-color built-in LED on GPIO15 (active LOW). The firmware uses 8-bit PWM brightness to mirror what the ESP32-S3 does with NeoPixel RGB colors, and the same 30ms/+6 breathing animation for charging.

| State            | ESP32-S3 (NeoPixel RGB)        | XIAO ESP32-C6 (PWM brightness) |
| ---------------- | ------------------------------ | ------------------------------- |
| Battery >= 70%   | Solid green                    | Solid bright (255)              |
| Battery 30–69%   | Solid yellow                   | Medium brightness (100)         |
| Battery < 30%    | Solid red                      | Dim (30) + fast blink (200ms)   |
| Charging         | Greenish breathing (fade 0–255, +6 every 30ms) | Same breathing animation on PWM |
| Logging          | —                              | Fast blink (250ms)              |
| Flushing/replay  | —                              | Very fast blink (100ms)         |
| BLE advertising  | —                              | Slow blink (1s)                 |
| BLE connected    | —                              | Shows battery level brightness  |

> [!info] If a NeoPixel/WS2812 is added to the daughter board in the future, the LED code can be swapped to use `neopixelWrite()` for full RGB like the ESP32-S3.

---

## Battery & Power (real-world numbers)

### Datasheet vs reality

| Mode | Seeed spec | Real community measurements |
|------|-----------|----------------------------|
| Deep sleep | **15 µA** | **37–50 µA typical**, up to 299 µA with poor config |
| BLE advertising | not specified | ~5–15 mA (interval dependent) |
| Matter / active Wi-Fi | not specified | **63 mA avg, 650 mA peaks** (Tomas McGuinness) |
| Active CPU + BLE idle | — | ~30–40 mA |

The gap between spec and reality comes from:
- Charging IC quiescent current (burns even when not charging)
- LDO leakage
- Battery divider / peripherals that stay live
- Arduino core background tasks

### Known board-level gotchas

> [!danger] No low-voltage protection
> The XIAO ESP32-C6 charging IC does **NOT** include LiPo undervoltage cutoff. A deep-discharged cell can be permanently damaged. **Firmware MUST enforce a 3.3 V minimum** — transition to deep sleep when `battMv < 3300`. Rule from [[01 - Critical Preservation Rules]] applies to this board too.

> [!warning] Charge LED drains battery 24/7
> The on-board charge LED is hardwired. It draws a few mA constantly whenever USB is present. For a future PCB revision, route the LED through a GPIO like the ESP32-S3 board does.

> [!warning] Known frying during charging
> Multiple Seeed forum reports of XIAO ESP32-C6 boards being damaged during USB charging. Power-path design issue. Consider adding external TVS / soft-start on next PCB rev.

### Battery budget for Cresento (300 mAh LiPo assumption)

| Activity | Current | Runtime |
|----------|---------|---------|
| Deep sleep (well-configured) | ~50 µA | **~250 days** |
| Deep sleep (default Arduino) | ~100–300 µA | 42–125 days |
| BLE advertising idle | ~10 mA | ~30 hours |
| Active idle (our current firmware, April 2026) | **~40–50 mA** (too high — see Power Audit below) | ~6–7 hours |
| Recording (IMU + BLE + flash writes) | ~40 mA expected | ~7.5 hours |
| Mesh gateway active | ~130 mA | ~2.3 hours |
| Wi-Fi upload burst | ~63 mA avg, 650 mA peaks | ~4.8 hours sustained |

**Per match (90 min + 3 min flush):** ~65 mAh → **~3–4 matches per charge on 300 mAh, ~5–6 on 500 mAh.**

Seeed themselves recommend **≥500 mAh** if Wi-Fi is used — smaller cells sag on the 650 mA peaks and recover poorly.

### Real measurement — April 2026 (bare XIAO, no daughter board yet)

> [!info] Test conditions
> Measurements below are from a **bare XIAO ESP32-C6 module**, **NO** daughter board attached yet. So BMI270 / BMA400 / MMC5603 / W25Q128 / SW1 / Q2 MOSFET / R3/R6/R8 etc. are all physically absent. The 50 mA drain is coming entirely from the XIAO module itself.

**Observed (pre-fix):** 200 mAh LiPo dropped 3.86 V → 3.42 V in 2 hours.
- That's ~50% of capacity, or **~50 mA average draw**
- Equivalent to fully-active mode with no power saving whatsoever
- Target for idle should be ~5–10 mA

### Power Audit (April 2026) — bare XIAO culprits

| Power drain | Estimate | Status |
|-------------|---------|--------|
| CPU at 160 MHz with busy loop | ~20 mA | **Fixed** — dropped to 80 MHz via `setCpuFrequencyMhz(80)` |
| BLE advertising at default ~20 ms interval | ~15 mA | **Fixed** — `setMinInterval(1600)`/`setMaxInterval(2400)` = 1000–1500 ms |
| No light sleep between loop iterations (`delay(1)` keeps CPU active) | ~10 mA | **Fixed** — `esp_pm_configure()` auto light-sleep + `delay(40)` when idle |
| Built-in user LED + PWM timer always running | ~1–2 mA | Partial — LED fully off when idle & disconnected |
| Charge IC quiescent current | ~1 mA | **Board-level, not fixable in firmware** |
| On-board 3.3 V LDO leakage | ~0.5 mA | **Board-level, not fixable in firmware** |
| Sensors (BMI270+BMA400+MMC5603) at 25 Hz | ~2 mA | N/A on bare board; `sensorsOff()` added for when daughter board attaches |
| W25Q128 standby vs deep-powerdown | ~30 µA | N/A on bare board; `flashPowerDown()` after init for when attached |

### Fixes applied (April 2026)

Commits to `Rishab_PCB_esp32c6.ino`:

1. **`setCpuFrequencyMhz(80)` at boot** — saves ~10–15 mA vs 160 MHz. Bumped back to 160 MHz only inside `doWifiUpload()` (Wi-Fi radio needs it), then dropped back.
2. **`esp_pm_configure({max 80 MHz, min 40 MHz, light_sleep_enable=true})`** — FreeRTOS idle task parks the CPU automatically when no task is ready.
3. **Main loop idle behavior** — `delay(40)` when not logging/replaying/sending instead of `delay(1)`. Allows longer light-sleep windows. Tight `delay(1)` still used during active sampling for 20 Hz timing.
4. **BLE advertising 1000–1500 ms interval** — `adv->setMinInterval(1600); setMaxInterval(2400)` (units are 0.625 ms).
5. **`sensorsOff()` after boot-time hardware probe** — VDD rail cut via Q2 MOSFET; only re-powered during `startLogging()` and cut again in `stopAndReplay()`. No effect on bare board but correct behavior once daughter board attaches.
6. **`flashPowerDown()` after `flashInit()`** — W25Q128 goes from ~30 µA standby to ~1 µA. No effect on bare board.

### Expected after fixes

- Idle draw: **~8–12 mA** (down from ~50 mA)
- Recording: ~40 mA (unchanged — IMU, BLE, flash all active)
- Wi-Fi upload: ~63 mA avg, 650 mA peaks (unchanged — bumped back to 160 MHz)
- Deep sleep: target ~50 µA (not yet validated on this hardware)

### Things we CANNOT fix in firmware (board-level)

- **Charge LED is hardwired** — drains a few mA continuously whenever USB power is present. Next PCB rev should route the charge LED through a GPIO like the ESP32-S3 board does.
- **No undervoltage protection** — software-enforced cutoff at 3.3 V is required ([[01 - Critical Preservation Rules]]). Still TODO.
- **On-board LDO / charge IC leakage** — floor of ~1–2 mA that can't be removed without cutting traces.

### TODO — still to optimize

- [ ] Implement 3.3 V undervoltage cutoff (firmware → deep sleep)
- [ ] LED fully off when disconnected + not charging (currently still blinks to show advertising)
- [ ] Measure actual deep-sleep current with PPK2 on this XIAO unit
- [ ] Validate idle draw after the fixes — target is 10 mA, need real measurement
- [ ] Evaluate turning off BLE entirely after extended idle (e.g. 5 min with no connection) and only re-enabling on button press

### Fundamentals to carry into the next PCB spin

- Route the charge LED through a GPIO (or add a DNP option)
- Add an undervoltage-cutoff load switch on the battery (TPS22916 / similar)
- Consider a dedicated TVS + soft-start for USB input
- Expose a BAT ADC voltage divider so battery reads work out of the box

### Sources

- [Seeed PPK2 deep sleep thread (15 µA baseline)](https://forum.seeedstudio.com/t/15ua-with-xiao-esp32c6-with-ppk2-while-sleeping-basic-example/276412)
- [Tomas McGuinness — Matter power optimization (63 mA / 650 mA peaks)](https://tomasmcguinness.com/2025/01/06/lowering-power-consumption-in-esp32-c6/)
- [Sleep current comparison C6/S3/C3](https://forum.seeedstudio.com/t/comparison-of-sleep-currents-for-xiao-esp32c6-s3-and-c3/276444)
- [Seeed forum — no undervoltage protection](https://forum.seeedstudio.com/t/does-xiao-esp32-c6-have-low-voltage-protection-for-li-ion-battery/292538)
- [Seeed forum — boards frying during charge](https://forum.seeedstudio.com/t/xiao-esp32c6-keeps-frying-when-charging/283403)
- [C3 quiescent current issue (similar board)](https://forum.seeedstudio.com/t/xiao-esp32c3-battery-ussage/266185)

---

## TODOs for hardware bring-up

1. **BMI270 config blob upload** — The BMI270 requires an 8 KB configuration binary to be written to the INIT_DATA register at boot. Without it, gyro data may read as zero. The blob is available from Bosch's BMI270 API (`bmi270_config_file[]`). The current driver initializes the chip and enables gyro but skips the blob upload.

2. **Battery ADC** — The XIAO ESP32-C6 has a built-in battery voltage divider when a LiPo is connected to the BAT pads. Need to identify the correct internal ADC channel and calibration.

3. **MMC5603 read function** — Init is done, but the magnetometer data read function isn't implemented. Not needed for the core Cresento pipeline (gyroY only) but useful for future heading/orientation features.

4. **SPI flash session storage** — The W25Q128JV driver has full read/write/erase functions but the firmware currently uses LittleFS on internal flash for session storage (same as the stub version). TODO: optionally use the external 16 MB flash for session storage, which would allow much larger sessions (16 MB vs ~2 MB internal).

5. **IMU interrupt-driven sampling** — Both BMI270 and BMA400 INT1 pins are wired to GPIO0. Could use hardware interrupts for precise 20 Hz timing instead of polling in `loop()`.

6. **BMA400 as low-power wake sensor** — The BMA400 is a dedicated low-power accel that can run at ~5 uA. Could use it for motion-detect wakeup while BMI270 is powered down.

7. **Enable gzip compression** — The miniz library files are stashed in `Rishab_PCB_esp32c6_libs/miniz/` (outside the sketch folder to avoid bloating the build by ~500 KB). To enable: copy all `miniz*.{c,h}` files into the sketch folder and set `#define USE_GZIP 1` in the .ino. Also requires **Huge APP (3MB No OTA)** partition scheme since the ESP32-C6 only has 4 MB flash. The ESP32-S3 version uses gzip (codec 2) and the app decoders handle both codecs, so this should be enabled once RAM usage is profiled under real sensor load.

---

## Key differences from ESP32-S3 SmallBoard

| Aspect            | ESP32-S3 SmallBoard              | XIAO ESP32-C6 + Rishab PCB          |
| ----------------- | -------------------------------- | ------------------------------------ |
| CPU               | Xtensa LX7 (dual-core, 240 MHz) | RISC-V (160 MHz HP + 20 MHz LP)     |
| RAM               | 8 MB PSRAM                       | 512 KB SRAM (no PSRAM)              |
| Wi-Fi             | Wi-Fi 4 (802.11n)                | Wi-Fi 6 (802.11ax)                  |
| Extra radios      | None                             | 802.15.4 (Thread/Zigbee/Matter)     |
| IMU               | ICM-42688 / QMI8658 / LSM6DS    | BMI270 (6-axis) + BMA400 (accel)    |
| Magnetometer      | None                             | MMC5603NJ                            |
| Flash storage     | W25Q128 16 MB SPI                | W25Q128JV 16 MB SPI (same chip!)    |
| Fuel gauge        | MAX17048                         | None — use XIAO built-in BAT ADC    |
| Charger IC        | BQ24075 (GPIO-controlled)        | XIAO built-in LiPo charge circuit   |
| Power gating      | SYSOFF pin                       | P-MOSFET + NPN (D2 GPIO2)           |
| LED               | NeoPixel (RGB)                   | Single-color built-in (GPIO15)      |
| Form factor       | Full custom PCB                  | COTS XIAO + custom daughter board    |

---

## Future possibilities (ESP32-C6 unique)

- **Thread / Zigbee / Matter:** Mesh networking between multiple ShinPads on a pitch
- **Wi-Fi 6:** Lower latency uploads, potentially real-time streaming during sessions
- **LP core (20 MHz RISC-V):** Could handle IMU sampling while main core sleeps
- **BMI270 + BMA400 fusion:** BMI270 for high-accuracy 6-axis, BMA400 as always-on wake sensor
- **MMC5603NJ heading:** Add compass heading for player orientation on the pitch

---

## Related

- [[Proposal - ESP-NOW Mesh Gateway]] — pad-to-pad ESP-NOW mesh proposal (exploring)
- [[BLE Protocol]] — frozen GATT contract
- [[Session State Machine]] — idle -> logging -> flushing
- [[ESP32-S3 SmallBoard Firmware]] — sibling firmware (ICM-42688 based)
- [[Nordic BLE Firmware (nRF52811)]] — original firmware
- [[Hardware Overview]] — cross-board comparison
- [[03 - Data Pipeline]] — how sensor data flows to Firebase
