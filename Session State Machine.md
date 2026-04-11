---
title: Session State Machine
type: shared-system
tags:
  - shared
  - protocol
  - critical
  - firmware
created: 2026-04-11
---

# 🔁 Session State Machine

The chip-side state machine that governs a recording session. **Carefully tuned**, do not restructure.

```
   ┌──────┐  start cmd     ┌──────────┐  stop cmd / disconnect / charge
   │ idle │ ─────────────► │ logging  │ ────────────────────────────────┐
   └───▲──┘                └──────────┘                                  │
       │                                                                  │
       │                  ┌──────────┐                                    ▼
       └──────────────────│ flushing │◄──────── flash drain to BLE
        flush done /      └──────────┘
        all pages acked
```

---

## States

### `idle`
- BLE advertising
- Low power
- Ready to receive `start` command
- Battery monitoring active

### `logging`
- ICM-42688 sampled at **20 Hz**
- Samples accumulated, then written to W25Q128 flash
- LED indicates active recording
- BLE connection NOT required to stay in this state — disconnect mid-recording is fine, the chip keeps logging to flash

### `flushing`
- Flash contents read back
- Delta-pack compressed (optionally gzipped on ESP)
- Streamed over BLE notifications with sequence numbers
- App acknowledges pages
- On completion → `idle`

---

## Transitions

| From       | Trigger                                                  | To         | Notes |
| ---------- | -------------------------------------------------------- | ---------- | ----- |
| `idle`     | `start` command on `Control` characteristic              | `logging`  | LED to recording state |
| `logging`  | `stop` command                                           | `flushing` | |
| `logging`  | Charging event (USB plug-in)                             | `flushing` | Charging stops a session |
| `logging`  | BLE disconnect                                           | `logging`  | Stays in state, keeps writing to flash |
| `flushing` | All pages acknowledged                                   | `idle`     | |
| `flushing` | BLE disconnect mid-flush                                 | `flushing` | **Resumes**, does NOT restart |

---

## Why mid-flush disconnect must resume, not restart

This is the rule that exists because of a bug:

If a flush is 80% done and the BLE link drops, restarting from byte 0 wastes time AND risks re-uploading already-stored pages on reconnect, causing duplicate `sensorData` documents in Firestore. The state machine must remember where it was and resume.

> [!danger] Rule
> Do not change resume-from-offset logic. It exists because the alternative caused duplicate sessions in Firestore in the past.

---

## Charging events

Plugging in USB while recording = end the session. The chip transitions `logging → flushing` automatically and the LED reflects the new state. This is intentional UX: "I'm done, plugging in to charge" should always end the session cleanly.

The ESP firmware additionally has a 1500 ms grace period on USB unplug before deep sleep, so BLE notifications can flush.

---

## Reading order

If you're new to the chip-side, read in this order:
1. This note
2. [[BLE Protocol]]
3. [[Nordic BLE Firmware (nRF52811)]] OR [[ESP32-S3 SmallBoard Firmware]]
4. The actual `.ino` file

---

## Related

- [[BLE Protocol]]
- [[Data Pipeline]]
- [[Nordic BLE Firmware (nRF52811)]]
- [[ESP32-S3 SmallBoard Firmware]]
- [[01 - Critical Preservation Rules#🔁 Session state machine]]
