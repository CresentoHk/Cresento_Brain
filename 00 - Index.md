---
title: Cresento Workspace Index
type: moc
tags:
  - moc
  - index
created: 2026-04-11
---

# 🧠 Cresento Workspace — Map of Content

Cresento is a sports analytics platform for soccer coaches and athletes. It spans **mobile apps, a web dashboard, embedded firmware on two different chips, a computer-vision pipeline, and AI tooling** — all backed by a single Firebase project (`cresento-8b603`).

> [!danger] Read this first
> Before touching ANY code, read [[01 - Critical Preservation Rules]]. The chip protocol, BLE LEDs, and session state machine are extremely sensitive. Breaking them is the #1 thing to avoid.

---

## 🗺️ Start Here

- [[01 - Critical Preservation Rules]] — what you must not break
- [[02 - Architecture Overview]] — how all pieces fit together
- [[03 - Data Pipeline]] — IMU sample → analytics, end-to-end

---

## 📦 Active Projects

| Project                                            | Stack                       | Role               | Status |
| -------------------------------------------------- | --------------------------- | ------------------ | ------ |
| [[Cresento React Native App]]                      | RN 0.80, TS, Firebase       | Player + Coach app | Active |
| [[Cresento Website]]                               | Next.js 14 App Router, TS   | Coach dashboard    | Active |
| [[Nordic BLE Firmware (nRF52811)]]                 | C/Arduino, S112 SoftDevice  | ShinPad firmware   | Active |
| [[ESP32-S3 SmallBoard Firmware]]                   | C/Arduino, ESP32 BLE + WiFi | Newer board design | Active |
| [[DataRecoveryIOS (ShinPad iOS)]]                  | Swift, SwiftUI, Firebase    | Native iOS app     | Active |
| [[Vision System (Veo Pipeline)]]                   | Python, OpenCV, YOLO        | Match-video CV     | Active |

---

## 🧪 Supporting / Reference Projects

| Project                                | Purpose                              |
| -------------------------------------- | ------------------------------------ |
| [[AI Coach (text_ai_coach)]]           | Python RAG soccer coach (Together)   |
| [[Experimental Implementation]]        | IMU rest-detection / data exploration|
| [[Reference Repos]]                    | Third-party ML models for soccer CV  |

---

## 🔗 Shared Systems

These are crosscutting — touching them affects every platform.

- [[Firebase Backend]] — auth, Firestore collections, paging rules
- [[Firestore Collection Audit 2026-04-11]] — current inventory + cleanup plan
- [[Agent Memory System]] — Cora's long-term memory (episodes, recall, future playbooks)
- [[BLE Protocol]] — UUIDs, command bytes, packet format (FROZEN)
- [[Session State Machine]] — idle → logging → flushing
- [[StatsEngine Cross-Platform]] — three implementations that must match

---

## 🔧 Hardware

- [[Hardware Overview]] — nRF52811 vs ESP32-S3 boards, sensors, pinouts

---

## 🏷️ Tags

Use these consistently when adding new notes:

- `#project/rn-app` `#project/website` `#project/nordic` `#project/esp` `#project/ios` `#project/vision` `#project/ai-coach`
- `#shared` — cross-platform concerns
- `#critical` — do-not-break code
- `#firmware` `#frontend` `#backend` `#ml` `#cv` `#hardware`
- `#protocol` — wire format / contract
