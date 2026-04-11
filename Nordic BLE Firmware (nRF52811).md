---
title: Nordic BLE Firmware (nRF52811)
type: project
tags:
  - project/nordic
  - firmware
  - hardware
  - critical
created: 2026-04-11
status: active
location: "Nordic_BLE_Test/"
---

# 🔵 Nordic BLE Firmware (nRF52811)

The original ShinPad firmware. Runs on a custom **nRF52811** board with Nordic's **S112 SoftDevice v7.3.0**. Implements the BLE GATT contract that the [[Cresento React Native App|RN app]] and [[DataRecoveryIOS (ShinPad iOS)|iOS app]] rely on.

> [!danger] This whole project is mostly hands-off
> Read [[01 - Critical Preservation Rules]] first. Then read the existing code. Then ask before changing anything BLE-related.

---

## Stack

- **Chip:** Nordic Semiconductor nRF52811
- **SoftDevice:** S112 v7.3.0 (peripheral-only Bluetooth stack)
- **Toolchain:** Arduino IDE (C/Arduino-style with Nordic SDK)
- **Programmer:** J-Link via SWD

---

## Hardware

| Component        | Purpose                                  | Pin / Address      |
| ---------------- | ---------------------------------------- | ------------------ |
| ICM-42688 IMU    | 6-axis motion @ 20 Hz                    | I²C (P0.11/P0.12)  |
| W25Q128 SPI flash| 16 MB session log storage                | SPI                |
| MAX17048         | Battery fuel gauge                       | I²C                |
| LED red          | Status                                   | P0.13              |
| LED green        | Status                                   | P0.08              |
| LED blue         | Status                                   | P0.09              |
| Button           | User input                               | P0.06              |

See [[Hardware Overview]] for the full picture.

---

## Layout

```
Nordic_BLE_Test/
├── Nordic_BLE_Test.ino     (~1999 lines — complete GATT server + main loop)
├── BLE_SETUP.md            (build environment setup)
├── LIBRARY_INSTALL.md      (Arduino library install steps)
└── UPLOAD_TROUBLESHOOTING.md
```

It's deliberately one big file. Don't split it without a very good reason.

---

## BLE service

**Service UUID:** `12345678-1234-1234-1234-1234567890AB`

**8 Characteristics:**
1. `Control` — incoming commands (start, stop, flush, etc.)
2. `Data` — outgoing IMU data (notifications, delta-pack compressed)
3. `BattState` — battery state machine
4. `BattPct` — battery percentage
5. `BattVolt` — battery voltage
6. `ChargingEvt` — charge events
7. `ChargingTime` — charge duration
8. `DeviceID` — unique device identifier

> [!danger] FROZEN
> These UUIDs and the command/notification format are hardcoded into the RN app, the iOS app, AND the [[ESP32-S3 SmallBoard Firmware|ESP firmware]]. Changing one side breaks all the others. See [[BLE Protocol]].

---

## Data path

1. ICM-42688 sampled at **20 Hz**
2. Samples accumulated in RAM, then written to W25Q128 flash
3. On flush trigger, flash is read back, **delta-pack compressed**, optionally **gzipped**, and streamed over BLE notifications
4. App side reassembles, uploads to Firebase (see [[Cresento React Native App#SessionUploader|SessionUploader]])

---

## Session state machine

`idle → logging → flushing → idle`

- `idle` — advertising, low power
- `logging` — recording IMU to flash
- `flushing` — draining flash to BLE (may resume after disconnect)

See [[Session State Machine]] for the full diagram and transitions.

---

## LED behaviour

> [!warning] Extremely sensitive
> The LED is the **only** UI on the chip. Sequences communicate state to the user. Off-by-100ms changes here have caused user-visible regressions in the past.

Don't touch LED timing without:
1. Reading the existing sequences carefully
2. Explaining why the change is safe
3. Testing on real hardware

---

## Zombie connection detection

5-second timeout is **intentional**. When a phone disconnects ungracefully (Bluetooth toggled off, app killed mid-session), the chip would otherwise stay in a half-connected state and miss the next reconnect. Don't "optimise" this number.

---

## 🛑 Hands off

Almost everything in this project is hands-off. Specifically:

- ❌ BLE service / characteristic UUIDs
- ❌ Command byte values
- ❌ Notification packet format
- ❌ LED sequences and timing
- ❌ State machine transitions
- ❌ Zombie connection 5 s timeout
- ❌ I²C / SPI pin assignments

Things that are normally OK to change (with care):
- ✅ Bug fixes in non-protocol logic
- ✅ Adding new commands (must add to ALL clients in lockstep)
- ✅ Battery curve tuning

---

## Related

- [[ESP32-S3 SmallBoard Firmware]] — newer chip, same GATT contract
- [[BLE Protocol]] — the wire contract
- [[Session State Machine]] — chip-side state diagram
- [[Hardware Overview]] — board-level hardware
- [[01 - Critical Preservation Rules]]
