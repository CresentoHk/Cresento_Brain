---
title: Critical Preservation Rules
type: rules
tags:
  - critical
  - shared
  - protocol
created: 2026-04-11
aliases:
  - Do Not Break
  - Preservation Rules
---

# 🛑 Critical Preservation Rules

> [!danger] Rule Zero
> **Do not break existing functionality.** Read the existing code BEFORE editing. When in doubt, **add**, do not modify. Preserve all callers.

This is the master "do not touch" list. Every item here exists because something downstream will silently break if you change it. Each rule links to the project notes that depend on it.

---

## 🔵 BLE protocol — frozen wire format

> [!danger] Never change these
> The same protocol is implemented FOUR times (Nordic firmware, ESP firmware, RN BleContext, iOS BLEViewModel). Changing one side without the others = bricked devices in the field.

- **Service UUID:** `12345678-1234-1234-1234-1234567890AB`
- **8 Characteristics** (Control, Data, BattState, BattPct, BattVolt, ChargingEvt, ChargingTime, DeviceID)
- **Command bytes** (start log, stop log, flush, get battery, etc.)
- **Notification packet format** (delta-pack compressed IMU, optional gzip, sequence numbers)
- **Zombie connection detection** — 5-second timeout is intentional, do not "optimise" it

See: [[BLE Protocol]], [[Nordic BLE Firmware (nRF52811)]], [[ESP32-S3 SmallBoard Firmware]]

---

## 💡 LED sequences — timing-sensitive

> [!warning] Test on hardware before merging
> LED behaviour communicates state to the user with no other UI. Off-by-one timings here look like "the chip is broken" to a coach in the field.

- LED pin assignments in Nordic firmware: P0.13 (red), P0.08 (green), P0.09 (blue)
- ESP board uses Adafruit NeoPixel — different driver, similar semantics
- **Always explain WHY a proposed LED change is safe** before making it
- The user has explicitly called this out as the most fragile part of the chip code

See: [[Nordic BLE Firmware (nRF52811)#LED behaviour]], [[ESP32-S3 SmallBoard Firmware#LED behaviour]]

---

## 🔁 Session state machine

> [!danger] Carefully tuned — do not restructure
> States: **idle → logging → flushing → idle**

- The flushing state exists to drain the W25Q128 flash before the chip is allowed back to idle
- Mid-flush disconnects must resume on reconnect, not restart
- Charging events transition state too — see [[Session State Machine]]

Touching any side of this (firmware OR app) will cause silent data loss.

---

## 📦 Firestore session paging

> [!warning] 700 KB page cap is load-bearing
> Firestore documents have a hard 1 MB limit. The 700 KB cap leaves headroom for metadata + atomic batch overhead. **Never bypass `SessionUploader`** to write `sensorData` directly.

- Paging logic lives in `Cresento/src/src/utils/SessionUploader.ts`
- All three apps (RN, iOS, Web) read paged sessions — changing the page schema breaks ALL clients
- Use existing wrappers: `lib/firestore.ts` on web, `SessionUploader.ts` + utils on RN, ViewModels on iOS

See: [[Firebase Backend]], [[Cresento React Native App#SessionUploader]]

---

## 🗓️ Calendar view session loader

> [!warning] Critical user-facing path
> The website's `components/dashboard/sessions-calendar.tsx` (~70 KB) loads sessions for the calendar. Changes to how sessions are queried, paged, or grouped here directly impact what coaches see.

- Pairs with `lib/firestore.ts` and `lib/analytics/`
- Trim functionality lives in `lib/trim-utils.ts` (~31 KB) — also must be preserved

See: [[Cresento Website#Sessions calendar]]

---

## 🧮 StatsEngine consistency across platforms

> [!info] Three implementations, one truth
> - RN: `Cresento/src/src/utils/StatsEngine.ts` (~37 KB)
> - Web: `Cresento Website/Cresento.net/lib/analytics/`
> - iOS: `DataRecoveryIOS/.../StatsEngine.swift`
>
> All three must produce **the same numbers** for the same input session. If you change a metric formula, change it in all three or you'll get reports of "the website says X but the app says Y".

See: [[StatsEngine Cross-Platform]]

---

## 🗃️ Repository quirks — leave them alone

- **Double-nested `src/src/`** in the RN app is intentional (history). Do **not** restructure.
- **Existing wrappers exist for reasons** — extend them, don't bypass them.
- **Don't refactor surrounding code** when fixing something. Bug fixes stay scoped to the bug.

---

## ✅ Safe-change checklist

Before merging anything in firmware, BLE-adjacent code, Firestore wrappers, or calendar view:

- [ ] Read the existing code
- [ ] Identify all callers / cross-platform mirrors
- [ ] Explain why the change is safe in the PR description
- [ ] Test on real hardware where the chip is involved
- [ ] Verify all three StatsEngine implementations still agree if you touched a metric
