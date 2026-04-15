---
title: Hardware Overview
type: shared-system
tags:
  - hardware
  - shared
created: 2026-04-11
---

# 🔧 Hardware Overview

Cresento has **three ShinPad firmware tracks in active development**, all speaking the same BLE protocol so the apps can ignore which chip is inside.

| Property      | Nordic board                       | ESP32-S3 board                         | XIAO ESP32-C6 board                    |
| ------------- | ---------------------------------- | -------------------------------------- | --------------------------------------- |
| MCU           | Nordic nRF52811                    | Espressif ESP32-S3                     | Espressif ESP32-C6 (RISC-V)            |
| BLE stack     | S112 SoftDevice v7.3.0 (peripheral)| Arduino BLEDevice                      | Arduino BLEDevice                       |
| Wi-Fi         | ---                                | Wi-Fi 4 (TCP streaming to RPi)         | Wi-Fi 6 (802.11ax)                     |
| Extra radios  | ---                                | ---                                    | 802.15.4 (Thread/Zigbee/Matter)         |
| Toolchain     | Arduino IDE + Nordic SDK           | Arduino IDE + ESP32 core               | Arduino IDE + ESP32 core                |
| Programmer    | J-Link / SWD                       | USB                                    | USB-C                                   |
| Form factor   | Custom PCB                         | Custom PCB                             | COTS module (21x17.8 mm)               |
| Firmware note | [[Nordic BLE Firmware (nRF52811)]] | [[ESP32-S3 SmallBoard Firmware]]       | [[ESP32-C6 XIAO Firmware (Rishab PCB)]] |

The Nordic and ESP32-S3 boards share the same peripherals (IMU, flash, fuel gauge). The XIAO ESP32-C6 is a bare module — external breakouts must be wired for sensor data.

---

## Shared peripherals

### ICM-42688 — IMU
- **6-axis** (3-axis gyro + 3-axis accel)
- Sampled at **20 Hz** in firmware
- **I²C** bus on Nordic at pins **P0.11 / P0.12**
- Same chip on both boards

### W25Q128 — SPI flash
- **16 MB** NOR flash
- **SPI** bus
- Used for offline session storage so the chip can record without a phone connected
- ESP variant accesses it through **LittleFS**

### MAX17048 — fuel gauge
- I²C battery state-of-charge gauge
- Reports voltage, percentage, charge state
- Both boards use the same custom driver pair: `FuelGauge_MAX17048.cpp/h`

---

## Nordic-specific pins

| Function   | Pin    |
| ---------- | ------ |
| LED red    | P0.13  |
| LED green  | P0.08  |
| LED blue   | P0.09  |
| Button     | P0.06  |
| I²C SDA    | P0.11  |
| I²C SCL    | P0.12  |

> [!warning] Pin assignments are baked into firmware
> Don't reassign pins without recompiling AND making sure the PCB matches.

---

## ESP-specific pins (high level)

- **NeoPixel** RGB LED (Adafruit driver) instead of three discrete LEDs
- **VBUS divider** for USB plug detect: top 100 kΩ, bottom 82 kΩ
- USB acts as charging input, serial port, and **deep-sleep wake source** (EXT1 wake)

---

## PCB designs

| Path                                                            | Board        |
| --------------------------------------------------------------- | ------------ |
| `Cresento/chip_code/NEW esp_chip design/PCB/CresentoProject4.PrjPCB` | ESP variant (Altium) |
| `Cresento/chip_code/Reference code_other_application/SENSAI/PCB/`   | Reference design (separate) |

The Altium project is the source of truth for the ESP board layout. Manufacturing files (Gerber, BOM in `bom_JLC.xls`) are siblings of the project file.

---

## Power profile (ESP)

- Deep sleep with `RTC_DATA_ATTR` variables persisting key state across wake
- Auto-sleep arms 1.5 s after USB unplug to give BLE notifications time to flush
- Unplug "freeze" hold of 60 s to prevent rapid plug/unplug cycling
- See [[ESP32-S3 SmallBoard Firmware#Power management]]

---

## Related

- [[Nordic BLE Firmware (nRF52811)]]
- [[ESP32-S3 SmallBoard Firmware]]
- [[ESP32-C6 XIAO Firmware (Rishab PCB)]]
- [[BLE Protocol]]
- [[Session State Machine]]
- [[Data Pipeline]]
