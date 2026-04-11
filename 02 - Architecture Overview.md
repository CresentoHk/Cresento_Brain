---
title: Architecture Overview
type: overview
tags:
  - architecture
  - shared
created: 2026-04-11
---

# 🏗️ Architecture Overview

Cresento is one product split across many runtimes. The unifying spine is a single Firebase project that all clients (mobile, web, firmware-via-app) read from and write to.

```
                    ┌────────────────────┐
                    │  ShinPad Hardware  │
                    │  ┌──────────────┐  │
                    │  │ nRF52811 OR  │  │ ← Two firmware tracks
                    │  │ ESP32-S3     │  │   (different chips)
                    │  │ + ICM-42688  │  │
                    │  │ + W25Q128    │  │
                    │  │ + MAX17048   │  │
                    │  └──────────────┘  │
                    └─────────┬──────────┘
                              │ BLE GATT (frozen UUIDs)
                              │ + ESP only: TCP/Wi-Fi to RPi
                ┌─────────────┼─────────────┐
                │             │             │
       ┌────────▼─────┐ ┌─────▼─────┐ ┌────▼─────────┐
       │ React Native │ │ iOS Swift │ │ (Raspberry Pi│
       │   (Cresento) │ │ (ShinPad) │ │  TCP sink)   │
       └────────┬─────┘ └─────┬─────┘ └──────────────┘
                │             │
                │  paged uploads (700 KB cap)
                │             │
                └──────┬──────┘
                       │
              ┌────────▼─────────┐
              │   Firebase       │
              │  (cresento-8b603)│
              │                  │
              │  • Auth          │
              │  • Firestore     │ ← shared collections
              │  • Storage       │
              │  • Functions     │
              │  • Crashlytics   │
              └────────┬─────────┘
                       │
              ┌────────▼──────────┐
              │  Next.js Website  │
              │  (coach dashboard)│
              └───────────────────┘

         (separate, video-only path)
              ┌───────────────────┐
              │  Veo Vision Pipe  │
              │  YOLO + homography│
              │   → heatmaps      │
              └───────────────────┘
```

---

## 🧩 The five runtimes

| Runtime          | Language       | Talks to                 | Note                              |
| ---------------- | -------------- | ------------------------ | --------------------------------- |
| Nordic firmware  | C/Arduino      | RN/iOS via BLE           | [[Nordic BLE Firmware (nRF52811)]]|
| ESP32 firmware   | C/Arduino      | BLE + WiFi/TCP           | [[ESP32-S3 SmallBoard Firmware]]  |
| RN app           | TS/React Native| Firebase + BLE           | [[Cresento React Native App]]     |
| iOS app          | Swift/SwiftUI  | Firebase + BLE           | [[DataRecoveryIOS (ShinPad iOS)]] |
| Website          | TS/Next.js     | Firebase                 | [[Cresento Website]]              |
| Vision pipeline  | Python         | Local files (videos)     | [[Vision System (Veo Pipeline)]]  |

The Vision pipeline is **separate** from the wearable pipeline. It processes match video on a workstation; results are not yet wired into Firebase.

---

## 🔄 Data flow: a recording session

1. **Coach starts recording** in the [[Cresento React Native App|RN app]] (or [[DataRecoveryIOS (ShinPad iOS)|iOS app]])
2. App sends **start command** over BLE to the [[Nordic BLE Firmware (nRF52811)|Nordic]] or [[ESP32-S3 SmallBoard Firmware|ESP]] chip
3. Chip transitions `idle → logging` ([[Session State Machine]])
4. ICM-42688 sampled at **20 Hz**, stored to W25Q128 flash
5. On stop / disconnect / charging event, chip transitions `logging → flushing`
6. Chip streams flash contents back over BLE notifications, **delta-pack compressed + optional gzip**
7. App decodes binary log → CSV in memory
8. App pages the CSV up to Firestore via [[Cresento React Native App#SessionUploader|SessionUploader]] — atomic batches, **700 KB / page**
9. [[StatsEngine Cross-Platform|StatsEngine]] computes 40+ metrics on the device
10. The [[Cresento Website|website]] reads the same paged session for the coach dashboard

See [[03 - Data Pipeline]] for the deep-dive on encoding, paging, and stats.

---

## 🧱 Why two firmware codebases?

There are **two parallel chip designs**, currently both in active development:

- **Nordic nRF52811** — original board. BLE only. Uses S112 SoftDevice. See `Nordic_BLE_Test/Nordic_BLE_Test.ino`.
- **ESP32-S3 (SmallBoard)** — newer board. BLE **plus** Wi-Fi/TCP for live streaming to a Raspberry Pi sink. See `Cresento/chip_code/New_SmallBoard_Code/`.

Both chips:
- Use the same **ICM-42688** IMU at 20 Hz
- Use the same **W25Q128** SPI flash for offline session storage
- Use the same **MAX17048** fuel gauge
- Speak the **same BLE GATT contract** (so the apps don't care which chip is inside) — see [[BLE Protocol]]

The ESP board adds Wi-Fi streaming on top of BLE, intended for live mode. The Nordic board is BLE-only.

---

## 🎨 Design system

| Platform | Theme            | Accent      | Notes                                          |
| -------- | ---------------- | ----------- | ---------------------------------------------- |
| RN app   | Dark             | `#00A483`   | Backgrounds `#141414` / `#1E1E1E` / `#323232`  |
| Website  | Dark + Tailwind v4 | shadcn/ui  | Radix UI primitives, Framer Motion             |
| iOS      | Dark             | matches RN  | Inter + PlusJakartaSans, haptic feedback       |

The teal accent `#00A483` is the brand colour and shows up everywhere.
