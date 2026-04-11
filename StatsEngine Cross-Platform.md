---
title: StatsEngine Cross-Platform
type: shared-system
tags:
  - shared
  - analytics
  - critical
created: 2026-04-11
---

# 🧮 StatsEngine — Cross-Platform Contract

The same analytics logic, **implemented three times**, on three different runtimes. The contract is: **same input session → same numbers**.

| Platform | File                                                    | Size       |
| -------- | ------------------------------------------------------- | ---------- |
| RN       | `Cresento/src/src/utils/StatsEngine.ts`                | ~37 KB     |
| Web      | `Cresento Website/Cresento.net/lib/analytics/`          | (multi-file)|
| iOS      | `DataRecoveryIOS/.../StatsEngine.swift`                | (Swift class)|

> [!danger] If they disagree, coaches lose trust
> The number-one source of "the platform is buggy" support tickets is "the website says X but my phone says Y." Keep them aligned.

---

## What it computes

40+ sport-science metrics from a single session of 20 Hz IMU data:

- **Kinematic:** speed (instantaneous, mean, max), acceleration, distance
- **Movement:** step count, cadence
- **Sprint detection:** sprint count, sprint distances, peak sprint speed
- **High-speed running (HSR):** distance above HSR threshold
- **Fatigue index:** early-session vs late-session sprint speed comparison
- **ACWR:** Acute:Chronic Workload Ratio (injury-risk proxy)
- **Position scoring:** comparison to Premier League position benchmarks (web does this in `lib/position-scoring.ts`)
- **Intensity zones:** time in low / moderate / high / very high
- **Energy expenditure:** estimated kcal

The exact list and formulas live in the RN `StatsEngine.ts` — treat that as the reference implementation.

---

## How they stay in sync

There is no automated test that compares the three. The discipline is:

1. **Decide on RN first.** Prototype in [[Experimental Implementation]] → port to RN `StatsEngine.ts`.
2. **Port to web** (`lib/analytics/`) with the same formula.
3. **Port to iOS** (`StatsEngine.swift`) with the same formula.
4. **Sanity-check on a real session** — pick one well-known session, run all three, compare the top-line metrics.
5. Land them in the same release window so users don't see version skew.

> [!warning] Don't change just one
> If you "fix a bug" in the web's stats but not RN/iOS, you've created a worse bug: divergent numbers across the user's own devices. Always touch all three.

---

## Why three implementations?

- **RN** computes on the phone so the player gets immediate feedback without a round trip
- **iOS** is the same idea on a different runtime
- **Web** computes server-side so the coach can rerun analysis on edited / trimmed sessions without needing the player's phone

A single shared implementation (e.g. WASM or a cloud function) has been considered but not done. The three-way maintenance burden is the cost of that decision.

---

## Position-specific benchmarks

`lib/position-scoring.ts` (~33 KB) contains positional benchmarks derived from Premier League data. These are tuned values, not arbitrary. Don't "round to nicer numbers."

---

## Intensity zones

The threshold values (low / moderate / high / very high) are duplicated across the three implementations. If you change them, change them everywhere. Consider centralising into a constants file shared between web and one of the others if you do it.

---

## Related

- [[Cresento React Native App#StatsEngine.ts|RN StatsEngine]]
- [[Cresento Website#Library use rules|Web lib/analytics]]
- [[DataRecoveryIOS (ShinPad iOS)#Stats engine|iOS StatsEngine]]
- [[Experimental Implementation]] — where prototypes start
- [[Data Pipeline]]
- [[01 - Critical Preservation Rules#🧮 StatsEngine consistency across platforms]]
