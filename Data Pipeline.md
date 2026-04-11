---
title: Data Pipeline
type: shared-system
tags:
  - shared
  - architecture
  - data
created: 2026-04-11
aliases:
  - "03 - Data Pipeline"
---

# 🔄 Data Pipeline

End-to-end: from a player kicking a ball to the coach seeing analytics on the website.

```
┌─────────────┐    ┌─────────┐    ┌──────┐    ┌──────┐    ┌──────────┐    ┌──────────┐    ┌────────┐
│ ICM-42688   │───►│ on-chip │───►│ flash│───►│ BLE  │───►│ phone    │───►│ Firestore│───►│ website│
│ 6-axis IMU  │    │ buffers │    │W25Q128│    │notify│    │ decode + │    │ paged    │    │ render │
│  20 Hz      │    │         │    │      │    │      │    │ uploader │    │ session  │    │        │
└─────────────┘    └─────────┘    └──────┘    └──────┘    └──────────┘    └──────────┘    └────────┘
                                       │             ▲
                                       │             │
                                       │  resume on  │
                                       │  reconnect  │
                                       └─────────────┘
                                       (state machine)
```

---

## Stage 1 — Acquisition

- **ICM-42688** sampled at **20 Hz** (gyro + accel)
- Raw int16 samples
- Currently configured to record gyroscope-only on the ESP variant (`COLS = 1`); the Nordic variant historically recorded all 6 axes — confirm in code before assuming.

---

## Stage 2 — On-chip buffering & flash

- Samples accumulate in RAM
- Periodically flushed to **W25Q128** SPI flash (16 MB)
- Survives BLE disconnects — the chip keeps logging even with no phone connected
- See [[Session State Machine]] for the `idle → logging → flushing` flow

---

## Stage 3 — Compression (delta-pack + optional gzip)

Two codecs are defined:

```c
#define CODEC_DELTAPACK   1   // delta encoding + bit-packing
#define CODEC_DP_GZIP     2   // delta-pack + gzip on top
```

- **Delta-pack** turns successive samples into smaller deltas, then bit-packs them
- **Gzip** (via miniz) optionally squeezes the output further
- Chunk sizes are tuned for the chip:
  - `CHUNK_ROWS = 512` rows on ESP (sweet spot 512–1024)
- Worst-case payload sizes are over-allocated as static buffers to bound RAM:
  - `RAW_PER_ROW_BYTES = COLS * 2` (int16 per column)
  - `MAX_BITPACK_PAYLOAD = CHUNK_ROWS * COLS * 3`
  - `MAX_GZIP_PAYLOAD = MAX_BITPACK_PAYLOAD + MAX_GZIP_OVERHEAD`

> [!warning] Encoder ↔ decoder must agree
> Whatever the chip writes, the apps must decode. **Don't change the codec on one side without the other.**

---

## Stage 4 — BLE notifications

- Compressed chunks sent on the **`Data` characteristic** (notify) with sequence numbers
- App reassembles in order
- See [[BLE Protocol]]

---

## Stage 5 — Phone-side decode

| Platform | Code path                                              |
| -------- | ------------------------------------------------------ |
| RN       | `Cresento/src/src/contexts/BleContext.tsx` (decoder)   |
| iOS      | `BLEViewModel` in `DataRecoveryIOS/`                   |

The decoder:
1. Validates sequence numbers
2. Decompresses (gzip → bit-unpack → undelta)
3. Reconstructs the int16 sample stream
4. Converts to CSV in memory

---

## Stage 6 — Paged Firestore upload

> [!danger] 700 KB cap is load-bearing
> Firestore documents are 1 MB max. The 700 KB page cap leaves headroom for metadata + atomic batch overhead.

- The CSV is split into pages
- Each page is **≤ 700 KB**
- Pages are written atomically as Firestore batches so partial sessions don't appear
- Implemented in `Cresento/src/src/utils/SessionUploader.ts`
- See [[Firebase Backend#Paging rules]]

**Never bypass the uploader.** If you need a new field on a session, add it through the uploader, not by reaching into Firestore directly.

---

## Stage 7 — Analytics (StatsEngine)

Three implementations of the same logic, must produce the same numbers:

| Platform | Path                                                    |
| -------- | ------------------------------------------------------- |
| RN       | `Cresento/src/src/utils/StatsEngine.ts` (~37 KB)       |
| Web      | `Cresento Website/Cresento.net/lib/analytics/`          |
| iOS      | `DataRecoveryIOS/.../StatsEngine.swift`                |

40+ metrics including: speed, acceleration, distance, step count, sprint detection, HSR (high-speed running), fatigue index, ACWR (acute:chronic workload ratio), position-specific benchmarks, intensity zones, energy expenditure.

See [[StatsEngine Cross-Platform]].

---

## Stage 8 — Website rendering

- The website's `lib/firestore.ts` reads paged sessions
- `components/dashboard/sessions-calendar.tsx` (~70 KB) groups + renders them
- `lib/trim-utils.ts` (~31 KB) supports clipping start/end of recordings
- `lib/position-scoring.ts` (~33 KB) compares against Premier League benchmarks
- `components/analytics/` (35+ components) produces the actual charts

---

## Failure modes & recovery

| Failure                                | Behaviour                                          |
| -------------------------------------- | -------------------------------------------------- |
| BLE disconnect mid-logging             | Chip keeps logging to flash; re-flush on reconnect |
| BLE disconnect mid-flushing            | Resume from last acked offset (do NOT restart)     |
| Phone app crash mid-upload             | Atomic batches mean no partial sessions appear     |
| Charging plug mid-session              | `logging → flushing` (intentional)                 |
| Decoder version mismatch               | Catastrophic — keep encoder/decoder in lockstep    |

---

## Related

- [[BLE Protocol]]
- [[Session State Machine]]
- [[Firebase Backend]]
- [[StatsEngine Cross-Platform]]
- [[Nordic BLE Firmware (nRF52811)]] / [[ESP32-S3 SmallBoard Firmware]]
- [[Cresento React Native App]] / [[DataRecoveryIOS (ShinPad iOS)]]
- [[Cresento Website]]
