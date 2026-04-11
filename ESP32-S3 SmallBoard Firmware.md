---
title: ESP32-S3 SmallBoard Firmware
type: project
tags:
  - project/esp
  - firmware
  - hardware
  - critical
created: 2026-04-11
status: active
location: "Cresento/chip_code/New_SmallBoard_Code/"
---

# 🟢 ESP32-S3 SmallBoard Firmware

The newer ShinPad firmware track. Runs on a custom **ESP32-S3** board. Same BLE GATT contract as the [[Nordic BLE Firmware (nRF52811)|Nordic firmware]] (so phone apps don't care which chip is inside) **plus** Wi-Fi/TCP live streaming to a Raspberry Pi sink.

> [!info] Two parallel firmware tracks
> Cresento has TWO chip designs in active development. This one is the ESP-based newer board. The Nordic one is the original. Both speak the same BLE protocol.

---

## Stack

- **Chip:** ESP32-S3 (Arduino core)
- **BLE:** Arduino BLE (BLEDevice / BLEServer / BLE2902)
- **Wi-Fi:** ESP32 Wi-Fi station mode
- **Filesystem:** LittleFS
- **Power management:** `esp_pm`, `esp_sleep`, RTC deep sleep
- **Compression:** `miniz` (gzip) bundled into the sketch folder
- **LEDs:** Adafruit NeoPixel
- **Fuel gauge:** MAX17048 (custom driver: `FuelGauge_MAX17048.cpp/h`)

---

## Hardware

| Component        | Purpose                                  |
| ---------------- | ---------------------------------------- |
| ICM-42688 IMU    | 6-axis motion @ 20 Hz                    |
| W25Q128 SPI flash| Session storage (used via LittleFS)      |
| MAX17048         | Battery fuel gauge                       |
| NeoPixel         | RGB LED status (Adafruit driver)         |
| USB              | Charging + serial + wake source          |

---

## Layout

```
Cresento/chip_code/New_SmallBoard_Code/
├── New_SmallBoard_Code.ino    (main sketch)
├── FuelGauge.h
├── FuelGauge_MAX17048.cpp / .h
├── miniz.{c,h}                (gzip — drop-in)
├── miniz_tdef.{c,h}, miniz_tinfl.{c,h}, miniz_zip.{c,h}
├── BatteryLimiter.zip
├── build/
├── Reference Code (Unedited onboarding 2)/
└── Struans attempt/           (alt branch — be careful which you edit)
```

> [!warning] Multiple variants exist
> The directory contains a working sketch plus alternate / reference variants. Always confirm which file is the active one before editing.

---

## Wi-Fi / TCP live streaming

Unique to the ESP board: when configured, it connects to a Wi-Fi access point and streams session data over TCP in real time, in addition to logging to flash for later BLE flush.

```c
const char* WIFI_SSID     = "cresento-1";
const char* WIFI_PASSWORD = "12345678";
const char* TCP_HOST      = "192.168.0.111";
const uint16_t TCP_PORT   = 8080;
```

The Raspberry Pi at the configured host acts as the sink. Reconnect interval is 2 s.

> [!warning] Hardcoded credentials
> SSID + password are baked into the sketch. Fine for the lab, **do not ship to users like this**.

---

## Compression pipeline

Same conceptual pipeline as Nordic, with explicit configuration:

```c
#define USE_GZIP          1     // 0 = no gzip, 1 = gzip after bit-pack
#define CHUNK_ROWS        512   // 512–1024 sweet spot for ESP32-S3
#define COLS              1     // (gyro Y only — was 6, currently reduced)
#define SAMPLE_HZ         20
#define CODEC_DELTAPACK   1
#define CODEC_DP_GZIP     2
```

Two codecs: plain delta-pack, or delta-pack + gzip. Worst-case payload sizes are intentionally over-allocated as static buffers (`bitpackBuf`, `gzipBuf`).

---

## Power management

- `RTC_DATA_ATTR` variables persist across deep sleep (e.g. `usbWakeSession`, `restedMv`)
- Auto-sleep arms 1500 ms after USB unplug to give BLE time to flush
- USB unplug "freeze" hold of 60 s
- VBUS divider (top 100 kΩ, bottom 82 kΩ) for plug-detect

This is the kind of thing that "looks like a battery bug" if you change it carelessly.

---

## Safe serial logging

`SAFE_PRINT*` macros gate `Serial` writes on `usbHostConnected`. Without this, `Serial.print` hangs the entire sketch when USB is unplugged.

```c
#define SAFE_PRINT(...)   if (usbHostConnected) { Serial.print(__VA_ARGS__); }
#define SAFE_PRINTLN(...) if (usbHostConnected) { Serial.println(__VA_ARGS__); }
#define SAFE_PRINTF(...)  if (usbHostConnected) { Serial.printf(__VA_ARGS__); }
```

**Always use `SAFE_PRINT*` instead of bare `Serial.print*`** in this sketch.

---

## LED behaviour

NeoPixel-driven. Same caveats as the Nordic LED rules — sequences communicate state, **don't change timing without testing on hardware**.

See [[01 - Critical Preservation Rules#💡 LED sequences — timing-sensitive]].

---

## Related sub-folders

| Path                                                          | What it is                                            |
| ------------------------------------------------------------- | ----------------------------------------------------- |
| `Cresento/chip_code/NEW esp_chip design/PCB/`                 | Altium PCB project (`CresentoProject4.PrjPCB`)       |
| `Cresento/chip_code/Onboarding (2) Unedited Raw github/`      | Older onboarding code variants (reference)           |
| `Cresento/chip_code/Reference code_other_application/SENSAI/` | A separate reference app (chip + PCB + phone app)    |
| `Cresento/chip_code/led_tester/led_tester.ino`                | Standalone LED bring-up sketch                       |

---

## 🛑 Hands off

- ❌ BLE UUIDs / commands / notification format (must match Nordic + apps)
- ❌ LED timing
- ❌ Power management timing constants (`AUTO_SLEEP_GRACE_MS`, `UNPLUG_HOLD_MS`)
- ❌ `RTC_DATA_ATTR` semantics (these survive deep sleep — assumptions are everywhere)
- ❌ Compression format (must match what the app decoder expects)

OK to change with care:
- ✅ Add new BLE commands (lockstep with app + Nordic)
- ✅ Tune Wi-Fi reconnect timing
- ✅ Bug fixes outside the LED / power / BLE areas

---

## Related

- [[Nordic BLE Firmware (nRF52811)]] — sibling firmware, same protocol
- [[BLE Protocol]] — the contract both chips speak
- [[Session State Machine]]
- [[Hardware Overview]]
- [[01 - Critical Preservation Rules]]
