---
title: Cresento React Native App
type: project
tags:
  - project/rn-app
  - frontend
  - mobile
created: 2026-04-11
status: active
location: "Cresento/"
---

# 📱 Cresento React Native App

The flagship mobile app, used by both **players** and **coaches** with role-based navigation. It is the primary interface for recording sessions, syncing the chip, and viewing analytics.

> [!warning] Sensitive areas
> Most of this app is editable freely. The few exceptions are called out in [[01 - Critical Preservation Rules]] and below in [[#🛑 Hands off]].

---

## Stack

- **React Native** 0.80
- **TypeScript** 5.0.4
- **Firebase** (auth, Firestore, storage, crashlytics)
- **Hermes** + **New Architecture** enabled
- **Android**: minSDK 24, targetSDK 35, Kotlin
- Functional components with hooks throughout

---

## Layout

```
Cresento/
├── App.tsx → index.js          (entry)
└── src/src/                    (yes, double-nested — leave it)
    ├── contexts/
    │   └── BleContext.tsx      (~2000 lines, GLOBAL BLE state, very sensitive)
    ├── utils/
    │   ├── StatsEngine.ts      (~37 KB, analytics)
    │   └── SessionUploader.ts  (paged Firestore writes, 700 KB cap)
    ├── navigation/
    │   ├── BottomNavBar.tsx    (player tab nav)
    │   └── CoachTabView.tsx    (coach tab nav)
    ├── pages/
    │   └── HomePage.tsx        (main recording interface)
    ├── algorithms/
    ├── hooks/
    └── theme.ts                (#00A483 accent, dark theme)
```

> [!info] Why `src/src/`?
> The double nesting is a historical mistake from initial scaffolding. **Do not restructure** — too many imports depend on the path. The pain is worth less than the breakage cost.

---

## Build commands

```bash
cd Cresento
npm start          # Metro bundler
npm run android    # Android build
npm run ios        # iOS build
```

---

## Key files (cheat sheet)

| File                                       | Purpose                                              |
| ------------------------------------------ | ---------------------------------------------------- |
| `src/src/contexts/BleContext.tsx`          | Global BLE state, scanning, connection, commands    |
| `src/src/utils/StatsEngine.ts`             | Computes 40+ analytics metrics (one of three impls) |
| `src/src/utils/SessionUploader.ts`         | Pages binary session into Firestore (700 KB cap)    |
| `src/src/navigation/BottomNavBar.tsx`      | Bottom tabs for player role                         |
| `src/src/navigation/CoachTabView.tsx`      | Coach role navigation                               |
| `src/src/pages/HomePage.tsx`               | Main recording screen                               |
| `src/src/theme.ts`                         | Design tokens — accent `#00A483`                    |

---

## 🛑 Hands off

These are the parts that have bitten the codebase before:

### BleContext.tsx
Holds the **global** BLE state machine. Scanning, connection, command dispatch, packet decode, zombie-connection detection (5 s timeout — intentional). It is ~2000 lines because every concern that touches BLE flows through it. Adding a new command? Extend it. Don't fork it.

See [[BLE Protocol]] for the wire contract.

### SessionUploader.ts
Implements the **700 KB Firestore page** rule. Firestore documents are 1 MB max; the cap leaves headroom for metadata + atomic batch overhead. **Never write `sensorData` directly** — go through this.

See [[Firebase Backend#Paging rules]].

### StatsEngine.ts
One of three [[StatsEngine Cross-Platform|cross-platform StatsEngine implementations]]. If you change a metric formula here, change it in the [[Cresento Website|web]] and [[DataRecoveryIOS (ShinPad iOS)|iOS]] versions too, or coaches will see different numbers in different places.

### LED feedback (chip-side)
The RN app sends commands that drive the LED on the chip. Don't change the command sequence around connection / start / stop without checking [[Nordic BLE Firmware (nRF52811)#LED behaviour]] and [[ESP32-S3 SmallBoard Firmware#LED behaviour]].

---

## Roles

- `"player"` — sees their own sessions, friends, settings
- `"coach"` — sees team, players, advanced analytics

The role lives in the user's Firestore profile (`users/{uid}.role`) and is checked at navigation level. See [[Firebase Backend#users collection]].

---

## Related

- [[Cresento Website]] — the coach-side dashboard
- [[DataRecoveryIOS (ShinPad iOS)]] — Swift sibling, must stay feature-aligned
- [[BLE Protocol]] — the wire contract this app speaks
- [[Firebase Backend]] — what it reads/writes
- [[Data Pipeline]] — end-to-end session flow
