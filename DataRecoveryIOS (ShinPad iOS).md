---
title: DataRecoveryIOS (ShinPad iOS)
type: project
tags:
  - project/ios
  - frontend
  - mobile
created: 2026-04-11
status: active
location: "DataRecoveryIOS/"
---

# 🍎 DataRecoveryIOS / ShinPad iOS App

The native iOS sibling of the [[Cresento React Native App|RN app]]. Implements the same flows in pure Swift / SwiftUI on top of the same Firebase backend and the same BLE GATT contract.

> [!info] Feature alignment
> RN and iOS must stay roughly aligned — same flows, same role-based UI, same metric values. If you add a feature on one side, plan it for the other.

---

## Stack

- **Swift** + **SwiftUI**
- **Firebase** (auth, Firestore, storage, crashlytics — same project as RN/Web)
- **CoreBluetooth** for BLE
- **Fonts:** Inter, PlusJakartaSans
- Dark mode + haptic feedback

---

## Layout (domain-organised)

```
DataRecoveryIOS/
├── Authentication/
├── Coach/
├── Home/
├── Friends/
├── Settings/
├── Global Stuff/         (shared services, ViewModels)
└── Resources/            (fonts, assets)
```

The app is structured by **feature domain**, not by MVC layer. Each domain owns its views, view models, and services.

---

## Key classes

| Class                  | Role                                            |
| ---------------------- | ----------------------------------------------- |
| `BLEViewModel` (v2.5.1)| BLE state machine, peer of `BleContext.tsx`     |
| `AuthViewModel`        | Auth + role + sign-in/out                       |
| `StatsEngine`          | Analytics — Swift port of [[StatsEngine Cross-Platform]] |
| `ActiveSessionStore`   | In-flight session bookkeeping                   |

---

## BLE responsibilities

`BLEViewModel` does the same job as `Cresento/src/src/contexts/BleContext.tsx` on the RN side — scanning, connecting, dispatching commands, decoding notifications, surfacing battery state. It MUST speak the same protocol byte-for-byte. See [[BLE Protocol]].

> [!warning] Lockstep with RN
> Adding a command? Add it to:
> 1. The chip firmware ([[Nordic BLE Firmware (nRF52811)]] AND [[ESP32-S3 SmallBoard Firmware]])
> 2. `BleContext.tsx` on RN
> 3. `BLEViewModel` on iOS
>
> Don't ship one without the others.

---

## Stats engine

The Swift `StatsEngine` is one of three implementations. The expectation is that **the same session produces the same numbers** in RN, iOS, and the Web's `lib/analytics/`. See [[StatsEngine Cross-Platform]] for the contract.

---

## Roles

Same as RN: `"player"` and `"coach"`. Role is read from the Firestore `users/{uid}` document and gates which views are shown.

---

## 🛑 Hands off

- ❌ BLE protocol implementation in `BLEViewModel`
- ❌ StatsEngine metric formulas (must match other platforms)
- ❌ Session decoding (must match firmware encoding)

OK to change:
- ✅ UI / UX in any of the domain folders
- ✅ Local state / view models
- ✅ Adding new screens

---

## Related

- [[Cresento React Native App]] — feature-aligned sibling
- [[Cresento Website]] — same backend
- [[BLE Protocol]]
- [[StatsEngine Cross-Platform]]
- [[Firebase Backend]]
