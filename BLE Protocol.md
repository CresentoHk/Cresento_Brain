---
title: BLE Protocol
type: shared-system
tags:
  - shared
  - protocol
  - critical
  - firmware
created: 2026-04-11
---

# 📡 BLE Protocol (FROZEN)

The wire contract that the chip firmware and the phone apps speak to each other. **Implemented in four places**, must stay byte-for-byte identical:

| Implementation                  | File                                                    |
| ------------------------------- | ------------------------------------------------------- |
| Nordic firmware                 | `Nordic_BLE_Test/Nordic_BLE_Test.ino`                  |
| ESP32 firmware                  | `Cresento/chip_code/New_SmallBoard_Code/...ino`         |
| RN app                          | `Cresento/src/src/contexts/BleContext.tsx`              |
| iOS app                         | `DataRecoveryIOS/.../BLEViewModel`                      |

> [!danger] FROZEN
> Changing one side of this without the others bricks the field. **All four implementations must change in lockstep, in the same release.**

---

## Service & characteristics

**Service UUID:** `12345678-1234-1234-1234-1234567890AB`

**8 GATT Characteristics:**

| #   | Name           | Direction      | Purpose                                  |
| --- | -------------- | -------------- | ---------------------------------------- |
| 1   | `Control`      | Write          | Commands from app to chip                |
| 2   | `Data`         | Notify         | IMU data stream from chip to app         |
| 3   | `BattState`    | Read / Notify  | Battery state (charging / idle / low)    |
| 4   | `BattPct`      | Read / Notify  | Battery percentage                       |
| 5   | `BattVolt`     | Read / Notify  | Battery voltage                          |
| 6   | `ChargingEvt`  | Notify         | Charge plug-in / plug-out events         |
| 7   | `ChargingTime` | Read           | Time spent charging                      |
| 8   | `DeviceID`     | Read           | Per-device unique identifier             |

---

## Command bytes (on `Control`)

> [!info] Specifics live in code
> The exact byte values are in `Nordic_BLE_Test.ino` near the GATT setup, mirrored in `BleContext.tsx` and `BLEViewModel.swift`. Keep this document at the conceptual level — duplicating values here invites them to drift.

Conceptual command set:
- Start logging
- Stop logging
- Flush flash to BLE
- Get battery state
- (and a small number of utility commands)

---

## Data notification format

Outbound IMU samples are sent on the `Data` characteristic as **delta-pack compressed** chunks, optionally **gzipped** on top (ESP only, configurable). Each notification includes a sequence number for reassembly.

- Sample rate: **20 Hz** (`SAMPLE_HZ` in firmware, decoder constant in apps)
- Packet payload sizes are over-allocated as static buffers in firmware to bound RAM usage
- Apps reassemble in order using the sequence number, then convert to CSV in memory

See [[Data Pipeline]] for the full encode → decode → upload flow.

---

## Zombie connection detection

> [!warning] 5 second timeout — intentional
> If a phone disconnects ungracefully (Bluetooth toggled off, app force-killed mid-session), the chip would otherwise sit in a half-connected state and miss the next legitimate reconnect. The 5 s timeout exists to clear this.
>
> Don't tune it. It has caused user-visible regressions in the past.

---

## Adding a new command — checklist

If you absolutely must extend the protocol:

- [ ] Pick the next available command byte
- [ ] Implement in `Nordic_BLE_Test.ino`
- [ ] Implement in `New_SmallBoard_Code.ino`
- [ ] Implement in `BleContext.tsx`
- [ ] Implement in iOS `BLEViewModel`
- [ ] Test all four together on hardware
- [ ] Ship them in the same release (don't OTA the chip ahead of the app or vice versa — devices in the field will see version skew)

---

## Why this is so locked down

There are physical chips on physical legs of physical players. You can't roll back firmware on a Sunday morning before a match. Every change to this contract has a non-trivial recovery cost if it breaks something. The conservatism is deliberate.

---

## Related

- [[Nordic BLE Firmware (nRF52811)]]
- [[ESP32-S3 SmallBoard Firmware]]
- [[Cresento React Native App]]
- [[DataRecoveryIOS (ShinPad iOS)]]
- [[Session State Machine]]
- [[01 - Critical Preservation Rules#🔵 BLE protocol — frozen wire format]]
